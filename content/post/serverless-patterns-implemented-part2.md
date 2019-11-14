---
title: 'Serverless Patterns implemented, part 2'
date: '2019-11-14'
categories:
- serverless
tags:
- patterns
- serverless framework
---

In this article I will continue with the implementation of some Serverless Patterns described in [this](https://www.jeremydaly.com/serverless-microservice-patterns-for-aws) article from [Jeremy Daly](https://twitter.com/jeremy_daly) about [Serverless Patterns]. Check the first post [here](https://vgaltes.com/post/serverless-patterns-implemented-part1/) Let's start!

## Common setup
All the projects will have a common setup, which is fairly simple. First, initialize a NodeJS project:
```
yarn init
```

Then install the serverless framework as a dev dependency
```
yarn add serverless --dev
```

And finally create a script to deploy the project
```
"scripts": {
    "deploy": "serverless deploy --aws-profile serverless-local"
  }
```

(Assuming that you have a profile called serverless-local, of course)


## The Gatekeeper
Read the article [here](https://www.jeremydaly.com/serverless-microservice-patterns-for-aws/#gatekeeper).

The big difference with previous patterns is that we need a custom Lambda authorizer. Let's take a look at the `serverless.yml` file:

```
service: Gatekeeper

plugins:
  - serverless-pseudo-parameters
  - serverless-iam-roles-per-function

provider:
  name: aws
  runtime: nodejs10.x
  region: ${opt:region, self:custom.defaultRegion}
  logs:
    restApi: true

custom:
  defaultRegion: eu-west-1
  tableName: ${self:provider.region}-GatekeeperTable
  authorizerTableName: ${self:provider.region}-GatekeeperAuthorizerTable

functions:
  GetItem:
    handler: src/functions/getItem.handler
    events:
      - http:
          method: get
          path: item/{itemId}
          authorizer: 
            name: CustomAuthorizer
            resultTtlInSeconds: 0
            identitySource: method.request.header.Authorization
            type: token
    environment:
      tableName: ${self:custom.tableName}
    iamRoleStatements:
      - Effect: Allow
        Action: dynamodb:getItem
        Resource: !GetAtt GatekeeperTable.Arn

  PutItem:
    handler: src/functions/putItem.handler
    events:
      - http:
          method: post
          path: item
          authorizer: 
            name: CustomAuthorizer
            resultTtlInSeconds: 0
            identitySource: method.request.header.Authorization
            type: token
            
    environment:
      tableName: ${self:custom.tableName}
    iamRoleStatements:
      - Effect: Allow
        Action: dynamodb:putItem
        Resource: !GetAtt GatekeeperTable.Arn

  CustomAuthorizer:
    handler: src/functions/authorizer.handler
    environment:
      tableName: ${self:custom.authorizerTableName}
    iamRoleStatements:
      - Effect: Allow
        Action: dynamodb:getItem
        Resource: !GetAtt AuthorizationTable.Arn

resources:
  Resources:
    GatekeeperTable:
      Type: AWS::DynamoDB::Table
      Properties:
        KeySchema:
          - AttributeName: id
            KeyType: 'HASH'
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: 'N'
        BillingMode: PAY_PER_REQUEST
        TableName: ${self:custom.tableName}

    AuthorizationTable:
      Type: AWS::DynamoDB::Table
      Properties:
        KeySchema:
          - AttributeName: id
            KeyType: 'HASH'
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: 'N'
        BillingMode: PAY_PER_REQUEST
        TableName: ${self:custom.authorizerTableName}
```

We're creating a new table for the authorization, where you can store what you need to authorize a user. The custom authorizer lambda is just another function. The difference in the other functions is that we're now setting the authorizer. In the name property we're setting the name of the lambda function that will authorize the request, in the identitySource we're setting the header that we'd like to use and in the type property we're it as a token, which is the simpler one. If you want to know more about custom authorizers, check [this article](https://www.alexdebrie.com/posts/lambda-custom-authorizers/) from [Alex DeBrie](https://twitter.com/alexbdebrie).

Let's take a look at the code of the custom authorizer function now.

```
const AWS = require("aws-sdk");

const dynamodb = new AWS.DynamoDB.DocumentClient();
const tableName = process.env.tableName;

module.exports.handler = async (event, context) => {
  console.log(event);

  const id = event.authorizationToken;

  const req = {
    TableName: tableName,
    Key: {
        'id': parseInt(id)
      }
  };

  const dynamodbResp = await dynamodb.get(req).promise();

  console.log(dynamodbResp);

  if (!dynamodbResp.Item){
    // 401
    context.fail('Unauthorized');

    // 403
    // context.succeed({
    //   "policyDocument": {
    //     "Version": "2012-10-17",
    //     "Statement": [
    //       {
    //         "Action": "execute-api:Invoke",
    //         "Effect": "Deny",
    //         "Resource": [
    //           event.methodArn
    //         ]
    //       }
    //     ]
    //   }
    // })
  }

  context.succeed( 
  {
    "principalId": dynamodbResp.Item.name,
    "policyDocument": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Action": "execute-api:Invoke",
          "Effect": "Allow",
          "Resource": event.methodArn
        }
      ]
    },
    "context": {
      "org": "my-org",
      "role": "admin",
      "createdAt": "2019-11-11T12:15:42"
    }
  });
};
```

The logic of the authorizer is not important here. What is important is that you'll receive the token (the value of the authorization header in our case) in the `authorizationToken` of the `event`. The other important bit is what do we have to return in a custom authorizer. There are three main cases here:
 - **401**: you need to call `context.fail('Unauthorized');`
 - **Success**: you need to call `context.success` passing a policy. In this policy you need to specify the principalId of the user, and as *Statement* a valid IAM Policy that allows access to the endpoint. The endpoint Arn comes in the event in the property `methodArn`. You can pass an object in the *context* object to add custom data to the context object that you internal lambda will receive.
 - **403**: you need to call `contex.success` but with a Deny in the IAM Policy.

You can check the code [here](https://github.com/vgaltes/ServerlessPatterns/tree/master/Gatekeeper).

## Internal API
You can read the article [here](https://www.jeremydaly.com/serverless-microservice-patterns-for-aws/#internalapi).

This pattern is much simpler, but what will change is the way that the lambda is invoked. So, the `serverless.yml` is fairly simple:

```
service: InternalAPI

plugins:
  - serverless-iam-roles-per-function

provider:
  name: aws
  runtime: nodejs10.x
  region: ${opt:region, self:custom.defaultRegion}

custom:
  defaultRegion: eu-west-1
  tableName: ${self:provider.region}-InternalAPITable

functions:
  GetItem:
    handler: src/functions/getItem.handler
    environment:
      tableName: ${self:custom.tableName}
    iamRoleStatements:
      - Effect: Allow
        Action: dynamodb:getItem
        Resource: !GetAtt InternalAPITable.Arn

  PutItem:
    handler: src/functions/putItem.handler
    environment:
      tableName: ${self:custom.tableName}
    iamRoleStatements:
      - Effect: Allow
        Action: dynamodb:putItem
        Resource: !GetAtt InternalAPITable.Arn

resources:
  Resources:
    InternalAPITable:
      Type: AWS::DynamoDB::Table
      Properties:
        KeySchema:
          - AttributeName: id
            KeyType: 'HASH'
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: 'N'
        BillingMode: PAY_PER_REQUEST
        TableName: ${self:custom.tableName}
```

Notice that now the Lambda functions don't have any event. So, how can we call those lambdas? Let's create a script to call the `PutItem` function.

```
const AWS = require('aws-sdk');

AWS.config.region = "eu-west-1";
var lambda = new AWS.Lambda();

const base64data = Buffer.from('{"AppName" : "InternalAPIApp"}').toString('base64');

var params = {
    ClientContext: base64data, 
    FunctionName: "InternalAPI-dev-PutItem", 
    InvocationType: " RequestResponse", 
    LogType: "Tail", // Set to Tail to include the execution log in the response.
    Payload: '{"id": 1, "name": "test1"}'
};

lambda.invoke(params, function(err, data) {
    if (err) console.log(err, err.stack); // an error occurred
    else     console.log(data);           // successful response
});

```

As you can see we need to use the AWS SDK. You can install it using `yarn add aws-sdk --dev`. The important bits here is that we need to specify the name of the function. The invocation type righ now (we'll see the other type in the next pattern) is `RequestResponse` which means that we will wait for a response from the lambda. 

So, assuming that you name this file `callPutItems.js` and that you have a profile called `serverless-local` you will need to call this script like `AWS_PROFILE=serverless-local node callPutItem.js`.

You can check the code [here](https://github.com/vgaltes/ServerlessPatterns/tree/master/InternalAPI).

## The Internal Handoff
Read the article [here](https://www.jeremydaly.com/serverless-microservice-patterns-for-aws/#internalhandoff).

This pattern is fairly similar to the previous one, with a couple of differences:
 - We're going to call the lambda with an invocation type of *event*, which will make the call asynchronous.
 - We're going to add a DLQ (in our case an SNS topic) for the failing messages.

Let's take a look at the `serverless.yml` file first:

```
service: InternalHandoff

plugins:
  - serverless-iam-roles-per-function
  - serverless-pseudo-parameters

provider:
  name: aws
  runtime: nodejs10.x
  region: ${opt:region, self:custom.defaultRegion}
  stage: ${opt:stage, self:custom.defaultStage}

custom:
  defaultRegion: eu-west-1
  defaultStage: dev
  tableName: ${self:provider.stage}-InternalHandofffTable
  dlqTopicName: ${self:provider.stage}-DLQTopicName
  dlqTopicArn: arn:aws:sns:#{AWS::Region}:#{AWS::AccountId}:${self:custom.dlqTopicName}

functions:
  GetItem:
    handler: src/functions/getItem.handler
    environment:
      tableName: ${self:custom.tableName}
    onError: ${self:custom.dlqTopicArn}
    iamRoleStatements:
      - Effect: Allow
        Action: dynamodb:getItem
        Resource: !GetAtt InternalHandofffTable.Arn
      - Effect: Allow
        Action: sns:Publish
        Resource: ${self:custom.dlqTopicArn}

  PutItem:
    handler: src/functions/putItem.handler
    environment:
      tableName: ${self:custom.tableName}
    onError: ${self:custom.dlqTopicArn}
    iamRoleStatements:
      - Effect: Allow
        Action: dynamodb:putItem
        Resource: !GetAtt InternalHandofffTable.Arn
      - Effect: Allow
        Action: sns:Publish
        Resource: ${self:custom.dlqTopicArn}

  ReadErrors:
    handler: src/functions/readErrors.handler
    events:
      - sns: ${self:custom.dlqTopicName}

resources:
  Resources:
    InternalHandofffTable:
      Type: AWS::DynamoDB::Table
      Properties:
        KeySchema:
          - AttributeName: id
            KeyType: 'HASH'
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: 'N'
        BillingMode: PAY_PER_REQUEST
        TableName: ${self:custom.tableName}
```

The main difference in the functions is that we've set the property `onError`. We need to set this property to the Arn of an SNS topic (you can't use a SQS right now, chech the why [here](https://serverless.com/framework/docs/providers/aws/guide/functions#dlq-with-sqs)). When we do that, we need to add a permission to be able to write to that topic.

Finally, we're creating a function that reads from that topic. When we do this, the framework will create the SNS topic for us. If we don't specify any function, we will need to create the topic in the `resources` section.

To check the DLQ, we're setting a condition in the code of our functions to throw an error if the name of the item is *error*. Let's see how the script that calls the function changes:

```
const AWS = require('aws-sdk');

const name = process.argv.slice(2)[0];

AWS.config.region = "eu-west-1";
var lambda = new AWS.Lambda();

const base64data = Buffer.from('{"AppName" : "InternalAPIApp"}').toString('base64');

var params = {
    ClientContext: base64data, 
    FunctionName: "InternalHandoff-dev-PutItem", 
    InvocationType: "Event", 
    LogType: "Tail", // Set to Tail to include the execution log in the response.
    Payload: `{"id": 1, "name": "${name}"}`
};

lambda.invoke(params, function(err, data) {
    if (err) console.log(err, err.stack); // an error occurred
    else     console.log(data);           // successful response
});
```

As you can see, the invocation type is now *Event*. Doing that, we'll receive a 202 from the function instead of a 200.

To call this script to generate an error you need to call it this way:
```
AWS_PROFILE=serverless-local node callPutItem.js error
```

(Use any other name if you don't want to generate an error.)

When we do that, AWS will try to deliver the message three times (aproximately once a minute). If we fail the three times, it will send the message to the DLQ.

You can check the code [here](https://github.com/vgaltes/ServerlessPatterns/tree/master/InternalHandoff).

## Summary
In this article, we've seen the implementation of three more patterns from [this](https://www.jeremydaly.com/serverless-microservice-patterns-for-aws) great article from [Jeremy Daly](https://twitter.com/jeremy_daly). We'll continue with that in following article.

Hope it helps!!
