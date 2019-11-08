---
title: 'Set up a multi-region active-active backend'
date: '2019-11-08'
categories:
- serverless
tags:
- api gateway
- serverless framework
- route 53
---

Congratulations! Your startup is starting to bring attention to many people and you're starting to have clients from different countries and continents. But your lambdas and API gateway are still in your initial region, and that might add some latency to some users. Apart from that, you want to increase the reliability of your system. So, you decide to go multi-region. Can you do that easily? In this article, we'll see how to do that.

**Disclaimer**
This article is a based in [this article](https://read.acloud.guru/building-a-serverless-multi-region-active-active-backend-36f28bed4ecf) from [Adrian Hornsby](https://twitter.com/adhorn) (who is Principal Evangelist at [AWSCloud](https://twitter.com/awscloud)). I usually write to articles to be able to learn and put in practice what I learn, joining information from different souces. In this case, the only source is Adrian's article, and I added some automation using the [Serverless framework](httsp://serverless.com). So, please, if you want to read the full story, go and read Adrian's article (and follow him on twitter!!).


## DynamoDB Global Tables
DynamoDB [Global Tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html) allows us to create multi-region multi-master tables. DynamoDB will replicate our data accross all the regions we specify, and we won't need to worry about anything.

So, first thing is to create a new service with a DynamoDB table with streaming enable. Something like this:
```
service: MultiRegionService

plugins:
  - serverless-pseudo-parameters

custom:
  defaultRegion: eu-west-1

provider: 
  name: aws
  runtime: nodejs10.x
  region: ${opt:region, self:custom.defaultRegion}

resources:
  Resources:
    MyGlobalTableEU:
      Type: AWS::DynamoDB::Table
      Properties:
        KeySchema:
          - AttributeName: id
            KeyType: 'HASH'
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: 'N'
        BillingMode: PAY_PER_REQUEST
        TableName: MyGlobalTable
        StreamSpecification:
          StreamViewType: NEW_AND_OLD_IMAGES
```

And we need to deploy this in two regions, let's say eu-west-1 and us-east-1. So, go to your `project.json` file and create the following scripts:

```
"deploy:europe": "serverless deploy --aws-profile serverless-local --region eu-west-1",
"deploy:us": "serverless deploy --aws-profile serverless-local --region us-east-1",
```

(change serverless-local for the profile you have configured in your laptop, if you have any.)

Now you can run `yarn run deploy:europe` and `yarn run deploy:us` and see your tables created in both regions. The next step is to create the Global Table. Unfortunately, you can't do that via Cloudformation, and we need to call an API. So, go again to your `package.json` file and add the following script:

```
"createGlobalTable": "aws dynamodb create-global-table --global-table-name MyGlobalTable --replication-group RegionName=eu-west-1 RegionName=us-east-1 --region eu-west-1 --profile serverless-local"
```

To check that it's working, add manually in the console (or via api or script) an item (`id` and `name`) to one of the tables, and see how it gets replicated in the other one. You'll see that DynamoDB has added three of fields, one of them being `aws:rep:updateregion`, where we'll see the region where the item has been created.

## API
Let's create the API now. We're going to create and endpoint to put items on DynamoDB and another one for getting them. We'll also need a status endpoint that we'll use later. Let's add the functions to the `serverless.yml` file, which should look like this:

```
service: MultiRegionService

plugins:
  - serverless-iam-roles-per-function
  - serverless-pseudo-parameters

custom:
  defaultRegion: eu-west-1

provider: 
  name: aws
  runtime: nodejs10.x
  region: ${opt:region, self:custom.defaultRegion}

functions:
  GetItem:
    handler: src/functions/getItem.handler
    events:
      - http:
          method: get
          path: item/{itemId}
          cors: true
    environment:
      tableName: MyGlobalTable
    iamRoleStatements:
      - Effect: Allow
        Action: dynamodb:getItem
        Resource: arn:aws:dynamodb:#{AWS::Region}:#{AWS::AccountId}:table/MyGlobalTable

  PutItem:
    handler: src/functions/putItem.handler
    events:
      - http:
          method: post
          path: item
          cors: true
    environment:
      tableName: MyGlobalTable
    iamRoleStatements:
      - Effect: Allow
        Action: dynamodb:putItem
        Resource: arn:aws:dynamodb:#{AWS::Region}:#{AWS::AccountId}:table/MyGlobalTable

  HealthCheck:
    handler: src/functions/health.handler
    events:
      - http:
          method: get
          path: health
          cors: true
    environment:
      status: 200

resources:
  Resources:
    MyGlobalTableEU:
      Type: AWS::DynamoDB::Table
      Properties:
        KeySchema:
          - AttributeName: id
            KeyType: 'HASH'
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: 'N'
        BillingMode: PAY_PER_REQUEST
        TableName: MyGlobalTable
        StreamSpecification:
          StreamViewType: NEW_AND_OLD_IMAGES
```

As you can see, we have three funtions here. The health function just takes a value from an environment variable which you shouldn't do in a real-life scenario.

This is the code of the *PutItem* function:

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

  console.log(params);

  const resp = await dynamodb.put(params).promise();

  const res = {
    statusCode: 200,
    body: JSON.stringify(resp)
  };

  return res;
};
```

This is the code of the *GetItem* function:

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

  console.log(req);

  const resp = await dynamodb.get(req).promise();

  const res = {
    statusCode: 200,
    body: JSON.stringify(resp.Item)
  };

  return res;
};
```

And, finally, this is the code of the health function

```
module.exports.handler = async (event) => {
  const status = process.env.status;

  const res = {
    statusCode: status,
    body: JSON.stringify("all done")
  };

  return res;
};
```

Looks great, isn't it? Deploy this service in the same regions we first deployed the DynamoDb tables, `us-east-1` and `eu-west-1`.

## Regional endpoints, certificates and custom domain names
A regional endpoint is an endpoint that improves the latency when the requests are originated from the same region as the API. We can create them with the [serverless-domain-manager](https://github.com/amplify-education/serverless-domain-manager) plugin that we saw in the [previous article](https://vgaltes.com/post/api-gateway-custom-domain/). Check that article and create a certificate for our endpoints. In this case, as we're using regional endpoints, you must create the same certificate in each region you want to deploy the API. In my case, I used the name `multiregion.authenticatedservices.net`. Validate the certificate and update the `serverless.yml` file to include the custom domain part. The difference with the previous article is that now, we don't want to create the record in Route 53 and that the type of the endpoint will be *regional*.

```
customDomain:
    domainName: multiregion.authenticatedservices.net
    basePath: ''
    stage: ${self:provider.stage}
    createRoute53Record: false
    endpointType: regional
```

Add a couple of scripts in the `package.json` file to create the domains:

```
"create_domain:europe": "serverless create_domain --aws-profile serverless-local --region eu-west-1",
"create_domain:us": "serverless create_domain --aws-profile serverless-local --region us-east-1",
```

And call them both to create the domains. After waiting up to 40 minutes, you can redeploy again the API in both regions. When you do that, you will see an output like this:

```
Serverless Domain Manager Summary
Domain Name
  multiregion.authenticatedservices.net
Distribution Domain Name
  Target Domain: d-87n10uz7oc.execute-api.us-east-1.amazonaws.com
  Hosted Zone Id: Z1UJRXOUMOOFQ8
```

Please, take not of the *target domain* for both regions because we will need them.

You can now test both APIs and make sure they work (they should!).

## Health checks
We will now need to create a health check for each endpoint in Route 53. Route 53 will use these health checks to decide if it can route traffic to that endpoint.

I created a new project in a new folder to do this, because I thought it shouldn't be part of my service. This new `serverless.yml` file looks like this:

```
service: MultiRegionCommonStuff

provider:
  name: aws
  runtime: nodejs10.x

custom:
  domains:
    us-east-1: <useastendpoint>.execute-api.us-east-1.amazonaws.com
    eu-west-1: <euwestendpoint>.execute-api.eu-west-1.amazonaws.com

resources:
  Resources:
    HealthCheck:
      Type: AWS::Route53::HealthCheck
      Properties: 
        HealthCheckConfig: 
          EnableSNI: true
          FailureThreshold: 3
          FullyQualifiedDomainName: ${self:custom.domains.${self:provider.region}}
          Port: 443
          ResourcePath: /dev/health
          Type: HTTPS
        HealthCheckTags:
          - Key: Name
            Value: MultiRegionHeatlthCheck-${self:provider.region}
```

You can create both health checks in the same file, or you can do like this and deploy twice. It's up to you. You need, obviously, to change the url of the endpoints to the ones you got when you deploy. You can chech that the creation has been successful in the Route 53 service on the AWS Console.

## Route policy
Before adding the policy we need to grab the target domain name from the endpoint. If you followed the article, you will have them from a previous step. If you've lost it, go to the *Amazon API Gateway* service on the AWS console and select *Custom Domain Names*. Click on your domain and you should see something like this:

![Custom Domain Name](/images/CustomDomainName.png)

It's now time to create the policy. Go to *Route 53* and select *Traffic policies* and create a new one that should look like this:

![Traffic policy](/images/TrafficPolicy.png)

At the end of the policy creation, or afterwards, you need to create a policy record. Create one with the new traffic policy and with your DNS name (the one that you've chosen when you created the certificate), in my case, `multiregion.authenticatedservices.net`. (Be careful because it has costs associated)

![Policy record](/images/PolicyRecord.png)

And we're done!!

## Testing
How can we test it? Assuming you're in Europe, what we're going to do is to bring down that endpoint and create a new item. When we get that item we'll see that it was created in the `us-east-1` region.

To bring down the endpoint, we need to make Route 53 believe that the endpoint is down. So go to the health Lambda function of the `eu-west-1` region in the AWS console and change the status environment variable to `404`. Wait until the health check returns failure and create a new item, in my case making a post call to `https://multiregion.authenticatedservices.net/item` with a body like 

```
{
  "id": 7,
  "name": "postman7"
}
```

And now, if you go and get that item at `https://multiregion.authenticatedservices.net/item/7` you should get back something like this

```
{
  "aws:rep:deleting":false,
  "aws:rep:updateregion":"us-east-1",
  "aws:rep:updatetime":1573139907.998001,
  "id":7,
  "name":"postman7"
}
```
As you can see, the region is `us-east-1` !!

## Summary
In this article, we've seen how we can create a multi-region active-active backend. This is an adaptation of the original [this article](https://read.acloud.guru/building-a-serverless-multi-region-active-active-backend-36f28bed4ecf) from [Adrian Hornsby](https://twitter.com/adhorn) but trying to automate as many things as we can.

Hope it helps!!
