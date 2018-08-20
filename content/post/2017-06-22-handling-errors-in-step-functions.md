---
published: true
title: 'Handling errors in AWS Step Functions'
date: '2017-06-22'
thumbnail: /images/step-functions.png
categories:
- serverless
tags:
- step functions
- aws
- serverless
comments: []
---
As we've seen in previous articles, Step Functions helps us to orchestrate lambda functions. One of the most important aspects when we're developing a system, distributed or not, is handling errors and retries. In this articles we'll see how easy is to do it using Step Functions and the serverless framework.

## Catching errors
### Coding the lambda
First of all we're going to catch some errors. Let's create a new project with one lambda inside it named ErrorLambda with the following code:

    public class ErrorLambda
    {
        public void Error(string input){
            throw new CustomException(input);
        }
    }

    public class CustomException : Exception{
        public CustomException(string message) : base(message){}
    }

### Creating the Step Function
Let's create a Step Function that catches errors. As always, create the serverless.yml and copy the following definition of the Step Function:

    stepFunctions:
        stateMachines:
            testErrorStepFunction:
                definition:
                    StartAt: Error
                    States:
                        Error:
                            Type: Task
                            Resource: arn:aws:lambda:${opt:region}:${self:custom.accountId}:function:${self:custom.errorService}-${opt:stage}
                            Catch:
                                - ErrorEquals: ["CustomException"]
                                    Next: CustomErrorFallback
                                - ErrorEquals: ["States.TaskFailed"]
                                    Next: "ReservedTypeFallback"
                                - ErrorEquals: ["States.ALL"]
                                    Next: "CatchAllFallback"
                            End: true
                        CustomErrorFallback:
                            Type: Pass
                            Result: "This is a fallback from a custom Lambda function exception"
                            End: true
                        ReservedTypeFallback:
                            Type: Pass
                            Result: "This is a fallback from a reserved error code"
                            End: true
                        CatchAllFallback:
                            Type: "Pass"
                            Result: "This is a fallback from any error code"
                            End: true

We can split what we're doing here in two parts. At the end of the definition we're defining the steps that we'll run after we catch an error. In this case, we'll define three types of errors and three different next states.

First, we defined our catchers. In this case we defined three different catchers. The first one is to catch exceptions that we throw from our Lambda. The string that help us to filter the error is the name of the class of the exception we want to catch. In the second and third catchers, we're using predefined error codes. We know that they are predefined error codes because they start with *States.* The possible values for a predifined error codes are *States.ALL*, *States.Timeout*, *States.TaskFailed*, *States.Permissions*. You can read more about them [here](http://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-errors.html#amazon-states-language-error-names)

If we now deploy the step function, run it, and we go to the AWS console we'll see the representation of it

![retries](/images/netcoreretry/step-function.png)

As you can see, is similar to a [parallel step](http://vgaltes.com/serverless/step-functions-parallel-state/), but in this case we just run one of the branches.

If we click in one of the catchers and inspect the input, we'll see the type of error and the stack trace:

    {
        "Error":"CustomException",
        "Cause":{
            "errorType": "CustomException",
            "errorMessage": "an error",
            "stackTrace": [
                "at StepFunctionsHandleErrors.ErrorLambda.Error(String input) in /StepFunctionsHandleErrors/ErrorLambda/ErrorLambda.cs:line 12",
                    "at lambda_method(Closure , Stream , Stream , ContextInfo )"
                    ]
        }
    }

The string in the errorType field is the one that you have to use in the definition of the catcher.

### Retrying
It's possible that when you detect an error (or an specific type of error) you want to retry a lambda to see if the error was a transient one. Configuring retries is a very simple step in Step Functions, you just need to add the following code just before the *Catch* element:

    Retry:
        - ErrorEquals: [ "CustomException" ]
          IntervalSeconds: 1
          BackoffRate: 2.0
          MaxAttempts: 4
        - ErrorEquals: [ "States.ALL" ]
          IntervalSeconds: 5

In this case we're specifying that, if we get a *CustomException* error we're going to retry 4 times at most (default 3), with an initial interval of 1 second (default 1) and a back-off rate of 2.0 (default 2.0).

If we want, we can specify a list of errors in the *ErrorEquals* field. If we want to specify a *States.ALL* retrier, it must appear alone and as the last retrier.

## Summary
We've seen how easy is to deal with errors and retries using Step Functions and, as always, how the [serverless](http://serverless.com) framework help us in deploying them.