---
title: 'Choice, pass, fail and succeed states in AWS Step Functions'
thumbnail: /images/step-functions.png
date: '2017-06-29'
categories:
- serverless
tags:
- step functions
- aws
- serverless
comments: []
---
This will be the last article explaining the different states we can use in a step function. We'll see three simple states like [Pass](http://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-pass-state.html), [Fail](http://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-fail-state.html) and [Succeed](http://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-succeed-state.html) and finally, we're going to a see a more complex state like [Choice](http://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-choice-state.html). And obviously, we're going to use the [http://serverless.com](serverless) framework to deploy them.

## Pass state
The pass state is a simple state that just passes its input to its output, without performing any work. Apart from the common fields, you can specify two optional fields:
 - Result: The result to pass to the next state, filtered by the ResultPath field.
 - ResultPath: Specifies where (in the input) put the output specified in the Result field.

## Fail state
The fail state stops the execution of the Step Function and marks it as a failure. It only allows the *Type* and *Comment* fields from the common fields, and you can use a couple of optional fields:
 - Cause: a custom failure string
 - Error: an error name that can be use for [error handling](http://vgaltes.com/serverless/handling-errors-in-step-functions/).

## Succeed state
The succeed state stops the execution of the Step Function successfully. It's a terminal state, so it doesn't have a Next or End field. It's a good target for a choice branch that you just want to stop the execution.

## Choice state
The choice state allows you to declare a decision logic into your state machine. You can specify different branches with different logic to access them. In addition to the common fields it adds a couple of fields:
 - Choices (required): an array of choice rules that determines the next state.
 - Default (optional but recommended): the state to transition if no choice rule is satisfied.

### Choice rules
Each choice rule contains a comparision and a next field. Comparisions can be composed using *And* or *Or* operators. You can check all the operators available [here](http://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-choice-state.html#amazon-states-language-choice-state-rules).

When defining a comparision you must specify two fields:
 - Variable: which value are you going to compare. It will be a [path](http://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-input-output-processing.html) of the input value of the function. 
 - Operator: the field name will be the operator you want to use and the value will be the value you want to compare with.

## Let's put everything together
So, let's put all we've learned so far in a single step function (without any lambda this time):

    stepFunctions:
        stateMachines:
            testChoiceStepFunction:
                definition:
                    StartAt: DoChoice
                    States:
                        DoChoice:
                            Type: Choice
                            Choices: 
                            - Variable: "$.value"
                                NumericGreaterThan: 0
                                Next: PositiveNumber
                            - Variable: "$.value"
                                NumericLessThan: 0
                                Next: NegativeNumber
                            Default: Zero
                        PositiveNumber:
                            Type: Pass
                            Result: {"result": "It's a positive number!"}
                            Next: FinalState
                        NegativeNumber:
                            Type: Pass
                            Result: {"result": "It's a negative number!"}
                            Next: FinalState
                        Zero:
                            Type: Fail
                            Cause: "It's a zero!"
                        FinalState:
                            Type: Succeed

What are we defining here is a step function with two choices (number greater than 0 and number less than zero) and a default state. Every choice has a next state. Both *PositiveNumber* and *NegativeNumber* are *Pass* states with a different result, the *Zero* state is a *Fail* state and the *FinalState* is a *Succeed* state. 

![step function](/images/netcorechoice/step_function.png)

Let's deploy and run the function and see what happens.

    sls invoke stepf --nam
e testChoiceStepFunction --data '{"value": 1}'

![choice positive result](/images/netcorechoice/positive.png)

    sls invoke stepf --nam
e testChoiceStepFunction --data '{"value": -1}'

![choice negative result](/images/netcorechoice/negative.png)

    sls invoke stepf --nam
e testChoiceStepFunction --data '{"value": 0}'

![choice zero result](/images/netcorechoice/zero.png)

## Summary
We've seen how easy is to set up a choice state in State Functions, allowing us to choose different paths depending on the input of the state.