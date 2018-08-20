---
title: 'Deploying a serverless application'
coverImage: /images/BuildProd-2.png
date: '2018-04-19'
categories:
- serverless
tags:
- devops
- lambda
- circleci
- aws
comments: []
lcb: "{{"
rcb: "}}"
---

Starting with AWS Lambda is really easy. You can even write a function in the browser! But that's not how you should work on a daily basis. You must have a CI/CD pipeline set up, you probably have two accounts (one for production and another one for development), you need a repeatable and reliable way to create your infrastructure and so on. In this article I'll show you how to create a simple continuous delivery pipeline that brings us closer to a professional application development.

## Disclaimer

The goal of this article is not to show you how to build or test a serverless application, so the sample application will be extremely easy and the tests might not make sense. The goal is to learn how we can create a CD pipeline to deploy a serverless application.

## Pre-requisites

To follow this article you'll need:
 - Two accounts in AWS, one simulating the development and the other one simulating the production environment.
 - A CircleCI account set up.
 - NodeJS installed in your computer.
 - The serverless framework installed on your computer.
 - An AWS profile created in your computer. In my case, the name of the profile is vgaltes-serverless
 - jq installed on your computer.

##Sample application

Let's start by creating a *VERY* simple NodeJS serverless application. So, open a terminal, create a new folder wherever you want, cd to it and type:
```
serverless create --template aws-nodejs
```

This will create a basic serverless project with a function that just says hello. Now it's time to add a test to that function. Create a file called handler.spec.js and copy the following code in it:

```
const mocha = require('mocha');
const chai = require('chai');
const should = chai.should();

const handler = require('./handler');

describe("The handler function", () => {
    it("returns a message", () => {
        handler.hello(undefined, undefined, function(error, response){
            let body = JSON.parse(response.body);
            body.message.should.be.equal('Go Serverless v1.0! Your function executed successfully!');
        });
    });
});
```

Before being able to run this test, you need to install the packages:

```
npm install --save-dev mocha
npm install --save-dev chai
```

We're now very close to be able to run the test. Just add the following line inside the scripts section of the package.json file:

```
"test": "./node_modules/.bin/mocha **/*.spec.js"
```

Run *npm test* and you should see a test passing

![A test](/images/CDServerlessArticle-1.png)

And that's all! As I told you, we're not going to be famous for writing a super-complex application.

## A basic CI pipeline

Now it's time to start working on our CI pipeline. I've chosen [CircleCI](https://circleci.com) because it's a PAAS service and I wanted to try it out. Choose your preferred CI system here, the steps will be very similar.

As we're using CircleCI, we need to create a folder named .circleci in the root of our project and a file named config.yml inside it. Do it and copy the followint yaml code in the file.

```
version: 2
jobs:
  build:
    docker:
      - image: circleci/node:8.10

    working_directory: ~/repo

    steps:
      - checkout

      # Download and cache dependencies
      - restore_cache:
          keys:
          - v1-dependencies-{{ checksum "package.json" }}
          # fallback to using the latest cache if no exact match is found
          - v1-dependencies-

      - run:
          name: Install Serverless CLI and dependencies
          command: |
            sudo npm i -g serverless
            npm install

      - save_cache:
          paths:
            - node_modules
          key: v1-dependencies-{{ checksum "package.json" }}
        
      # run tests!
      - run: 
          name: Run tests with coverage
          command: npm test --coverage

      - run:
          name: Deploy application
          command: sls deploy --stage pre
```

The code is pretty straightforward. We're using a docker image with Node 8.10 inside it, checking out the code, installing the serverless framework, installing our project dependencies, run some unit tests and deploy to a stage called pre.

### Configure CircleCI

Before pushing this file to github, we need to configure CircleCI to listen to our repository and execute the accions we set on the config.yml. First of all, create an administrator user for you CI (I named it circleci) and disable the console password so it only can access via code. 

First of all, go to *Add projects* and click on the *Set Up Project* button of the project you want to add. Check that the operating system is linux and that the languaje is Node and click Start building. This will fire your first build.

Now, click on the settings button of the builds page, on the top right corner. On the left hand side of the screen, there's a section called permissions. Click the *AWS Permissions* button.

![Project permissions](/images/ProjectPermissions.png)

Set the keypair of your CI user there.

![AWS Permissions](/images/AWSPermissions.png)

It's time to push and see what happens in CircleCI.

![CircleCI Build](/images/CDServerlessArticle-2.png)

Everything looks good. Let's take a look at our development account in AWS.

![AWS Lambda console](/images/CDServerlessArticle-3.png)

Voilà!

## AWS Assume role

We now have a lambda deployed into our development account using CircleCI, which is pretty good. But most organisations have a completely separate account for their production stuff. How can we deploy from a system where we have a dev account configured to a completely different account? AWS Assume role to the rescue!

First of all we need to get the account id of our dev account. Go to support -> support center and get the id from there. 

Now we need to create a role in the production account, so log in there with your administrator account and go to the IAM service and create a new role. The type should be "Another AWS Accound" and you must provide the dev account id.

![Create new role - step 1](/images/CDServerlessArticle-4.png)

Clicking next we will provide the permissions for the role. Let's choose administrative permissions for now to check that all works well and we'll change that later.

![Create new role - step 1](/images/CDServerlessArticle-5.png)

And finally let's set a name for the role, in our case circleci_role

![Create new role - step 1](/images/CDServerlessArticle-6.png)

We now have the role created. We now have to change user accounts (or groups) in our dev account to allow them to switch to this role and hence, access the production account.

We're going to do this for a particular user, but you can choose a group as well. That will depend on how do you want to organise your team and your organisation. Do you want everybody to be able to deploy directly to prod or we just want that the CI user can deploy to prod? Second approach sounds sensible.

So, go to your dev account, go to IAM service and select the user you want to give permissions to. Click on permissions -> add inline policy and use the following JSON:
```
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::262170986110:role/circleci_role"
  }
}
```

![Create new policy](/images/CDServerlessArticle-7.png)

Click on review and set a name for the policy, for example *allow-assume-circleci-role-production*

It's time to test it that works. We can do 4 types of tests:

### Test 1 - Using the UI
We can assume a role using the website UI. Click on your username and then "Switch Role". Fill the following form with the required data: the production account id and the production role we've just created.

![Assume role](/images/AssumeRoleUI-1.png)

Now, you should see the indicator on the place where your name previously was. You're now in your production account and you can take a look at your S3 buckets there.

![Assume role](/images/AssumeRoleUI-2.png)

### Test 2 - CLI Using profiles
We can assume a role configuring properly an AWS named profile. As you might now, profiles are a way to have more than one account credentials, accessible by name.

Edit your ~/.aws/credentials and add the following configuration:

[vgaltes-prod]
role_arn = arn:aws:iam::262170986110:role/circleci_role
source_profile = vgaltes-serverless

The source profile is the named profile of your dev account and the role arn is the arn on the production account, the role you want to assume.

Let's try this. Go to your project base folder and type:

```
aws s3 ls —-profile vgaltes-prod
```

And you should see your S3 buckets on the production account.

### Test 3 - Getting the credentials using the CLI
The third option you have (and the first one that will lead us to what we'd need to do in the CI environment) is to retrieve the credentials needed to log into the production account (a temporary ones) via AWS STS. Open the console and type:

```
aws sts assume-role --role-arn "arn:aws:iam::262170986110:role/circleci_role" --role-session-name "Vgaltes-Prod" --profile vgaltes-serverless
```

Where the role arn is the role you want to assume and profile is your dev profile.

This command should return something like this:

```
{
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAIHUHXX66KCXPBZHP4:Vgaltes-Prod", 
        "Arn": "arn:aws:sts::262170986110:assumed-role/circleci_role/Vgaltes-Prod"
    }, 
    "Credentials": {
        "SecretAccessKey": "Q833K5zxeaWiWI1hBbbmfTYF01Px72imuvcGB3TA", 
        "SessionToken": "FQoDYXdzENz//////////wEaDKbdCGIckdfH7OdGcyLwAY1V/m+S38Fu/Mh+KHWNbg58kfm7047AAMYZR1kVdFjuvr/idqEHLqkYq3aRWr2G+x1ZM2fQeB0RaHTlNpQ9gxrSFooFRSTSKK3rOpfIHyX9uQN5yPSUotA/At6LRYCqhDEIRsWVYNFFNjjjCgL4N8uyDsZfLcIsZ2zbTV/3oJSOQdm2i07pWwBI5w4J5l4oEgN7vZlq41/HK10jTHKtoYf+xoDUD6fa/R6gUvsphhXJ16Cs9OlhbljvcIweYKrnZCDR/u0GCyLs7PrIyAA1AgEFSqeb3dmI5ONwTpX8GSmLPuTOOfRaHTvbKeI3hyy0hCiJ0ePWBQ==“, 
        "Expiration": "2018-04-19T20:05:45Z", 
        "AccessKeyId": "ASIAJMXB2HHJRVWLFMXQ"
    }
}
```

Now we need to use this data. Go to your console and type:

```
export AWS_ACCESS_KEY_ID=ASIAJMXB2HHJRVWLFMXQ
export AWS_SECRET_ACCESS_KEY=Q833K5zxeaWiWI1hBbbmfTYF01Px72imuvcGB3TA
export AWS_SESSION_TOKEN=FQoDYXdzENz//////////wEaDKbdCGIckdfH7OdGcyLwAY1V/m+S38Fu/Mh+KHWNbg58kfm7047AAMYZR1kVdFjuvr/idqEHLqkYq3aRWr2G+x1ZM2fQeB0RaHTlNpQ9gxrSFooFRSTSKK3rOpfIHyX9uQN5yPSUotA/At6LRYCqhDEIRsWVYNFFNjjjCgL4N8uyDsZfLcIsZ2zbTV/3oJSOQdm2i07pWwBI5w4J5l4oEgN7vZlq41/HK10jTHKtoYf+xoDUD6fa/R6gUvsphhXJ16Cs9OlhbljvcIweYKrnZCDR/u0GCyLs7PrIyAA1AgEFSqeb3dmI5ONwTpX8GSmLPuTOOfRaHTvbKeI3hyy0hCiJ0ePWBQ==
```

Obviously you'll need to use what the previous command has returned.

Now, if you type any command using the AWS CLI it will use this user. Let's try this. Open a console and type:

```
aws s3 ls
```

You should see the same bucket than in the previous example.

### Test 4 - Automating the STS access
We're now going to create a bash script to automate what we did in our previous test. Open your favourite editor and type:

```
unset  AWS_SESSION_TOKEN

temp_role=$(aws sts assume-role \
                    --role-arn "arn:aws:iam::262170986110:role/circleci_role" \
                    --role-session-name "vgaltes-prod" \
                    --profile vgaltes-serverless)

export AWS_ACCESS_KEY_ID=$(echo $temp_role | jq .Credentials.AccessKeyId | xargs)
export AWS_SECRET_ACCESS_KEY=$(echo $temp_role | jq .Credentials.SecretAccessKey | xargs)
export AWS_SESSION_TOKEN=$(echo $temp_role | jq .Credentials.SessionToken | xargs)
```

Where role-arn is the role you want to assume and profile is your dev profile. Note that you need to have jq installed.

Give a name to the file (aws-cli-assumerole.sh, for example), give it the required execution permisions (chmod +x aws-cli-assumerole.sh) and source it (source aws-cli-assumerole.sh). Now, you have the production credentials set, so you can try to get the S3 buckets again:

```
aws s3 ls
```

You should see the production bucket as a result.


## Deploying to production using CircleCI
It's time now to use the script we've created to deploy our solution to production. Copy the script into your project, let's say inside a folder called scripts (scripts/aws-cli-assumerole.sh). Now, you need to update your config.yml in order to call the script before deploying to prod. You might need to change your dependencies sections in order to install the AWS CLI.

```
version: 2
jobs:
  build:
    docker:
      # specify the version you desire here
      - image: circleci/node:8.10

    working_directory: ~/repo

    steps:
      - checkout

      # Download and cache dependencies
      - restore_cache:
          keys:
          - v1-dependencies-{{ checksum "package.json" }}
          # fallback to using the latest cache if no exact match is found
          - v1-dependencies-

      - run:
          name: Install Serverless CLI and dependencies
          command: |
            sudo npm i -g serverless
            npm install
            sudo apt-get install awscli

      - save_cache:
          paths:
            - node_modules
          key: v1-dependencies-{{ checksum "package.json" }}
        
      # run tests!
      - run: 
          name: Run tests with coverage
          command: npm test --coverage

      - run:
          name: Deploy application
          command: sls deploy --stage pre

      - run:
          name: Deploy application
          command: |
            chmod +x scripts/aws-cli-assumerole.sh
            source scripts/aws-cli-assumerole.sh
            sls deploy --stage prod
```

Let's push this change and see if it works! (Note: the bill might take a bit more time to run now, as it has to install the AWS CLI. A possible workaround is to use a Docker image with the CLI already installed. Check the CircleCI documentation to know how to do this.)

![Production build](/images/BuildProd-1.png)

Build looks promising, let's take a look at our production account:

![Lamda deployed in production](/images/BuildProd-2.png)

Voilà! Look what's there!!

## Final touch
We're using and administrator account to deploy to production, which is not a very good idea. We should restrict the permissions of that account by setting the appropiate permissions to it creating a custom policy. In order to know which permissions you need to set, you can use the [Serverless Policy Generator](https://github.com/dancrumb/generator-serverless-policy). Another option is to go to Cloud Trail in your development account and look for the event history of your ci user, in my case *circleci*.

![Cloud trail for circleci user](/images/CloudTrail.png)
