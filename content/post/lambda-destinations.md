---
title: 'Lambda destinations'
date: '2020-03-29'
categories:
- serverless
tags:
- patterns
- serverless framework
---

AWS recently introduced [lambda destinations](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html#invocation-async-destinations) for asynchronous invocations. So, if you have, let's say, a lambda function attached to an SNS event, you can configure a destination when the execution is successful and a destination when the execution fails. The destination can be either an SQS queue, an SNS topic, EventBridge or another Lambda function.

As usual, the [serverless framework](https://serverless.com/framework/docs/providers/aws/guide/functions#destinations) implemented this feature quickly. Let's take a look how to do it and what's the difference with a DLQ.

## Project set up
We're going to demonstrate this by creating a simple project with an asynchronous invocation. We'll have a lambda triggered by an HTTP event that will put a message in an SNS topic. A lambda will be attached to this topic and will trigger an error under some circumstances. This will be the `serverless.yml` file for this scenario:

```
service: demo-destinations

provider:
  name: aws
  runtime: nodejs12.x
  region: ${opt:region, self:custom.defaultRegion}
  stage: ${opt:stage, self:custom.defaultStage}

plugins:
  - serverless-iam-roles-per-function
  - serverless-pseudo-parameters

custom:
  defaultRegion: eu-west-1
  defaultStage: dev${env:SLSUSER, ""}
  snsTopicName: ${self:service}-topic
  snsTopic: arn:aws:sns:#{AWS::Region}:#{AWS::AccountId}:${self:custom.snsTopicName}

functions:
  entryPoint:
    handler: src/functions/entryPoint.handler
    events:
      - http:
          path: api/start
          method: post
    iamRoleStatements:
      - Effect: Allow
        Action: sns:Publish
        Resource: !Ref SnsTopic
    environment:
      snsTopic: ${self:custom.snsTopic}
  functionWithError:
    handler: src/functions/functionWithError.handler
    events:
      - sns: 
          arn: !Ref SnsTopic
          topicName: ${self:custom.snsTopicName}

resources:
  Resources:
    SnsTopic:
      Type: AWS::SNS::Topic
      Properties: 
        TopicName: ${self:custom.snsTopicName}
```

This is our entry point function. As you can see, we need code to put a message on the SNS topic.

```
const AWS = require("aws-sdk");
const SNS = new AWS.SNS();

module.exports.handler = async (event) => {
    console.log(JSON.stringify(event));

    const body = JSON.parse(event.body);
    const type = body.type;
    const message = body.message;
    const data = {
        type,message
    }

    const params = {
        Message: JSON.stringify(data),
        TopicArn: process.env.snsTopic
      };
    
      await SNS.publish(params).promise();

    return {
        statusCode: 200,
        body: JSON.stringify({ message })
    };
};

```

And this is the code of the function that is attached to the SNS topic

```
module.exports.handler = async (event) => {
    console.log(JSON.stringify(event));
    const snsMessage = JSON.parse(event.Records[0].Sns.Message);

    const type = snsMessage.type;
    const message = snsMessage.message;

    if (type === "error"){
        throw new Error("Simulating error");
    }
    
    console.log(`Function not erroring. Dispatching to success destination. Message is ${message}`);

    return {
        statusCode: 200,
        body: JSON.stringify({ message })
    };
};
```

## Testing the DLQ
Until we could use destinations, a DLQ was the standard mechanism to handle errors in asynchronous invocations. We can define the onError property on the function to specify the SNS topic we'd like the messages to be sent in case of error. Let's do this step by step.

First, define the DLQ topic name in the custom section.

```
DLQSnsTopicName: ${self:service}-dlqTopic
```

Next, describe the topic in the resources section:

```
DLQSnsTopic:
  Type: AWS::SNS::Topic
  Properties: 
    TopicName: ${self:custom.DLQSnsTopicName}
```

And now, change the definition of the function to include the onError property and the required permissions:

```
functionWithError:
  handler: src/functions/functionWithError.handler
  events:
    - sns: 
        arn: !Ref SnsTopic
        topicName: ${self:custom.snsTopicName}
  onError: !Ref DLQSnsTopic
  iamRoleStatements:
    - Effect: Allow
      Action: sns:Publish
      Resource: !Ref DLQSnsTopic
```

Finally, we can create a function that subscribes to that queue:

```
functionWithErrorDLQ:
  handler: src/functions/errorDLQ.handler
  events:
    - sns: 
        arn: !Ref DLQSnsTopic
        topicName: ${self:custom.DLQSnsTopicName}
```

And this function can have the following code:
```
module.exports.handler = async (event) => {
    console.log(JSON.stringify(event));
}
```

Now you can deploy this and call the HTTP endpoint to generate an error in the downstream lambda function. Something like this: `curl https://xxxxxx.execute-api.eu-west-1.amazonaws.com/dev/api/start --data '{"type":"error", "message": "hello"}'`

AWS will try to deliver the message three times, and after that, it will put the message in the DLQ. This is an example of the message that will go to the DLQ:

```
{
    "Records": [
        {
            "EventSource": "aws:sns",
            "EventVersion": "1.0",
            "EventSubscriptionArn": "arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-dlqTopic:c7e1b02b-6f04-4836-8700-83361c10c64f",
            "Sns": {
                "Type": "Notification",
                "MessageId": "9695054e-a478-5fde-952f-53b41ccac1db",
                "TopicArn": "arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-dlqTopic",
                "Subject": null,
                "Message": "{\"Records\":[{\"EventSource\":\"aws:sns\",\"EventVersion\":\"1.0\",\"EventSubscriptionArn\":\"arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-topic:37ded1fd-a827-49c0-9963-660867649cc4\",\"Sns\":{\"Type\":\"Notification\",\"MessageId\":\"b83135dc-14b4-5ab3-b336-6326acbf55bb\",\"TopicArn\":\"arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-topic\",\"Subject\":null,\"Message\":\"{\\\"type\\\":\\\"error\\\",\\\"message\\\":\\\"hello\\\"}\",\"Timestamp\":\"2020-03-29T14:50:51.599Z\",\"SignatureVersion\":\"1\",\"Signature\":\"IHDgnyGLc9ueIknZbWB4/wx1R/mzuxKomNAZlLScn+c1u+2tvlyJkIFzXsUEKaF67uRDxEMDHMhVL4GWKyBVl18J/awiOUfAxFtHx0pneQ3kmo23lGYi4wNtQ+DrDJhA9E2TfwvGnU/OGhdifWYl2Tw2aVPYcha8WuCxCHTDRZHsxs3UXpRe6rS7gcX5/OtJBJGVH8aAOeW7/oK1hSJ9pM5K17FQdCOg/nISTarwyBqM/ddEV8PS8CTrtn9rxmFCgtM4aLdO3hHdVE3O3yBuMlyJBWGzmXf/+gHwCwczNMW/u/zB1I6A1T07/pL4tYlrQaBl3AkOnJtK8aC6kTvpIQ==\",\"SigningCertUrl\":\"https://sns.eu-west-1.amazonaws.com/SimpleNotificationService-a86cb10b4e1f29c941702d737128f7b6.pem\",\"UnsubscribeUrl\":\"https://sns.eu-west-1.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-topic:37ded1fd-a827-49c0-9963-660867649cc4\",\"MessageAttributes\":{}}}]}",
                "Timestamp": "2020-03-29T14:53:38.631Z",
                "SignatureVersion": "1",
                "Signature": "YHIpSEbZb1E73lANC/yqh8JG+9EkNFfp+eTAmamfdk6CLYWnfhkurURsP0Cl8kiMTI96KRwHg9FCQcQ3UWzizQIvDsPL35sDOFJBOPLhLL6Itf8kgbAnXx2uWEZUmA//UQGEd6vWkNv3nN3QM43y0UrmzlAMVSS4az1Iv/cG4xqF9JIfapU2wGrvrhAikw4R0LL6qdJd0mzrsZdphWMWEgnZN7/7tlh14tMExBe0zujbp88+c8axzTNjH8l6+KctGPdjZIJZjo0Np25O4dSY7+uGwzc7ogrY6vMUj5Mj8TJCAh7eMbPDEDV3FnESJXKv2epL0mGpc5LWBLShcWwsfg==",
                "SigningCertUrl": "https://sns.eu-west-1.amazonaws.com/SimpleNotificationService-a86cb10b4e1f29c941702d737128f7b6.pem",
                "UnsubscribeUrl": "https://sns.eu-west-1.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-dlqTopic:c7e1b02b-6f04-4836-8700-83361c10c64f",
                "MessageAttributes": {
                    "RequestID": {
                        "Type": "String",
                        "Value": "86162f4e-6b2c-492f-93de-5e1b9c5f8bd3"
                    },
                    "ErrorCode": {
                        "Type": "String",
                        "Value": "200"
                    },
                    "ErrorMessage": {
                        "Type": "String",
                        "Value": "Simulating error"
                    }
                }
            }
        }
    ]
}
```

That's quite good. Let's take a look at what we have to do to get to the same place using destinations.

## Error destination
First of all, we need to create the function that will get called:

```
errorDestination:
    handler: src/functions/errorDestination.handler
```

As you can see, we don't need to specify any event, as we'll configure the destination to directly call this function.

This function can have the following code:
```
module.exports.handler = async (event) => {
    console.log(JSON.stringify(event));
}
```

Next, we need to update the main function definition to add the error destination, with the required permissions:

```
functionWithError:
    handler: src/functions/functionWithError.handler
    events:
      - sns: 
          arn: !Ref SnsTopic
          topicName: ${self:custom.snsTopicName}
    destinations:
      onFailure: errorDestination
    onError: !Ref DLQSnsTopic
    iamRoleStatements:
      - Effect: Allow
        Action: sns:Publish
        Resource: !Ref DLQSnsTopic
      - Effect: Allow
        Action: lambda:InvokeFunction
        Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${self:provider.stage}-errorDestination 
```

And that's all. Let's make the same call and the function will be called at the same time than the DLQ, after the last retry. Let's take a look at the event we get there:

```
{
    "version": "1.0",
    "timestamp": "2020-03-29T14:53:38.705Z",
    "requestContext": {
        "requestId": "86162f4e-6b2c-492f-93de-5e1b9c5f8bd3",
        "functionArn": "arn:aws:lambda:eu-west-1:XXXXXXXXXXX:function:demo-destinations-dev-functionWithError:$LATEST",
        "condition": "RetriesExhausted",
        "approximateInvokeCount": 3
    },
    "requestPayload": {
        "Records": [
            {
                "EventSource": "aws:sns",
                "EventVersion": "1.0",
                "EventSubscriptionArn": "arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-topic:37ded1fd-a827-49c0-9963-660867649cc4",
                "Sns": {
                    "Type": "Notification",
                    "MessageId": "b83135dc-14b4-5ab3-b336-6326acbf55bb",
                    "TopicArn": "arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-topic",
                    "Subject": null,
                    "Message": "{\"type\":\"error\",\"message\":\"hello\"}",
                    "Timestamp": "2020-03-29T14:50:51.599Z",
                    "SignatureVersion": "1",
                    "Signature": "IHDgnyGLc9ueIknZbWB4/wx1R/mzuxKomNAZlLScn+c1u+2tvlyJkIFzXsUEKaF67uRDxEMDHMhVL4GWKyBVl18J/awiOUfAxFtHx0pneQ3kmo23lGYi4wNtQ+DrDJhA9E2TfwvGnU/OGhdifWYl2Tw2aVPYcha8WuCxCHTDRZHsxs3UXpRe6rS7gcX5/OtJBJGVH8aAOeW7/oK1hSJ9pM5K17FQdCOg/nISTarwyBqM/ddEV8PS8CTrtn9rxmFCgtM4aLdO3hHdVE3O3yBuMlyJBWGzmXf/+gHwCwczNMW/u/zB1I6A1T07/pL4tYlrQaBl3AkOnJtK8aC6kTvpIQ==",
                    "SigningCertUrl": "https://sns.eu-west-1.amazonaws.com/SimpleNotificationService-a86cb10b4e1f29c941702d737128f7b6.pem",
                    "UnsubscribeUrl": "https://sns.eu-west-1.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-topic:37ded1fd-a827-49c0-9963-660867649cc4",
                    "MessageAttributes": {}
                }
            }
        ]
    },
    "responseContext": {
        "statusCode": 200,
        "executedVersion": "$LATEST",
        "functionError": "Unhandled"
    },
    "responsePayload": {
        "errorType": "Error",
        "errorMessage": "Simulating error",
        "trace": [
            "Error: Simulating error",
            "    at Runtime.module.exports.handler (/var/task/src/functions/functionWithError.js:9:15)",
            "    at Runtime.handleOnce (/var/runtime/Runtime.js:66:25)"
        ]
    }
}
```

The main difference here is that you get a bit more information in this event, with the responseContext and responsePayload properties. 

So, you get more information with less hassle, as you don't have to create the topic. It looks like a winner to me!

## Success destination
We can do a very similar thing when the function succeeds. Let's do this step by step, although it will be the same as the error destination.

Let's start creating the success function:
```
successDestination:
  handler: src/functions/errorDestination.handler
```

With the same code:

```
module.exports.handler = async (event) => {
    console.log(JSON.stringify(event));
}
```

Now it's time to update the main function definition to add this destination.
```
functionWithError:
    handler: src/functions/functionWithError.handler
    events:
      - sns: 
          arn: !Ref SnsTopic
          topicName: ${self:custom.snsTopicName}
    destinations:
      onFailure: errorDestination
      onSuccess: successDestination
    onError: !Ref DLQSnsTopic
    iamRoleStatements:
      - Effect: Allow
        Action: sns:Publish
        Resource: !Ref DLQSnsTopic
      - Effect: Allow
        Action: lambda:InvokeFunction
        Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${self:provider.stage}-errorDestination
      - Effect: Allow
        Action: lambda:InvokeFunction
        Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${self:provider.stage}-successDestination
```

And that's all, let's call the entry point with a payload that doesn't create an error:
`curl https://ukcpjjf9ec.execute-api.eu-west-1.amazonaws.com/dev/api/start --data '{"type":"success", "message": "hello"}'`

And we'll get this message in the function:

```
{
    "version": "1.0",
    "timestamp": "2020-03-29T15:53:48.122Z",
    "requestContext": {
        "requestId": "965949be-107c-4d02-b5c3-de7645cb7671",
        "functionArn": "arn:aws:lambda:eu-west-1:XXXXXXXXXXX:function:demo-destinations-dev-functionWithError:$LATEST",
        "condition": "Success",
        "approximateInvokeCount": 1
    },
    "requestPayload": {
        "Records": [
            {
                "EventSource": "aws:sns",
                "EventVersion": "1.0",
                "EventSubscriptionArn": "arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-topic:37ded1fd-a827-49c0-9963-660867649cc4",
                "Sns": {
                    "Type": "Notification",
                    "MessageId": "4cbf0cb9-ce8f-59fd-bfe1-5a4308445979",
                    "TopicArn": "arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-topic",
                    "Subject": null,
                    "Message": "{\"type\":\"success\",\"message\":\"hello\"}",
                    "Timestamp": "2020-03-29T15:53:47.470Z",
                    "SignatureVersion": "1",
                    "Signature": "gYlc07lDjG+gAe1gD9YlYUgSab9L64NgpPOk2Jx9boAioK1opKK3owNCY9uZWciO2PeFCkDCinhp9JZyUEenEXOcglLhnlnhQeUbpibvHmzRo5jdJa5bjBjkLAZ1ivo8O9r9KhZI+pjUEETLzmwUljp+pvi5PxXw4GKZZaCbQoZRNrkBlNIkJ/gMLfaCngTFQRgZ4PFLdKTT5YYg+F1V51reCyaBIDV1tT9LAHBIp3yMT7LhC0GssmToYp4MNOVUBx1HmJeODl+GjlYI0249tQfZZGqVB4DL8r8dnjHMSqOys7coswIe64UWj2yn78uUxN12+UyXbsdRQm75qlLZMg==",
                    "SigningCertUrl": "https://sns.eu-west-1.amazonaws.com/SimpleNotificationService-a86cb10b4e1f29c941702d737128f7b6.pem",
                    "UnsubscribeUrl": "https://sns.eu-west-1.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:eu-west-1:XXXXXXXXXXX:demo-destinations-topic:37ded1fd-a827-49c0-9963-660867649cc4",
                    "MessageAttributes": {}
                }
            }
        ]
    },
    "responseContext": {
        "statusCode": 200,
        "executedVersion": "$LATEST"
    },
    "responsePayload": {
        "statusCode": 200,
        "body": "{\"message\":\"hello\"}"
    }
}
```

The interesting part here is that we can forward events without adding any code to our functions. That's great!! As [Ben Kehoe](https://twitter.com/ben11kehoe) once said: "hours of coding can save you from minutes of configuration".

The "bad" news here is that you can't change the event that is sent to the destination, so if you need to do this, you will have to send the event manually.

## Summary
In this article, we've seen how to set up lambda destinations and how they can be a substitute for DLQs. Hope it helps!

You can find the source code of this demo [here](https://github.com/vgaltes/DemoDestinations).