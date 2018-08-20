---
title: 'Sequences in F#'
date: '2016-2-20'
categories:
- FSharp
tags:
- data structures
- F#
---

A sequence is a list of potential values (all of them of the same type) computed on demand.

### Sequence creation

As with [arrays](http://vgaltes.com/fsharp/arrays-in-fsharp/) there are several ways to create a sequence. 

**Create from a range expression**

You can create a new sequence from a range expression. In this case, instead of using `[|` and `|]` you should use `{` and `}`

``` fsharp
    let numbers = {1..20}
```

**Create from a sequence expression**

You can use an expression inside brackets (and after seq keyword) to create a new sequence:

``` fsharp
    let numbers = seq {for i in 1..20 do yield i}
```

We can write a compacted version using the forward arrow followed by the value to yield:

``` fsharp
    let numbers = seq {for i in 1..20 -> i}
```

**Create using a function in the Seq module**

As with arrays, there are some functions in the Seq module to create a sequence.

We can use Seq.init to initialise a sequence of n elements.

``` fsharp
    let numbers = Seq.init 20 (fun i -> i * 2)
```
    
Or we can use initInifinite to create an infinite sequence.

``` fsharp
    let numbers = Seq.initInfinite (fun i -> i * 2)
```
    
Finally, we can also create a sequence directly from an IEnumerable

``` fsharp
    let files =
        System.IO.Directory.EnumerateDirectories("C:\\Windows")
        |> Seq.map(fun f -> f.Length)
```
        
### Operations in Seq module

All the operations we've seen in the last article about arrays are applicable to this one, changing Array by Seq (i.e. from Array.iter to Seq.iter) and the behavior is the same. Let's see some other functions that can be valuable.

**Seq.unfold**

Unfold is a way to generate a sequence from a generator function. You can see it as an extension to Seq.initInfinite. The function takes two parameters: the first one is the generator function, which must return an [option](http://fsharpforfunandprofit.com/posts/the-option-type/) tuple with the next element of the sequence and the next state value to be passed to the next iteration. The second parameter is the initial state.

For example, if we want to generate the square of a number up to a certain threshold we can do something like this

``` fsharp
    >let squareUpTo (top) =
        1
        |> Seq.unfold(fun i -> if ( i*i > top ) then None
                                else Some(i*i, i + 1))
                                
    1 4 9 16 25 36 49 64 81 100 121 144 169 196 225 256 289 324 361 400 441 484 529 576 625 676 729 784 841 900 961 
```

The first 1 is the initial state, in this case 1. This initial state is passed to unfold function. The generator checks if the result will be greater than the threshold and, if it is, returns None. If it isn't, returns a tuple with the next element of the sequence (in this case de square) and the next state to be passed to unfold (in this case the next number).

**Seq.find**

Find takes a boolean as a parameter and returns the first element of the sequence that where the function returns true. If it can't find any result, returns an exception.

Following the previous example if we do

``` fsharp
    >let j = 
        squares
        |> Seq.find(fun i -> i < 0)
```
        
We get the following error
    System.Collections.Generic.KeyNotFoundException: Exception of type 'System.Collections.Generic.KeyNotFoundException' was thrown.
   
On the other hand, if we do

``` fsharp
    let j = 
        squares
        |> Seq.find(fun i -> i > 200 )
```

We get

``` fsharp
    val j : int = 225
```

If we don't want to get an exception, we can use the tryFind function, which returns an option type. Then, the follwing code

``` fsharp
    let j = 
        squares
        |> Seq.tryFind(fun i -> i < 0)
```

Returns

``` fsharp
    val j : int option = None
```

**Seq.pick**

Given a sequence takes the first result of the function provided as a parameter that is not a None (the function must return an option)

``` fsharp
    let j = 
        squares
        |> Seq.pick(fun i -> if (i > 200) then Some(i) else None)
        
    val j : int = 225
```

If there isn't any valid value, it throws an exception. If you want to avoid this, use tryPick.

**Seq.findIndex**

Same as Seq.find but returns the index of the element, not the element itself. If you want to avoid the exception, use tryFindIndex.

``` fsharp
    let j = 
        squares
        |> Seq.findIndex(fun i -> i > 200 )

    val j : int = 14
```


**Seq.exists**

Returns true if the function supplied returns true for any of the values of the sequence.

``` fsharp
    let j =
        squares
        |> Seq.exists(fun i -> i = 100)

    val j : bool = true
```


**Seq.groupBy**

Groups a sequence by the results of the function supplied. Returns sequence of key/value pairs.

``` fsharp
    let howBigIsTheNumber i =
        if i < 100 then "Small"
        elif i < 500 then "Medium"
        else "Big"

    let groupedSquares =
    squares
    |> Seq.groupBy(fun i -> howBigIsTheNumber i)
```


**Seq.disctint**

Given a sequence, return only the unique elements

``` fsharp
    >let random = System.Random()
     let randomNumbers = Seq.init 50 (fun i -> random.Next(1, 5))
     let disctintNumbers =
        randomNumbers
        |> Seq.distinct
        |> Seq.iter (fun i -> printf "%d " i )

    3 2 1 4
```

If we need to get the disctint elements given a function, we can use disctintBy


**Seq.pairwise**

Given a sequence creates a sequence of tuples. The first tuple will contain the first and second element of the original sequence, the second tuple will contain the second and third, and so on.

``` fsharp
    >let numbers = {1..5}
    let tNumbers = 
    numbers
    |> Seq.pairwise
    |> Seq.iter (fun i -> printf "(%d, %d) " (fst i) (snd i) )
    
    (1, 2) (2, 3) (3, 4) (4, 5) 
```
    
**Seq.windowed**

Given a sequence creates a sequence of arrays of the length supplied. The first array will contain the elements between the first element of the original sequence and length. The second one will contain the elements between the second element of the original sequence and length + 1, and so on.

``` fsharp
    >let numbers = {1..10}
     let tNumbers = 
        numbers
        |> Seq.windowed 5
        |> Seq.iter (fun i -> printf "%s\n" ( String.Join(",", i)) )
        
    1,2,3,4,5
    2,3,4,5,6
    3,4,5,6,7
    4,5,6,7,8
    5,6,7,8,9
    6,7,8,9,10
```
    
    
**Seq.collect**

Given a sequences applies a function to each element that creates a sequence and concatenates the results:

``` fsharp
    >let numbers = {1..5}
    numbers
    |> Seq.collect (fun i -> {i*10..i*10+5} )
    |> Seq.iter (fun i -> printf "%d " i )
    
    10 11 12 13 14 15 20 21 22 23 24 25 30 31 32 33 34 35 40 41 42 43 44 45 50 51 52 53 54 55 
```
    
#### Summary

In this article, we've continued taking a look at data structures in F#, in this case sequences. We've seen some other functions that can also be applied to other data structures.