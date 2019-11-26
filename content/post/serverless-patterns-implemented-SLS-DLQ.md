---
title: 'Serverless Patterns implemented: using an SQS queue as a DLQ for a SNS topic'
date: '2019-11-26'
categories:
- serverless
tags:
- patterns
- serverless framework
---

In the last two articles ([here](https://vgaltes.com/post/serverless-patterns-implemented-part1/) and [here](https://vgaltes.com/post/serverless-patterns-implemented-part2/)) we implemented some of the Serverless Patterns described in [this](https://www.jeremydaly.com/serverless-microservice-patterns-for-aws) article from [Jeremy Daly](https://twitter.com/jeremy_daly). In this article, we're going to concentrate in just one pattern, the [Notifier](https://www.jeremydaly.com/serverless-microservice-patterns-for-aws/#notifier). We're going to do this, because of the [recent announcement] from AWS that you can now use an SQS queue as a Dead Letter Queue for an SNS topic.

If you read the article, you will see that this DLQ is complementary to the DLQ you might define in a function that is triggered by an SNS topic, as [Otavio Ferreira](https://twitter.com/otaviofff) explains [here](https://twitter.com/otaviofff/status/1195463848481832960). As we saw the lambda function DLQ in a previous article, we're going to focus on the DLQ of the topic. In this scenario, messages will be sent to the DLQ when SNS is not able to deliver the message to the subscribed endpoint. As [this article](https://docs.aws.amazon.com/sns/latest/dg/sns-dead-letter-queues.html) explain, this can happen because the endpoint is not available, which is what we're going to simulate. (client error) or because system responsible for the subscribed endpoint becomes unavailable or returns an exception that indicates that it can't process a valid request from Amazon SNS (server error). In both cases, the message will be sent to the SNS topic DLQ. We can't simulate a failure on a AWS Service, so we're going to simulate that the endpoint of the subscription is not available.

In this article we're going to see how we can implement this using the [Serverless](https://serverless.com) framework. As it's a very new feature of the SNS topics, is not yet implemented, so we're going to need to write quite a bit of CloudFormation. I'm sure the folks from Serverless are already working on making this easier to use.

## Setup
As usual, let's do the basic steps to setup our project. Let's start initializing the nodejs project.
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

(Assuming that you have a profile called serverless-local, of course). 

We will need the `serverless-pseudo-parameters` plugin as well, so let's install it:
```
yarn add serverless-pseudo-parameters --dev
```

Now, create a file called `serverless.yml` and include the usual boilerplate code:

```
service: Notifier

plugins:
  - serverless-pseudo-parameters

provider:
  name: aws
  runtime: nodejs10.x
  region: ${opt:region, self:custom.defaultRegion}
  stage: ${opt:stage, self:custom.defaultStage}

custom:
  defaultStage: dev
  defaultRegion: eu-west-1
```

# SNS Topic
Let's start defining the SNS topic in the resource section of the file. It should look like this:
```
SNSNotifier:
  Type: AWS::SNS::Topic
  Properties:
    DisplayName: ${self:service}-${self:provider.stage}-SNSNotifier
    TopicName: ${self:service}-${self:provider.stage}-SNSNotifier
```

Nothing really strange here. 

Now, it's time to declare the SQS queue we're going to use as DLQ for our topic

```
NotifierDLQ:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: ${self:service}-${self:provider.stage}-NotifierDLQ
```

The next step would be to create the subscription in the SNS topic. The subscription is where we define where do we want the messages to be delivered.

```
NotifierSubscription:
  Type: AWS::SNS::Subscription
  Properties: 
    Endpoint: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${self:provider.stage}-ReadFromSNS-badOne
    Protocol: lambda
    TopicArn: !Ref SNSNotifier
    RedrivePolicy:
      deadLetterTargetArn: !GetAtt NotifierDLQ.Arn
```

In this case we're going to define that we'll have a lambda that will be responsible to receive these messages. There are two interesting parts here. The first one is that in the *Endpoint* section we're defining a lambda that it really doesn't exist. The second one is the RedrivePolice section, where we define the Dead Letter Queue of the topic, in this case the SQS queue previously defined.

For this topic to be able to put messages to a lambda (although in the subscription we're simulating that the lambda doesn't exist for some reason) we need to define a lambda permission:

```
LambdaInvokePermissionFromSNS:
  Type: AWS::Lambda::Permission
  Properties:
    Action: lambda:InvokeFunction
    Principal: sns.amazonaws.com
    SourceArn: !Ref SNSNotifier
    FunctionName: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${self:provider.stage}-ReadFromSNS
  DependsOn: ReadFromSNSLambdaFunction
```

The lambda resource needs to be already created, that´s why we put the DependsOn clause. You can´t reference a resource created by the serverless framework directly in a CloudFormation resource, but you can do the trick to use the name that the framework will use for the lambda. In this case, is the name of the function with `LambdaFunction` concatenated.

(for the purpose of this particular example you can skip this step, but in case you want to see how the messages are delivered to a lambda with no issues, you will need to add it).

Finally, for the SNS topic to be able to deliver messages to the SQS DLQ we need to define an SQS Queue Policy, where we will tell SQS to allow messages comming from the SNS Topic:

```
NotifierDLQPolicy:
  Type: AWS::SQS::QueuePolicy
  Properties:
    PolicyDocument:
      {
        "Version": "2012-10-17",
        "Id": "NotifierDLQPolicy",
        "Statement": [{
          "Sid":"NotifierDLQPolicy001",
          "Effect":"Allow",
          "Principal":"*",
          "Action":"sqs:SendMessage",
          "Resource":"arn:aws:sqs:#{AWS::Region}:#{AWS::AccountId}:${self:service}-${self:provider.stage}-NotifierDLQ",
          "Condition":{
            "ArnEquals":{
              "aws:SourceArn":"arn:aws:sns:#{AWS::Region}:#{AWS::AccountId}:${self:service}-${self:provider.stage}-SNSNotifier"
            }
          }
        }]
      }
  
    Queues:
      - !Ref NotifierDLQ
```

Finally, and outside the resources section, we can define the functions that will read messages from the SNS Topic and the SQS Queue:

```
functions:
  ReadFromSNS:
    handler: src/functions/readFromSNS.handler
    events:
      - sns:
          arn: !Ref SNSNotifier
          topicName: SNSNotifier

  ReadFromDLQ:
    handler: src/functions/readFromDLQ.handler
    events:
      - sqs:
          batchSize: 1
          arn: !GetAtt NotifierDLQ.Arn
```

And the code of both functions. This is the code of the readFromSNS function:
```
module.exports.handler = (event, context) => {
  const message = event.Records[0].Sns.Message;

  console.log(`Message received via SNS. ${message}`) ;

  return "all done";
};
```

And this is the code of the readFromDLQ function:
```
module.exports.handler = (event, context) => {
  const message = JSON.stringify(event.Records[0]);

  console.log(`Message received in DLQ. ${message}`) ;

  return "all done";
};
```

If we deploy, the serverless framework would have been added the subscription for the *ReadFromSNS* lambda in the SNS topic. You can delete it manually if you want to make sure that the message never gets there.

And that's it. If you now try to put a message in the topic (using the console, or the CLI, or the SDK) you will see that the message ends up in the DLQ.

You can check the code [here](https://github.com/vgaltes/ServerlessPatterns/tree/master/Notifier).

## Summary
In this article, we've seen how we can define a SNS Queue as a Dead Letter Queue for an SNS topic. I expect this to not be needed in a near future because the Serverless framework will take care of it, but it's never bad to know the internals and, in the meantime it gets implemented, you will be able to take care of this situations with the code we've seen in this article.

Hope it helps!!
