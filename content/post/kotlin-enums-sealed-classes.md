---
title: 'Learning Kotlin: sealed classes and enums'
date: '2022-11-06'
categories:
- kotlin
tags:
- sealed class
- enum
comments: []
---

In Kotlin we have two constructs that alow us to create a closed typed hierarchy: sealed classes and enums. Let's take a look to them and see the differences amongst them.

## Sealed classes
A sealed class is a class that we know all its subclasses at compile time. We can't declare a subclass of a sealed class outside the module where it's defined. 

One of the differences between a sealed class and an enum is that a subclass of a sealed class can have multiple instances, each with its own state.

A subclass of a sealed class can have its own methods and properties.

Let's take a look to an example:


```
sealed class Error
class FileError(val fileName: String, val severity: Int): Error() {
    fun changeDrive(): String = "Changing drive"
}
class NetworkError(val address: String, val severity: Int): Error() {
    fun retry(): String = "Retrying"
}
class EsotericError(val reason: String, val severity: Int): Error() {
    fun pray(): String = "Praying"
}
```

Here we have a sealed class and three subclasses, each of them with their own method. How can we use them?

```
fun process(error: Error) {
    val action = when (error) {
        is FileError -> {
            error.changeDrive()
        }
        is NetworkError -> {
            error.retry()
        }
        is EsotericError -> {
            error.pray()
        }
    }

    println(action)
}
```

One interesting benefit of sealed classes is that when you use them in a when expression, if the expression is an expression and not an statement (so, you are using the result), you don't need to use the else clause in the expression if it covers all the cases (like in the previous example).

We can use sealed classes when we are dealing with a set of alternative classes. One important characteristic is that they can hold instance-specific data.

## Enums
It's a way to represent a constant set of possible options. 

```
enum class DangerLevel {
    HIGH, MEDIUM, LOW
}
```

We can use enums easily in a when expression

```
fun process(d: DangerLevel){
    when(d) {
        DangerLevel.HIGH -> { /* Do something */}
        DangerLevel.MEDIUM -> { /* Do something */}
        DangerLevel.LOW -> { /* Do something */}
    }
}
```

We can add properties and functions to an enum if we like

```
enum class DangerLevel (val level: Int){
    HIGH(3),
    MEDIUM(2),
    LOW(1);

    fun getLevel() = "Level is $level"
}
```

And we can even have functions inside each value

```
enum class DangerLevel (val level: Int){
    HIGH(3) {
        override fun getLevel(): String = "This is dangerous"
    },
    MEDIUM(2) {
        override fun getLevel(): String = "This is not that dangerous"
    },
    LOW(1){
        override fun getLevel(): String = "This is not dangerous"
    };

    abstract fun getLevel(): String
}
```

This is kind of nice, but it would be more elegant to define this as an extension function

```
enum class DangerLevel (val level: Int){
    HIGH(3),
    MEDIUM(2),
    LOW(1)
}

fun DangerLevel.getLevel(): String {
    return when(this) {
        DangerLevel.HIGH -> "This is dangerous"
        DangerLevel.MEDIUM -> "This is not that dangerous"
        DangerLevel.LOW -> "This is not dangerous"
    }
}
```

Enums have convenient properties and methods that help us to work with them:
- name: returns the name of Enum's instance
- values(): returns an array of all items of Enum
- ordinal: returns the position of Enum's instance
- valueOf(<String>): returns an instance of Enum by its name with String type and case sensitivity

Other nice properties of enums are that their serialization/deserialization is simple and efficient, and they have automatically implemented toString, hashCode and equals methods.

## Summary
In this article, we've seen the differences between sealed classes and enums.
