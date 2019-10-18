---
title: 'Basic API Gateway endpoint authentication with Cognito User Pools'
date: '2019-10-18'
categories:
- serverless
tags:
- authentication
- serverless framework
- cognito
- amplify
---

In many occasion,s you don't want your whole API open to the public. Maybe you want to make some endpoints available to authenticated users. In this article we're going to see how to do that using [Amazon Cognito User Pools](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html) and [AWS Amplify](https://aws.amazon.com/amplify/). Let's start!

## Amazon Cognito User Pools
As the documentation says, a user pool is a user directory in Amazon Cognito. You can allow your users to sign-up, sign-in, etc. You can also implement social sign-in with other identity providers, but we'll see that another day. Today we're going to create a simple user pool to allow users to sign-up and sign-in using their email.

We can create a user pool using the console, but as we like Infrastructure as Code, we're going to use the serverless framework to create it. So, go to your preferred terminal, create a folder called, for example, TestCognitoUserPool, and start a new nodejs project. I'm going to use yarn this time.

```
yarn init
```

Now install the serverless framework as dev dependency

```
yarn add serverless --dev
```

Now, create a file called serverless.yml and copy the following content

```
service: testcognitouserpool

provider:
  name: aws
  runtime: nodejs10.x
  region: ${opt:region, self:custom.defaultRegion}
  stage: ${opt:stage, self:custom.defaultStage}

custom:
  defaultRegion: eu-west-1
  defaultStage: dev${env:SLSUSER, ""}

resources:
  Resources:
    CognitoUserPool:
      Type: AWS::Cognito::UserPool
      Properties:
        UserPoolName: ${self:provider.stage}-testauthsls-user-pool
        UsernameAttributes:
          - email
        AutoVerifiedAttributes:
          - email

    CognitoUserPoolClient:
      Type: AWS::Cognito::UserPoolClient
      Properties:
        ClientName: ${self:provider.stage}-testauthsls--user-pool-client
        UserPoolId:
          Ref: CognitoUserPool
        GenerateSecret: false
```

We're here creating a basic user pool, where the user will sign-in using her email as username. We're also creating an app client, specifying to not create the client secret, as the Javascript SDK doesn't have support for it.

Let's deploy this. Create a script on your package.json file:

```
"scripts": {
    "deploy": "serverless deploy --aws-profile serverless-local"
  }
```

(In my case, I have a profile on my machine called serverless-local. Check the serverless framework [documentation](https://serverless.com/framework/docs/providers/aws/guide/credentials/) to see how you can set it up).

Now, we can finally deploy:

```
yarn run deploy
```

Now, go to the AWS console and take note of the of the user pool id and the app client id.

## Secured endpoint
It's time now to create an api with an unsecured endpoint and a secured one. Return to your terminal and create another folder. Do the same node initializtion we did before and install the serverless-pseudo-prameters plugin, as we'll need it.

```
yarn add serverless-pseudo-parameters --dev
```

Now create the serverless.yml file with the following content:

```
service: testauthsls

plugins:
  - serverless-pseudo-parameters

provider:
  name: aws
  runtime: nodejs10.x
  region: ${opt:region, self:custom.defaultRegion}
  stage: ${opt:stage, self:custom.defaultStage}

custom:
  defaultRegion: eu-west-1
  defaultStage: dev${env:SLSUSER, ""}

functions:
  helloWorld:
    handler: src/functions/helloWorld.handler
    events:
      - http:
          path: api/helloWorld
          method: get
          cors: true
  helloWorldSecured:
    handler: src/functions/helloWorldSecured.handler
    events:
      - http:
          path: api/helloWorldSecured
          method: get
          cors: true
          authorizer:
            name: authorizer
            arn: arn:aws:cognito-idp:#{AWS::Region}:#{AWS::AccountId}:userpool/<theUserPoolId>

## to add cors to the API GW errors
resources:
  Resources:
    GatewayResponseDefault4XX:
      Type: 'AWS::ApiGateway::GatewayResponse'
      Properties:
        ResponseParameters:
          gatewayresponse.header.Access-Control-Allow-Origin: "'*'"
          gatewayresponse.header.Access-Control-Allow-Headers: "'*'"
        ResponseType: DEFAULT_4XX
        RestApiId:
          Ref: 'ApiGatewayRestApi'
    GatewayResponseDefault5XX:
      Type: 'AWS::ApiGateway::GatewayResponse'
      Properties:
        ResponseParameters:
          gatewayresponse.header.Access-Control-Allow-Origin: "'*'"
          gatewayresponse.header.Access-Control-Allow-Headers: "'*'"
        ResponseType: DEFAULT_5XX
        RestApiId:
          Ref: 'ApiGatewayRestApi'

```

Nothing too strange here. First, we're creating a couple endpoints, both with CORS activated. In the secured one, we're saying that the authorizer will be our recently created user pool. In the resources section we're telling API Gateway to add CORS headers when it's failing for any reason, for example for a 401 unauthorized. This way, debugging errors from the client will be easier.

Let's create the files that will hold our functions. Start with `src/functions/helloWorld.js`:

```
module.exports.handler = async event => {
    const res = {
        statusCode: 200,
        headers: {
            "Access-Control-Allow-Origin": "*"
        },
        body: JSON.stringify('Hello from an unsecured endpoint')
    };

    return res;
};
```

And now it's time for `src/functions/helloWorldSecured.js`:

```
module.exports.handler = async event => {
    const userEmail = event.requestContext.authorizer.claims.email;

    const res = {
        statusCode: 200,
        headers: {
            "Access-Control-Allow-Origin": "*",
            'Access-Control-Allow-Credentials': true
        },
        body: JSON.stringify(`Hello from an secured endpoint -> ${userEmail}`)
    };

    return res;
};
```

As you can see, we're taking the user email from the event, as it will come with the user information.

Let's deploy this (assuming you have the same script as before in the package.json file):

```
yarn run deploy
```

take note of the endpoint address, as we'll need it in the next step.

## Client App

It's now time to create a client App. For now, we're going to create a basic web site with HTML and Javascrip. I'm not going to copy here the HTML that I'm using because it's long and awful, but it's just a buch of inputs and buttons.

Let's see how we can deal with users. First of all you will need to configure the Authentication part of AWS Amplify:

```
import Amplify, { Auth } from 'aws-amplify';

Amplify.configure({
    Auth: {
        // REQUIRED - Amazon Cognito Region
        region: 'eu-west-1',

        // OPTIONAL - Amazon Cognito User Pool ID
        userPoolId: 'eu-west-XXXXX',

        // OPTIONAL - Amazon Cognito Web Client ID (26-char alphanumeric string)
        userPoolWebClientId: 'YYYYYYYYYYYYYYYYYYYYYYYYYY',

        // OPTIONAL - Enforce user authentication prior to accessing AWS resources or not
        mandatorySignIn: false
    }
});
```

Let's see what we need to call in order to sign-up a new user:

```
SignUpButton.addEventListener('click', async (evt) => {
  const email = document.getElementById('SignUpEmail').value;
  const password = document.getElementById('SignUpPassword').value;

  await Auth.signUp({
    username: email,
    password
    })
    .then(data => {
      console.log(data);
    })
    .catch(err => console.log(err));
});
```

As you can see, we need to call the signUp method, passing an object with the username and password. If you use other attributes, you'll need to pass them too, but we'll see that in another article.

When the user signs up, she will receive an email with a confirmation number. This is configurable (you can receive an SMS, for example), but the default is an email. If you go to the browsers console, you will see the user object with the field confirmed as false.

So, go to your email, copy the confirmation number and call this function:

```
ConfirmUserButton.addEventListener('click', async (evt) => {
  const username = document.getElementById('ConfirmationCodeUser').value;
  const code = document.getElementById('ConfirmationCodeCode').value;

  await Auth.confirmSignUp(username, code).then(data => console.log(data))
    .catch(err => console.log(err));
});
```

Great, the user is now confirmed, we can sign her in:

```
SignInButton.addEventListener('click', async (evt) => {
  const username = document.getElementById('SignInLogin').value;
  const password = document.getElementById('SignInPassword').value;

  await SignIn(username, password);
});

async function SignIn(username, password) {
  try {
      const user = await Auth.signIn(username, password);
  } catch (err) {
    if (err.code === 'UserNotConfirmedException') {
      } else if (err.code === 'PasswordResetRequiredException') {
      } else if (err.code === 'NotAuthorizedException') {
      } else if (err.code === 'UserNotFoundException') {
      } else {
          console.log(err);
      }
  }
}
```

Easy peasy. Now, you can get the logged user whenever you want:

```
let refreshAuthenticatedUserInfo = async () => {
  await Auth.currentAuthenticatedUser({
    bypassCache: false  // Optional, By default is false. If set to true, this call will send a request to Cognito to get the latest user data
  }).then(user => {
    UserLoggedInSpan.innerText = user.attributes["email"]
  })
  .catch(err => {
    UserLoggedInSpan.innerText = '';
    console.log(err)
  });
}
```

And finally, you can log out if you want:

```
LogOutButton.addEventListener('click', async (evt) => {
  await Auth.signOut()
    .then(data => {
      console.log(data);
      refreshAuthenticatedUserInfo();
    })
    .catch(err => console.log(err));
});
```

But before logging out, we want to make a call to our secured and unsecured endpoints. To do that, we can let Amplify help us again, by using the API module. Let's change the import to:

```
import Amplify, { Auth, API } from 'aws-amplify';
```

And add the following code to the configuration object we pass to the configure function:

```
API: {
    endpoints: [
        {
            name: "MyAPIGatewayAPI",
            endpoint: "https://xxxxxx.execute-api.eu-west-1.amazonaws.com" // your API GW base url
        }
    ]
}
```

Let's make a call to an non-secured endpoint:

```
RequestToNonSecuredEndpointButton.addEventListener('click', async evt => {
  await API.get("MyAPIGatewayAPI", '/dev/api/helloWorld').then(response => {
    console.log(response);
  }).catch(error => {
      console.log(JSON.stringify(error))
  });
});
```

And finally let's make a call to a secured endpoint:

```
RequestToSecuredEndpointButton.addEventListener('click', async evt => {
  let myInit = { 
    headers: { Authorization: `Bearer ${(await Auth.currentSession()).getIdToken().getJwtToken()}` }
}
  await API.get("MyAPIGatewayAPI", '/dev/api/helloWorldSecured', myInit).then(response => {
      console.log(response);
  }).catch(error => {
      console.log(JSON.stringify(error))
  });
});
```

As you can see, we're constructing the authorization header with Amplify's help. In case all your endpoint is secured, you can put that in the endpoint configuration, doing something like:

```
{
    name: "MyAPIGatewayAPI",
    endpoint: "https://XXXXXXX.execute-api.eu-west-1.amazonaws.com",
    custom_header: async () => { 
        return { Authorization: `Bearer ${(await Auth.currentSession()).getIdToken().getJwtToken()}` }
    }
}
```

## Summary
In this article we've seen how we can secure an endpoint using Cognito User Pools. We've also seen how can we access those endpoints using Amplify and how we can use Amplify to creat and sign in users. Hope you find it useful!
