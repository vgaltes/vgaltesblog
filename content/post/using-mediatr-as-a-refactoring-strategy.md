---
title: 'Using MediatR as a refactoring strategy'
date: '2020-02-13'
thumbnail: 
categories:
- aspnet core
- agile
tags:
- refactoring
---

Let me introduce you to Rachel, the new developer of the team. Rachel is an excellent developer eager to make a huge impact on the team and the organisation. When Rachel lands in the team, they treat them very well. They show her all the facilities, they teach her on how to make a good coffee with the new and shiny coffee machine and they even deploy to production in her first day. Rachel is impressed! I'm going to have a great time here, she thinks.

But the next day, Rachel starts pairing with Josh in a new user story (or feature, or piece of work, or whatever you call it). Josh starts showing here where the controllers are, and how those controllers have some dependencies on various application services. Then, Josh tells Rachel that the application services are a bit bloated, so the easier way to go to the method they need to change is to go to the action in the controller and see what it's calling. When they Ctrl + click on that method, they land in a HUGE application service class with lots of methods, thousands of lines and tens of dependencies. Rachel is horrified and she thinks that maybe it wasn't a good idea to leave her old and cozy job.

This is not a strange situation. Hopefully you're working on a software project that brings a lot of money to your company. Usually, such software are the evolution of years of development and tens of developers, some of them just following the "this is how the work is done here" rule. So it's not rare to stumble upon such big classes.

Those classes are a problem for you and your team. We spend around 60% - 70% of our time reading code, so we need that time to be as much pleasant as we can. Having a big class, with lots of methods, blurred responsibilities and a lot of dependencies doesn't make your time reading that code very pleasant or useful. The cognitive load of processing that class is huge, and sometimes unbereable.

## What can we do about it?

The first word that appears in our mind is *refactoring*. We should study that class, see the responsibilities, start breaking it up and end with a nice and clean code. 5 months later. Without having delivered a single feature. With the future of the company at stakes.

Can we do something less agressive?

There's a library very used in the .Net ecosystem called [MediatR](https://github.com/jbogard/MediatR). As you can see in the website, MediatR helps us *decoupling the in-process sending of messages from handling of messages*. This allow us to only put the call to mediator in the controller's actions and then code the handlers of those messages. MediatR will do the "dirty job" for you. It has other advantatges, like the ease of implementing cross-cutting concerns as [behaviors](https://github.com/jbogard/MediatR/wiki/Behaviors).

What does it mean for our refactoring? It means that we can start breaking up the application services in smaller handlers that implement each of the methods of it (Ideally splitted in commands and queries). Then, each handler will only have the code of one method of the application service, and will only have the dependencies it needs. Another advantage is that this is a refactor that you can implement method by method, hence it's easy to do applying the [boy scout rule](https://deviq.com/boy-scout-rule/). And finally, if you implment this, you will end up with bunch of folders in your solution which describe the actions and queries that you can make in your system, being that much easier to find what you need and to understand what is the system doing.

Let's see how we can do this refactoring. As you will see, we can do this quite easily and in a "safe" manner (in case you don't have tests yet) by "just" copy and pasting code. Let's see a wee example.

Imagine that you have this code

```
public class UsersApplicationService
{
    public UsersApplicationService(IDependency1 d1, IDependency2 d2, 
        IDependency3 d3, IDependency4 d4, IDependency5 d5)
    {
        // field initialization
    }

    // Lots of methods

    public LoginResult LoginUser(string userName, string password)
    {
        // work with d1

        // work with d2

        // return result
    }

    // More methods
}
```

Let's start by creating a folder called `Commands\LoginUser`. Let's then create there a file called LoginUserResponse, that would be something like this:

```
public class LoginUserResponse
{
    public LoginUserResponse(LoginResult result)
    {
        Result = result;
    }

    public LoginResult Result {get;}
}
```

Then, let's create a class that will define the request. In our case we need to wrap *username* and *password* in a class. We need to indicate to MediatR that this will be a request, implementing the `IRequest<>` interface:

```
public class LoginUserRequest : IRequest<LoginUserResponse>
{
    public LoginUserRequest(string username, string password)
    {
        Username = username;
        Password = password;
    }

    public string Username { get; }
    public string Password { get; }
}
```

And finally, we need to create the handler. This will be a class that implements the `IRequestHandler<,>` interface and that will have a method that receives a `LoginUserRequest` and returns a `LoginUserResponse`. This class will only need two dependencies to be injected and we'll just need to copy the code that we had in the application service method and just change the points where we use the username and password (or create a variable for it) and the object that we return (that will include the previous return object).

```
public class LoginUserHandler : IRequestHandler<LoginUserRequest, LoginUserResponse>
{
    public LoginUserHandler(IDependency1 d1, IDependency2 d2)
    {
        // field initialization
    }

    public Task<LoginUserResponse> Handle(LoginUserRequest request, CancellationToken cancellationToken)
    {
        var username = request.UserName; // we can inline it afterwards
        var password = request.Password; // we can inline it afterwards

        // work with d1 

        // work with d2

        // return result => Task.FromResult(new LoginUserResponse(result));
    }
}
```

Finally, we'll need to change the controller to use mediator instead of calling the application service directly.

If we want to add unit tests afterward, those will be easier to setup, as we only will have two dependencies. They will live in the same folder structure as the source code, so they will be easy to find.

And if we want to add validation, we can do that easily using behaviors. If the validation already exist in the method, we can move it to a validator quite easily.

## Summary
This has been a wee (but real!) example of a refactor to a more understandable organization of the code. If you keep doing this in your codebase, you will end up having a codebase easier to understand and hence, easier to change. Hope you find it useful!