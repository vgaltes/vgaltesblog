---
title: 'Deploying Azure Functions using CircleCI'
date: '2018-11-15'
categories:
- serverless
tags:
- devops
- azure functions
- azure
---

In the last [article](https://vgaltes.com/post/deploying-azure-functions-an-introduction/) we saw how to deploy an azure function from the CLI. In this article we'll see how we can use the same script to deploy the function from a continuous integration environment. In this case we'll use [CircleCI](https://circleci.com).

## Creating a new docker image
As we saw, if we want to deploy the Function App using the same script we'd need to use the Azure CLI and the Azure Functions Core Tools. We will need the .Net Core SDK to be installed to be able to build the solution. Currently there isn't any [pre-built CircleCI Docker Images](https://circleci.com/docs/2.0/circleci-images/) with all of that installed, so we'll need to create our own image. Fortunately, the CircleCI people makes this easy to do. What we'll need to do is follow the instructions that you can find [here](https://github.com/circleci-public/dockerfile-wizard).

Let's start forking the repository into our account. The next step is to add your [Docker Hub](https://hub.docker.com/) username (DOCKER_USERNAME) and password (DOCKER_PASSWORD) to CircleCI (yes, you need to creat an account there). I've added this into the project specific environment variables. After that, run the `make ready` and `make setup` to prepare the different files that we'll use to create the Docker image and publish it to Docker Hub. I've selected UBUNTU_XENIAL as the image base, and selected to install Python version 3.7.1. Now it's time to commit and push your changes and see what happens.

What will happen is that the build will not work. It will complain that the `org-global` context doesn't exist. What do you need to do is to create this context. So, go `Settings -> Contexts` and add a new context.

![Add context](/images/AddContext.png)

Now, if you rebuild the workflow it won't work either, because the file has some errors. Let's fix this errors replacing the current content of the file by this (feel free to change the name of the image and the tag, but if you do this, remember to change the name of the image in the test_image job):
```
image_config: &image_config

  IMAGE_NAME: azurefunctionsbuildanddeploy

  IMAGE_TAG: "1.0"

  LINUX_VERSION: UBUNTU_XENIAL

  PYTHON_VERSION_NUM: 3.7.1

  JAVA: false

  MYSQL_CLIENT: false

  POSTGRES_CLIENT: false

  DOCKERIZE: false

  BROWSERS: false

version: 2
jobs:
  build:
    machine: true
    environment:
      <<: *image_config

    steps:
      - checkout

      - run: bash scripts/generate.sh > Dockerfile

      - run: docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD

      - run: docker build -t $DOCKER_USERNAME/$IMAGE_NAME:$IMAGE_TAG .

      - run: docker push $DOCKER_USERNAME/$IMAGE_NAME:$IMAGE_TAG && sleep 10

      - store_artifacts:
          path: Dockerfile

  test_image:
    docker:
      - image: $DOCKER_USERNAME/azurefunctionsbuildanddeploy:1.0
        environment:
          <<: *image_config

    steps:
      - checkout

      - run:
          name: bats tests
          command: |
            mkdir -p test_results/bats
            bats scripts/tests.bats | \
            perl scripts/tap-to-junit.sh > \
            test_results/bats/results.xml

      - store_test_results:
          path: test_results

      - store_artifacts:
          path: test_results

workflows:
  version: 2
  dockerfile_wizard:
    jobs:
      - build:
          context: org-global

      - test_image:
          context: org-global
          requires:
            - build

```

So, let's commit and push and see this working.

The next step is to install the tools we need. How this works, is that CircleCI will create the Dockerfile running the `generate.sh` script. If you take a look at the script you will see that is creating a Dockerfile using echo's. So, what we need to do is update this script to install what we need. Take into account that if you change your base image (Ubuntu Xenial) you might need to change some of the following instructinos. Let's start adding what we need to install the AzureCLI:
```
echo "# install Azure CLI
RUN apt-get update \
  && apt-get install apt-transport-https -y \
  && echo 'deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ xenial main' | tee /etc/apt/sources.list.d/azure-cli.list \
  && apt-get install curl -y \
  && curl -L https://packages.microsoft.com/keys/microsoft.asc | apt-key add - \
  && apt-get update \
  && apt-get install libssl-dev libffi-dev python-dev apt-transport-https azure-cli -y"
```

The next step is to install the .Net Core SDK:
```
echo "#dotnet sdk
RUN apt-get install wget \
  && wget -q https://packages.microsoft.com/config/ubuntu/16.04/packages-microsoft-prod.deb \
  && dpkg -i packages-microsoft-prod.deb \
  && apt-get update \
  && apt-get install dotnet-sdk-2.1 -y"
```

And finally let's install the Azure Functions Core Tools:
```
echo "#Azure Functions core tools
RUN apt-get install azure-functions-core-tools -y"
```

Next step is to add some tests to check that everything is correctly installed. This project is using [BATS](https://github.com/bats-core/bats-core) to perform the tests. So, let's add some tests to make sure we've installed everything correctly. Edit the tests.bats file and add the following lines at the end of the file:
```
@test "dotnet" {
  dotnet --version
}

@test "azure cli" {
  az --version
}

@test "azure functions core tools" {
  func --version
}
```

Cool, you can now commit and push and hopefully you'll see the build passing a new docker image in your Docker Hub account. It's time to create a build definition for our Azure Functions project.

## Create a new Service Principal
When we deployed the Function App from our local environment, we used `az login` with a username and a password. From our CI agent we will use a service principal. So, the first thing we need is to create a new one. Open a new terminal session and type `az ad sp create-for-rbac --name <ServicePrincipalName> --password <PASSWORD>`. You can find more information about how to create a service principal [here](https://docs.microsoft.com/en-gb/cli/azure/create-an-azure-service-principal-azure-cli?toc=%2Fazure%2Fazure-resource-manager%2Ftoc.json&view=azure-cli-latest).

## Create a new build definition
Let's go to your Azure Functions project (the one we created in the last article) and create a folder called `.circleci` with a file called `config.yml` inside. This is our build definition file.

First thing we need to do is to specify that we want to use our newly created image to build this project. So, add this as the first lines of your config.yml file:
```
version: 2
jobs:
  build:
    docker:
      - image: vicencgarcia/azurefunctionsbuildanddeploy:1.0

    working_directory: ~/repo
```

Next, we need to specify the different steps of our build. The first one is to checkout the code:
```
   steps:
      - checkout
```

Then, we need to login into our Azure account:
```
      - run:
          name: Login into azure
          command: az login --service-principal -u $azPrincipal -p $azPassword -t $azTenant
```

As you can see, we're using three environment variables. So, you'll need to add them to the project settings. You'll have the information from the step where you've created the Service Principal.

And finally, we need to deploy the Function App:
```
      - run:
          name: Deploy
          command: ./deploy.sh AFTestDeploy westeurope uat
```

Feel free to change the service name, the location and the stage.

Let's commit, push and see if we've deployed the Function App.

## Adding settings
The build is green, which is good, but if we go to the portal we'll see that everything has beed deployed but the HelloWorld function. What has happened? Let's take a look at the build output:

```
  Restore completed in 2.12 sec for /root/repo/TestDeploy.csproj.
  TestDeploy -> /root/repo/bin/Debug/netcoreapp2.1/bin/TestDeploy.dll

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:04.92
Publisihng function to Azure
Getting site publishing info...
Preparing archive...
Uploading content...
Upload completed successfully.
Deployment completed successfully.
Syncing triggers...
Functions in AFTestDeploy-uat:
```

Weird. The .Net SDK is not publishing anything in the /root/repo/bin/Debug/netcoreapp2.1/publish folder, and therefore nothing is deployed to Azure. What's happening?

## Adding local.settings.json
It turns out that the Azure Functions Core Tools needs a parameter that exists on the local.settings.json file. But this file is on the .gitignore file and it's not safe to upload it to the repository. What can we do? Well, we can create the minimum file that we need in order to be able to deploy the function. Let's add this step to our build configuration file, just before the deploy step:

```
      - run:
          name: Create local.settings.json
          command: |
            cat > local.settings.json << EOF
            {
              "IsEncrypted": false,
              "Values": {
                "AzureWebStorage": "",
                "FUNCTIONS_WORKER_RUNTIME": "dotnet"
              }
            }
            EOF
```

Let's commit and push and see if the function is deployed...

![Deployment](/images/deployment.png)

Voilà! Here it is!

## Summary
In this article we've seen how we can use CircleCI to deploy and Function App. We had to create a new Docker image and create a build definition file that uses it.
