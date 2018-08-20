---
title: 'Using different configuration per stage'
date: '2018-05-16'
categories:
- serverless
tags:
- devops
- lambda
- circleci
- aws
---

In the [previous article](./deploy-serverless-app/) we saw how to create a basic deployment pipeline for a serverless application. In this article, we're going to enrich the deployment by allowing to have different values for configuration settings in each stage. 

## Background

The moment your application starts to be a little bit more complex, you need to use configuration settings. These settings can be things like the log level, addresses of external services, usernames and (encrypted) passwords, etc. But we don't want to use the same settings in all the environments, as we don't want to access production services while doing tests in dev. Let's see how we can do this in a serverless application.

## Serverless framework as a dev dependency

In the previous article, we installed the serverless framework as a global dependency. Although this is quite convenient, it can be a problem. Different versions of the framework can behave in a different way, or introduce breaking changes and it's not practical that every member of the team has its own version installed. So, a good practice is to install the framework as a dev dependency and always use the version inside node_modules to deploy. 

So let's install the framework as dev dependency:
```
npm install serverless --save-dev
```

and create some scripts in the package.json file to make our live easier:

```
"deploy": "./node_modules/.bin/serverless deploy",
"deploy:pre": "./node_modules/.bin/serverless deploy --stage pre",
"deploy:prod": "./node_modules/.bin/serverless deploy --stage prod"
```

Now we should change the CircleCI config.yml file to use the new way to deploy. We're also going to change how we install the packages in the CI pipeline. So, go to the section named "Install Serverless CLI and dependencies" and substitute the "npm install" line with the following ones:

```
# update npm
sudo npm install -g npm
npm ci
```

We want to use the new *ci* command in npm (more info [here](https://docs.npmjs.com/cli/ci)) so we need to update npm to the last version. If the docker image you're using has already version 5.7 or above, you can skip this step.

The next step is to change the deployment steps to use the new scripts. The pre-production step should now look like

```
- run:
    name: Deploy application
    command: npm run deploy:pre
```

And the production step should look like:

```
- run:
    name: Deploy application
    command: |
    chmod +x scripts/aws-cli-assumerole.sh
    source scripts/aws-cli-assumerole.sh
    npm run deploy:prod
```

Push the changes and give it a go. 

## Config file
It's time to create a config file to store the different settings. For this quick demo, we're just going to use one. So, go to the root of the project (where you have the serverless.yml file), create a file called vars.yml and copy the following content:

```
dev-user:
  message: "Environment variable from dev-user"
pre:
  message: "Environment variable from pre"
prod:
  message: "Environment variable from prod"
```

We're creating a configuration setting called message which will have different values in each environment. We're here defining a stage for the user (more on this later), a stage for pre and stage for prod. What we want is that the developers on their machines use the dev-user settings and that the CI environment uses the pre settings when deploying to pre and the prod settings when deploying to prod.

## Loading the settings
We need now to use this setting on the serverless.yml file. The first thing we need to do is to define the default stage where we want to deploy the application. This will be the user stage. But we want each of our devs to have their own stage, so they can make tests independently of each other. The serverless framework makes this easy, as it already creates the functions and resources with the stage as a part of the name. So what we just need is to provide a different stage name for each user. Let's do this by adding this line to the provider section of the serverless.yaml:

```
stage: dev${env:SLSUSER}
```

So, if each developer creates an environment variable called SLSUSER on its laptop with a unique value, the serverless framework will use that value when naming the functions.

Now it's time to define some custom properties:
```
custom:
  stage: ${opt:stage, self:provider.stage}
  default-vars-stage: dev-user
  vars: ${file(./vars.yml):${opt:stage, self:custom.default-vars-stage}}
```

The first property is to define the stage we want to use, which will be the stage provided by the arguments of the *sls deploy* command or, in case we don't provide any, the value of the provider.stage property (which will be the dev user stage).
The second one is to define the default stage inside the vars.yml file we want to use. Again, the default one will be the dev user stage.
And finally, we're loading the variables in the vars.yml file. If the user provides the stage argument in the deploy command, we'll use that stage. If not, we'll use the default one, which is the dev-user.

## Using the settings
To demonstrate how we can use those settings, we're going to define an environment variable in the hello function that we'll print. We're also going to define an HTTP event so the invocation will be easier. Let's create the environment variable in the function:

```
environment:
    MESSAGE: ${self:custom.vars.message}
```

Thanks to the previous step, we have all the variables in the custom.vars object, so we can access the variable as a property of that object.

## Using the environment variable in the function
Let's change the code of the function to use the environment variable:

```
module.exports.hello = (event, context, callback) => {
  const response = {
    statusCode: 200,
    body: JSON.stringify({
      message: `The message is: ${process.env.MESSAGE}`,
      input: event,
    }),
  };

  callback(null, response);
};
```

We'll need to change the unit test as well:

```
describe("The handler function", () => {
    it("returns a message", () => {
        process.env.MESSAGE = "message"
        handler.hello(undefined, undefined, function(error, response){
            let body = JSON.parse(response.body);
            body.message.should.be.equal(`The message is: message`);
        });
    });
});
```

## Deploying
Let's first try to deploy the solution using our dev account:

```
npm run deploy -- --aws-profile vgaltes-dev
```

This will deploy using a local aws profile called vgaltes-dev. If the deployment is successful, you'll something like this in your console:

```
Service Information
service: test-circleci
stage: devvgaltes
region: us-east-1
stack: test-circleci-devvgaltes
api keys:
  None
endpoints:
  GET - https://XXXXXXX.execute-api.us-east-1.amazonaws.com/devvgaltes/hello
functions:
  hello: test-circleci-devvgaltes-hello
```

Note the devvgaltes suffix, as my SLSUSER environment variable is set to vgaltes.

If you click on the link, you will see a response like:

```
{"message":"The message is: Environment variable from dev-user",...
```

This looks good. Let's push everything and see the result of the deployment using CircleCI. Once the deployment is successful, go to the result of the deployment on pre and the result of the deployment on prod and click on the links. For pre, you should see that the response is:

```
{"message":"The message is: Environment variable from pre",...
```

And for prod the response should be:

```
{"message":"The message is: Environment variable from prod",...
```

## Summary
In this article, we've seen how we can set different values for a configuration variable per environment. Another option you have is to use [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html). You can see how to do this in [this article](https://theburningmonk.com/2017/09/you-should-use-ssm-parameter-store-over-lambda-env-variables/) from [Yan Cui](https://twitter.com/theburningmonk).


Hope it helps!
