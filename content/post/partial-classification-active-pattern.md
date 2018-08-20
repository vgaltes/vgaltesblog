---
title: 'Partial classification active pattern'
date: '2016-10-17'
categories:
- FSharp
tags:
- F#
- Pattern matching
- Active patterns
---

[Pattern matching](https://docs.microsoft.com/en-us/dotnet/articles/fsharp/language-reference/pattern-matching) is a powerful and amazing characteristic of F#. Actually, is so amazing that Microsoft is starting to [port](https://blogs.msdn.microsoft.com/dotnet/2016/08/24/whats-new-in-csharp-7-0/) it to C#.

There are different kinds of pattern matching. In this post we're going to take a look at a partial classification active pattern that takes an argument and returns a value.

As its name denotes, is a pattern that partially classificates what you match with it. That means that doesn't try to define all possible options but just one. This is quite convenient when using an active pattern to define a business rule of your domain, because you can split the different cases in different partial active patterns making your code much more readable.

You will identify a partial active pattern because the definition looks something like this:

```
let (|WithRemainder|_|) divisor divident =
    // active pattern code
```

You can see between brackets the name of the classification followed by the wilcard name (_), the two of them between |. Just after that you can see the two arguments that the function will take.

This kind of active pattern has to return an Option value. In our case, we are going to return the remainder, if any. So our active pattern will look like:

```
let (|WithRemainder|_|) divisor dividend =
    let remainder = dividend % divisor
    if remainder <> 0 then Some remainder else None
```

How we use it? Well, the first time you see this active pattern applied can seem a bit difficult to understand (at least it was for me! :P), but is not that difficult:

```
match 10 with
| WithRemainder 3 r -> printfn "The remainder is %d" r
| _ -> printfn "No remainder"
```

The difficult part here is to see that the value in the match sentence is passed as the last argument of the active pattern. For example, in our case 3 is the divisor (first argument), 10 is the dividend (second argument) and r is what the pattern returns inside the Option type, in this case the remainder (is not the Option type, is the value itself). Once you understand that, there will be no secrets for you to create and use this kind of active pattern! 