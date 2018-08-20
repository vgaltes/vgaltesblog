---
title: 'Calling a Step Function'
thumbnail: /images/step-functions.png
date: '2017-07-09'
categories:
- serverless
tags:
- step functions
- aws
- serverless
comments: []
---
Until now we've seen how to create a Step Function, but we've always called them using the serverless framework. In this article we're going to see how to call them programatically.

We have two options to call a Step Function: the first one is to use the [API Gateway](https://aws.amazon.com/api-gateway) and create an HTTP endpoint as the Event source of the Step Function. The second one is to call the step function from a Lambda function using the [AWS SDK](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/index.html).

## HTTP endopoint
We can create the HTTP endpoint easily just modifying the serverless.yaml file. The basic code we need is the following one:

    stepFunctions:
        stateMachines:
            hello:
            events:
                - http:
                    path: hello
                    method: GET

If we want to pass some data to the Step Function, just change GET by POST. Now, we you deploy the function using serverless *sls deploy* you'll see something like this in the ouput:

    ........................
    Serverless: Stack update finished...
    Service Information
    service: TestCallStepFunction
    stage: dev
    region: us-east-1
    api keys:
    None
    endpoints:
    GET - https://awo2ongx54.execute-api.us-east-1.amazonaws.com/dev/hello
    functions:
    hello: TestCallStepFunction-dev-hello

So, if you open that link in the browser you will see the output of the call.

## Call via the SDK
The other option we have, is use the SDK to call the Step Function from a Lambda function. Let's do that using NodeJS.

First of all, install the aws sdk as a dev dependency in your project using *npm install --dev aws-sdk* or *yarn add --dev aws-sdk*.

Now you need to import the sdk in your Lambda function code:

    const AWS = require('aws-sdk');

Create the object you'll need to call the methods associated to the step functions:

    var stepfunctions = new AWS.StepFunctions();

And call the step function:

    var params = {
    stateMachineArn: '<the step function arn>',
    input: '{"value": "hello!"}'
  };

  stepfunctions.startExecution(params, function(err, data) {
    if (err) console.log(err, err.stack); // an error occurred
    else     console.log(data);           // successful response
  });

You can get the Step Function arn from the dashboard.

If you now call the Lambda *sls invoke -f hello* you will see the output of the Lambda. To see if the Step Function was called we need to go to [CloudWatch](https://aws.amazon.com/cloudwatch/) and see the logs of the invocation. So, go to your AWS console -> CloudWatch -> Logs, select the Lambda function and select the last execution.

You will see an error like this:

    { 
        AccessDeniedException: User: arn:aws:sts::165940758985:assumed-role/TestCallStepFunction-dev-us-east-1-lambdaRole/TestCallStepFunction-dev-hello is not authorized to perform: states:StartExecution on resource: arn:aws:states:us-east-1:165940758985:stateMachine:TestCall-Z249EWN421QQ
        ...
    }

So, the role serverless is creating for us to run the Lambda and the Step Function does not have permissions to start the execution of the Step Function from the Lambda. We need then to add that permission. Fortunately, serverless makes it easy. Just add this piece of code inside the provider section of the serverless.yml file:

    iamRoleStatements:
    -  Effect: "Allow"
       Action:
         - "states:StartExecution"
       Resource: <the step function arn>

Now, if we go to the CloudWatch logs we will see no error.

## Making the code robust
What's the problem with the current code? The problem is that we're hardcoding the arn of the Step Function in the serverless.yml file and in the code of the Lambda function. If we re-deploy the Step Function, its arn will change and we'll need to change both files. Not very practical. Wouldn't be better if we can export the arn of the Step Function and use it everywhere? [Environment variables](http://docs.aws.amazon.com/lambda/latest/dg/env_variables.html) to the rescue!

Let's start by defining an output that will export the arn of the Step Function to an environment variable. Open the serverless.yml file and add:

    resources:
        Outputs:
            TestCall:
            Description: The ARN of the state machine
            Value:
                Ref: TestCall

Now we need to use this environment variable when we're defining the IAM role. Let's change the resource value to use the environment variable:

    iamRoleStatements:
    -  Effect: "Allow"
       Action:
         - "states:StartExecution"
       Resource: ${self:resources.Outputs.TestCall.Value}

The next step is to pass the environment variable to the Lambda function. We can do that easily just adding a couple of lines in the Lambda function definition:

    functions:
        hello:
            handler: handler.hello
            events:
                - http:
                    path: hello
                    method: GET
            environment:
                testcall_arn: ${self:resources.Outputs.TestCall.Value}

And finally, we need to change the code of the Lambda to use the environment variable:

    var params = {
        stateMachineArn: process.env.testcall_arn,
        input: '{"value": "hello!"}'
    };

With these changes, we can redeploy the Step Function and the Lambda function and we'll continue to be able to call the Step Function from the Lambda function.

## Summary
We've seen how we can interact programmatically with a Step Function, using the API Gateway or using the AWS SDK inside a Lambda function.