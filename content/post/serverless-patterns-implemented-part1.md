---
title: 'Serverless Patterns implemented, part 1'
date: '2019-11-013'
categories:
- serverless
tags:
- patterns
- serverless framework
---

I think that the best way to learn something is to practice it and to try to explain it, so this is what I'm going to do in the next series of posts. These posts will be based on the amazing article from [Jeremy Daly](https://twitter.com/jeremy_daly) about [Serverless Patterns](https://www.jeremydaly.com/serverless-microservice-patterns-for-aws). I'm not going to copy Jeremy's words here, so for each pattern, go to the article and read it. I'll provide a technical implementation here and I will mention more resources I found interesting. Let's start!

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


## The Simple Webservice
Read the explanation [here](https://www.jeremydaly.com/serverless-microservice-patterns-for-aws/#simplewebservice)

To implement this pattern we need to create a service with a DynamoDB table and, at least, a function that get or sets data from it. So the `serverless.yml` will look like this:
```
service: SimpleWebService

plugins:
  - serverless-iam-roles-per-function

provider:
  name: aws
  runtime: nodejs10.x
  region: ${opt:region, self:custom.defaultRegion}

custom:
  defaultRegion: eu-west-1
  tableName: ${self:provider.region}-SimpleWebServiceTable

functions:
  GetItem:
    handler: src/functions/getItem.handler
    events:
      - http:
          method: get
          path: item/{itemId}
    environment:
      tableName: ${self:custom.tableName}
    iamRoleStatements:
      - Effect: Allow
        Action: dynamodb:getItem
        Resource: !GetAtt SimpleWebServiceTable.Arn

  PutItem:
    handler: src/functions/putItem.handler
    events:
      - http:
          method: post
          path: item
    environment:
      tableName: ${self:custom.tableName}
    iamRoleStatements:
      - Effect: Allow
        Action: dynamodb:putItem
        Resource: !GetAtt SimpleWebServiceTable.Arn

resources:
  Resources:
    SimpleWebServiceTable:
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

We'll need to install the `serverless-iam-roles-per-function` plugin. We're creating a DyanamoDB table and passing the name to the functions via environment variable. In each function, we're just giving the permissions it needs.

Let's take a look at the implementation of the PutItem function:
```
const AWS = require("aws-sdk");

const dynamodb = new AWS.DynamoDB.DocumentClient();
const tableName = process.env.tableName;

module.exports.handler = async (event) => {
  const body = JSON.parse(event.body);
  const id = parseInt(body.id);
  const name = body.name;

  const params = {
    TableName: tableName,
    Item: {
      'id' : id,
      'name' : name
    }
  };

  const resp = await dynamodb.put(params).promise();

  const res = {
    statusCode: 200,
    body: JSON.stringify(resp)
  };

  return res;
};
```

And finally let's take a look at the implementation of the GetItem function:
```
const AWS = require("aws-sdk");

const dynamodb = new AWS.DynamoDB.DocumentClient();
const tableName = process.env.tableName;

module.exports.handler = async (event) => {
  const id = event.pathParameters.itemId;

  const req = {
    TableName: tableName,
    Key: {
        'id': parseInt(id)
      }
  };

  const resp = await dynamodb.get(req).promise();

  const res = {
    statusCode: 200,
    body: JSON.stringify(resp.Item)
  };

  return res;
};
```

You can check the full solution [here](https://github.com/vgaltes/ServerlessPatterns/tree/master/SimpleWebService).

## The Scalable Webhook
Read the explanation [here](https://www.jeremydaly.com/serverless-microservice-patterns-for-aws/#scalablewebhook).

In this pattern, we're going to introduce an SQS queue between two services and that queue will have a dead letter queue in case we find some error. So, let´s start with the `serverless.yml` file:

```
service: ScalableWebhook

plugins:
  - serverless-iam-roles-per-function

provider:
  name: aws
  runtime: nodejs10.x
  region: ${opt:region, self:custom.defaultRegion}

custom:
  defaultRegion: eu-west-1

functions:
  Flooder:
    handler: src/functions/flooder.handler
    events:
      - http:
          method: post
          path: flooder
    environment:
      queueUrl: !Ref WorkerQueue
    iamRoleStatements:
      - Effect: Allow
        Action: SQS:SendMessage
        Resource: !GetAtt WorkerQueue.Arn

  Worker:
    handler: src/functions/worker.handler
    memorySize: 256
    reservedConcurrency: 5
    events:
      - sqs:
          batchSize: 10
          arn: !GetAtt WorkerQueue.Arn

  DLQReader:
    handler: src/function/dlqReader.handler
    events:
      - sqs:
          batchSize: 10
          arn: !GetAtt ReceiverDeadLetterQueue.Arn

resources:
  Resources:
    WorkerQueue:
      Type: "AWS::SQS::Queue"
      Properties:
        QueueName: "WorkerQueue"
        VisibilityTimeout: 30 # 20 seconds
        MessageRetentionPeriod: 60 # 60 seconds
        RedrivePolicy:
          deadLetterTargetArn: !GetAtt ReceiverDeadLetterQueue.Arn
          maxReceiveCount: 3

    ReceiverDeadLetterQueue:
      Type: "AWS::SQS::Queue"
      Properties:
        QueueName: "WorkerDLQ"
        MessageRetentionPeriod: 1209600 # 14 days in seconds

```

This file is a bit more complicated than the previous one. The interesting bits are in the queue definition, where we're setting the properties of the queue. We're going to explain now what those parameters are, but I strongly recommend you to read [this article](https://www.jeremydaly.com/serverless-consumers-with-lambda-and-sqs-triggers/) from Jeremy about SQS queues, and please read all the comments as well as there is interesting information there. You can also take a look at [this article](https://cloudly.tech/blog/aws-lambda-sqs-trigger-error-handling/) where the author explains how the error handling in SQS works. And finally, you can go to the [official documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-dead-letter-queue.html).

The [visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html) is the time that the message remains in the queue without other consumers being able to receive and process the message, waiting for the confirmation (or the error) from the original consumer. If, after this time the queue has not receive the deletion request from the original consumer, the queue makes the message available for the next consumer.

The message retention period is the time a message is placed on a queue before being deleted by the system if nobody consumes it. The maximum is 14 days.

The [redrive policy](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-dead-letter-queue.html) is where we specify what happens when a message can't be processed by the consumer. In our case, we're saying that we would like to specify a DLQ and that a messsage will go to the DLQ after three failed attempts to be processed.

The code of the flooder is the same (or very similar) than the one Jeremy has in his post about SQS:
```
const AWS = require('aws-sdk');
const SQS = new AWS.SQS();

module.exports.handler = async (event, context) => {
    const body = JSON.parse(event.body);
    const times = parseInt(body.times);
    const queue = process.env.queueUrl;

    console.log(`Queue is: ${queue}`);

    for (let i=0; i<times; i++) {
        await SQS.sendMessageBatch({ Entries: createMessages(), QueueUrl: queue }).promise()
    }

    return {
        statusCode: 200,
        body: JSON.stringify("all done")
    };
}

const createMessages = () => {
    let entries = []
   
    for (let i=0; i<10; i++) {
        entries.push({
          Id: 'id'+parseInt(Math.random()*1000000),
          MessageBody: 'value'+Math.random()
        })
    }
    return entries
}
```

And the code of the worker and the DLQReader are basically the same:
```
let counter = 1
let messageCount = 0
let funcId = 'id'+parseInt(Math.random()*1000)
 
module.exports.handler = async (event) => {
    counter++;

    if (counter % 10 === 0){
        throw new Error('Simulating error');
    }
    
    // Record number of messages received
    if (event.Records) {
        messageCount += event.Records.length
    }
    console.log(funcId + ' REUSE: ', counter)
    console.log(funcId + ' Message Count: ', messageCount)
    console.log(JSON.stringify(event))
    console.log(funcId + ' processing...');
    await sleep(2000);
    console.log(funcId + ' job done!');
    return 'done'
};

const sleep = (milliseconds) => {
    return new Promise(resolve => setTimeout(resolve, milliseconds))
}
```

You can check the full solution [here](https://github.com/vgaltes/ServerlessPatterns/tree/master/ScalableWebhook).

## Summary
In this article, we've seen the implementation from a couple of patterns from [this](https://www.jeremydaly.com/serverless-microservice-patterns-for-aws) great article from [Jeremy Daly](https://twitter.com/jeremy_daly). We'll continue with that in following article.

Hope it helps!!
