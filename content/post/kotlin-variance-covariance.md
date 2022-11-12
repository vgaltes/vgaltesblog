---
title: 'Learning Kotlin: variance and covariance'
date: '2022-11-12'
categories:
- kotlin
tags:
- generics
- variance
- covariance
comments: []
---

When we work with generics, sometimes we need to help the compiler in order to accomplish what we want. By doing that, we'll be telling the compiler if two classes are covariant or contravariant. Let's see a couple of examples to help understand these concepts.

## Covariance
Imagine that we want to model a factory of toys. Let's start by creating an interface that will take a generic type:

``` kotlin
interface Factory<T> {
    fun create(name: String): T
}
```

Now we need to define what we want to produce

``` kotlin
open class Toy(val name: String)
class Ball(name: String) : Toy(name)
class Transformer(name: String) : Toy(name)
class TableGame(name: String) : Toy(name)
```

So now, we're able to define some of the factories our company want to build. We can have some factories more specialised, like a `TableGameFactory`, and some more generic where more toys can be created. 

``` kotlin
class ToyFactory : Factory<Toy> {
    override fun create(name: String): Toy {
        return when {
            name.startsWith("Ball") -> Ball(name)
            name.startsWith("Transformer") -> Transformer(name)
            name.startsWith("TableGame") -> TableGame(name)
            else -> throw Exception("Not a valid toy")
        }
    }
}

class TableGameFactory : Factory<TableGame> {
    override fun create(name: String): TableGame {
        return TableGame(name)
    }
}
```

And finally, imagine that in our code we have created a variable of type `TableGameFactory`.

``` kotlin
var tableGameFactory = TableGameFactory()
```

What happens if we try to do something like this?

``` kotlin
val toyProducer:Factory<Toy> = tableGameFactory
```

Well, the compiler will say no. The compiler knows that a `TableGame` inherits from Toy, but it doesn't know anything about the relationship between a `Producer` of `Toy` and a `Producer` of a `TableGame`. We need to help it. We need to tell it that those classes are covariant in that type parameter, i.e., the inheritance relationship is kept. So if a `TableGame` inherits from `Toy`, we need to tell the compiler that `Producer<TableGame>` is a subtype of `Producer<Toy>`. And we can do this using the `out` keyword in the interface definition:

``` kotlin
interface Factory<out T> {
    fun create(name: String): T
}
```

The keyword is `out`, because we are declaring producers of that type. So T, can't be in any function parameter, it only can be returned so we can do the assignation we did before.

### Contravariance
Now that we know what covariance is, let's take a look at contravariance. Imagine that we are modelling how people eat. Let's start with the `Person` class that can be something like this:

``` kotlin
class Person<T> {
    fun eat(aThing: T) {}
}
```

What do people eat? Well we eat aliments, and those aliments can be, for example, vegetables and eggs.

``` kotlin
open class Food{}
class Vegetable : Food()
class Egg : Food()
```

A person can be omnivorous, so she can eat any type of Food

``` kotlin
val omnivorous = Person<Food>()
```

But what if that person, during some time wants to be vegan an only eat vegetables? Can we do this?

``` kotlin
val vegetarian: Person<Food> = omnivorous
```

No, we can't because not only `Person<Food>` is not a subtype of `Person<Vegetable>`, but quite the opposite: `Vegetable` is a subtype of `Food`. But does it make sense that we make this assignation? Yes, it does. As a person that can eat anything, I can spend some time eating only vegetables. So, we need to help the compiler. In this case, we will help it using the keyboard in, because we are using the contravariant type as an input parameter.

``` kotlin
class Person<in T> {
    fun eat(aThing: T) {}
}
```

## Summary
In this article, we've seen what variance and covariance is. We have used what is called **declaration-site variance**, because we've defined the variance in the class, not in a method. In another article we'll take a look at **use-syte variance**, **type projections** and **type erasure**
