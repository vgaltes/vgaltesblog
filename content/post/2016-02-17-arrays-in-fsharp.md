---
title: 'Arrays in F#'
date: '2016-2-17'
categories:
- FSharp
tags:
- data structures
- F#
---
Arrays are one of the basic data structures in [F#](http://fsharp.org/). In this article we're going to see an introduction of what can we do with them.

Creation
--------

There are several ways to create an array in F#

### Create from a literal

We can create an array with a predefined set of values. To do that, we just need to specify the values separated by semicolons and wrapped between `[|` and `|]`

``` fsharp
    >let numbers = [|1;2;3;4|]
    
    val numbers : int [] = [|1; 2; 3; 4|]
```

### Create a range

We can create an array of predifined values using the range notation:

``` fsharp
    >let numbers = [|100..120|]

    val numbers : int [] =
        [|100; 101; 102; 103; 104; 105; 106; 107; 108; 109; 110; 111; 112; 113; 114; 115; 116; 117; 118; 119; 120|]
```

In the previous code, we are creating an array of numbers between 100 and 120. 

Wan can specify the gap between those numbers:

``` fsharp
    >let numbers = [|100..3..120|]

    val numbers : int [] = [|100; 103; 106; 109; 112; 115; 118|]
```

And we can use an expression inside the brackets too:

``` fsharp
    >let numbers = [|for i in 100..120 do
                    yield i * 2|]
                    
    val numbers : int [] =
    [|200; 202; 204; 206; 208; 210; 212; 214; 216; 218; 220; 222; 224; 226; 228; 230; 232; 234; 236; 238; 240|]
```
  
### Create from a function in the Array module

We can create an array using the Array.create function. This function takes two parameters, the number of positions you want to create and the value you want to use.

``` fsharp
    >let numbers = Array.create 4 5

    val numbers : int [] = [|5; 5; 5; 5|]
```

Another function we can use is init, which is very similar to create but instead of taking the value it takes a function to create the different values

``` fsharp
    >let numbers = Array.init 4 (fun i -> i * 2 )

    val numbers : int [] = [|0; 2; 4; 6|]
```

We can also use zeroCreate to create an array filled with zeros

``` fsharp
    >let numbers : int[] = Array.zeroCreate 4

    val numbers : int [] = [|0; 0; 0; 0|]
```

Finally we can create an array from other array or IEnumerable

``` fsharp
    >let files =
        System.IO.Directory.EnumerateDirectories("C:\\Windows")
        |> Array.ofSeq
```

### Accessing elements in an Array

It's easy to access an element in an Array. Just use the following notation:

``` fsharp
    >let number = numbers.[0]
```

You must take into account that arrays are 0 based.

### Operations in Array module

There are more than 70 functions in the Array module. Let's see some of the most used.

**Array.map**

Takes an array and returns another array of the same length with the result of applying a function to each element.

``` fsharp
    >let numbers = [|1..5|]
    >let squares = 
        numbers 
        |> Array.map (fun i -> i * i)
    
    val squares : int [] = [|1; 4; 9; 16; 25|]
```

**Array.mapi**
 
Is very similar to Array.map but it provides the index of each element.

``` fsharp 
    >let letters = [|'a';'b';'c';'d'|]
    >letters |> Array.mapi (fun i l -> sprintf "The letter at index %i is %c" i l)
    val letters : char [] = [|'a'; 'b'; 'c'; 'd'|]
    val it : string [] =
    [|"The letter at index 0 is a"; "The letter at index 1 is b";
        "The letter at index 2 is c"; "The letter at index 3 is d"|]
```
    
**Array.iter**

Iterates and call a function with each element, but it doesn't returns anything (only has side effects). We can user Array.iteri if we need the index.

``` fsharp
    >let letters = [|'a';'b';'c';'d'|]
    >letters |> Array.iteri (fun i l -> printf "The letter at index %i is %c" i l)
```

**Array.filter**

Given an array only returns those elements on which the function applied returns true.

``` fsharp
    >let numbers = [|1..20|]
    >let evenNumbers = 
      numbers
      |> Array.filter (fun n -> n % 2 = 0)
  
    val evenNumbers : int [] = [|2; 4; 6; 8; 10; 12; 14; 16; 18; 20|]
```

**Array.choose**

Given an array only returns those elements on wich the function applied returns a 'Some' result. So, the function applied must return an option type.

``` fsharp
    >let numbers = [|1..20|]
    >let evenNumbers = 
      numbers
      |> Array.choose (fun n -> if ( n % 2 = 0 ) then Some(n) else None)
  
    val evenNumbers : int [] = [|2; 4; 6; 8; 10; 12; 14; 16; 18; 20|]
```

**Array.sum**

Sum the values of the array. The type of the array must support addition and must have a zero member.

``` fsharp
    >let numbers = [|1..20|]
    >let sum = 
        numbers
        |> Array.sum
  
    val sum : int = 210
```
 
**Array.sumBy**
 
Same as sum but takes a function that select the element to sum.
 
Let's start the example defining a function to get random strings

``` fsharp 
    >let random = System.Random()
    >let randomStr len = 
        let chars = "ABCDEFGHIJKLMNOPQRSTUVWUXYZ0123456789"
        let charsLen = chars.Length

        let randomChars = [|for i in 0..len -> chars.[random.Next(charsLen)]|]
        new System.String(randomChars)`
    
    val randomStr : len:int -> System.String  
```
    
Now, create some random strings

``` fsharp    
    >let strings = 
        [|10..15|]
        |> Array.map randomStr
        
    val strings : System.String [] =
        [|"ZEQNA1HUXS3"; "1C8K1Z5UO58A"; "FT9O8MDAVGFO4"; "G85O8P1NSLE6HX";
            "63XOR0DL4ANJKUS"; "JV6VQW09FPRHUUH4"|]
```
            
And finally, sum the length of those strings

``` fsharp
    >let sum = 
        strings
        |> Array.sumBy (fun s -> s.Length)
        
    val sum : int = 81
```
    
**Array.sort**

Given an array, returns the array sorted by the element. If we use sortBy, we can specify a function to be used to sort

``` fsharp
    >let sortedStrings =
        strings
        |> Array.sort
        
    val sortedStrings : System.String [] =
    [|"1C8K1Z5UO58A"; "63XOR0DL4ANJKUS"; "FT9O8MDAVGFO4"; "G85O8P1NSLE6HX";
        "JV6VQW09FPRHUUH4"; "ZEQNA1HUXS3"|]
```
        
**Array.reduce**

Given an array, uses the supplied function to calculate a value that is used as accumulator for the next calculation. Throws an exception in an empty input list.

``` fsharp
    >let strings = [|"This"; "is"; "a"; "sentence"|]
     let sentence =
     strings
        |> Array.reduce (fun acc s -> acc + " " + s)
        
   val sentence : string = "This is a sentence"
```   

**Array.fold**

Same as reduce, but takes as a parameter the first value of the accumulator.

``` fsharp
    >let strings = [|"This"; "is"; "a"; "sentence"|]
     let sentence =
        strings
        |> Array.fold  (fun acc s -> acc + " " + s) "Fold:"
```       
        
**Array.scan**        
        
Like fold, but returns each intermediate result

``` fsharp
    >let strings = [|"This"; "is"; "a"; "sentence"|]
     let sentence =
        strings
        |> Array.scan  (fun acc s -> acc + " " + s) "Scan:"
        
    val sentence : string [] =
        [|"Scan:"; "Scan: This"; "Scan: This is"; "Scan: This is a";
            "Scan: This is a sentence"|]
```            
            
**Array.zip**
 
Takes two arrays of the same size and produce another array of the same size with tuples of elements from each input array.

``` fsharp 
   >let colorNames = [|"red";"green";"blue"|]
    let colorCodes = [|"FF0000"; "00FF00"; "0000FF"|]
    let colors =
       Array.zip colorNames colorCodes
        
   val colors : (string * string) [] =
       [|("red", "FF0000"); ("green", "00FF00"); ("blue", "0000FF")|]
```
        
There's a very similar function called zip3, wich take three array as inputs, and another call unzip (and unzip3) with takes an array of tuples and decomposes it in two arrays of single values.
 
### Summary
 
We've seen the basics of the Array module. We've seen how to create arrays and some of the most used functions in the Array module.