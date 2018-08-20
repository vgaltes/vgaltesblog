---
title: 'Function composition and pipeline operator'
date: '2016-4-2'
categories:
- FSharp
tags:
- F#
---

As [Scott Wlaschin](https://twitter.com/ScottWlaschin) points out in his excellent [article](https://fsharpforfunandprofit.com/posts/function-composition/) function composition it's  not the same as using the pipeline operator.

The definition of the pipeline operator is this one:

    let (|>) x f = f x

So, take the thing on the left hand side of the operator and use it as the last parameter on the function in the right hand side.

On the other hand, we have this definition for the forward composition operator ([>>](https://msdn.microsoft.com/en-us/library/dd233228.aspx))

    let (>>) f g x = g ( f(x) )

That means, take the parameter and pass it to the function on the left hand side of the operator, and then apply the function on the right hand side of the operator to the result.

For example, if we have this couple of functions:

    let add1 a = a + 1
    let mult2 a = a * 2
    
We can define the composite function as 

    let addAndSum = add1 >> mult2
    
And therefore, when we do

    let res = addAnSum 3
    > val res : int = 8
    
As you can see, we are applying the first function (add1) to 3 and then apply the second function to the result ( ( 3 + 1 ) * 2 = 8 )

Preparing my talk at [FSharp eXchange](https://skillsmatter.com/conferences/7145-f-exchange-2016), Visual Studio Powertools warn me about replacing a pipeline by a function composition. The code (simplified version) is this one.

    let tuples = [|("a", [|1|]);("b", [|1;2|]);("c", [|1;2;3|]);("d", [|1;2;3;4|])|]
    tuples
    |> Array.sortByDescending(fun m -> snd m |> Array.length)

Let's decompose this code to see what is it doing. The previous code is equivalent to the following one:

    tuples
    |> Array.sortByDescending(fun m -> Array.length(snd m))

Looking at the part in the right hand side of the arrow we can see that this is equivalent to the definition of composition we've seen previously, being Array.length equivalent to g and snd equivalent to f. That means that we can change the previously code by this one:

    tuples
    |> Array.sortByDescending(snd >> Array.length)
    
As you can see, the final code is much more elegant and succint, and easier to read when you understand what is it doing.