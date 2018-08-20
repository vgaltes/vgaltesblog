---
title: 'Decorator pattern in F#'
date: '2016-6-1'
categories:
- FSharp
tags:
- F#
- Design patterns
---

## Introduction ##
A couple of weeks ago my friend [Ian Russell](http://twitter.com/ijrussell) explained the decorator pattern at work. For those who don't know exactly how it works, the decorator pattern allows us to add behavior to an individual object without affecting the behavior of other objects of that class ([Wikipedia](https://en.wikipedia.org/wiki/Decorator_pattern)).

## The object oriented solution ##
There are a couple of typical implementations of the pattern in C#. The "lightweight" version is "just" to create decorators implementing the same interface of the decorated class, and passing the decorated object in the constructor of the decorator. So we need, at least, to create a new interface (maybe not needed anywhere else) and implement a different class for each decorator. Another option is to create an abstract class defining the decorator (that implements the same interface than the decorated object) and subclass it to create the different decorators. As you can imagine, a lot of boilerplate there.

## The functional way ##
Let's see how we can implement that using a functional language as F#.

Let's start defining the object that we're going to decorate:

    let consoleWriter message =
        printfn "Console writer -> %s" message

Is a simple function that prints a message. Let's decorate that function with a function that logs the message:

    let loggingWriter logger writer message =
        logger message
        writer message
        
The function is like a mix between the class receiving a logger and a writter as a constructor parameters and the function to write the message itself in the object oriented implementation.

Now we need to write the logger. As usual, it will be a function: 

    let consoleLogger message =
        printfn "Console logger -> %s" message

And now, let's create a decorator to add a simple validation:

    let emptyStringValidator writer message = 
        if System.String.IsNullOrWhiteSpace message then
            printfn "Validation error"
        else
            writer message 

One more time, instead of having a class that receives the writer as a constructor paramenter and have a function to write the message, we just have a function that receives the function and the message.

Now it's time to glue all this functions together. We are going to take advantage of partial application to define a function that uses all the previously defined functions:

    let decoratedWriter = emptyStringValidator (loggingWriter consoleLogger consoleWriter)
    
If we take a look at the function signature we can see that it is:

    string -> unit
    
So, it's a function that takes a string (in this case the message) and returns unit (because we are using printfn). Using partial application we have composed a couple of functions: the first one, the one between parenthesis, is a writer (in this case a logginWriter) that takes the consoleLogger and the consoleWriter as a parmeters, and have a free parameter (the message). So, its a function that gets a string and returns unit, that is the function type that the emptyStringValidator is expecting as its first parameter. So, we can pass this function to the emptyStringValidator and we'll get the function that we want, that takes a string as a parameter and return unit.

If we call this function with a string, we get the following result:

    decoratedWriter "Hello world!"
    > Console logger -> Hello world!
      Console writer -> Hello world!
      
And if we call it using an empty string we get the follogins result:

    decoratedWriter ""
    > Validation error
    
## Summary ##

We have seen how much more elegant our code results when using a functional language. In the object oriented code, we have a lot of code that is not really adding any value (interfaces, abstract classes, constructors...). In the functional code, we just have functions and we use a well know feature of the language as it is partial application to compose the function that we really want. You can find the code on [Github](https://github.com/vgaltes/DecoratorPattern)