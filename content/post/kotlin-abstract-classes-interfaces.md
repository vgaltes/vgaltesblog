---
title: 'Learning Kotlin: abstract classes and interfaces'
date: '2022-11-05'
categories:
- kotlin
tags:
- abstract class
- interface
comments: []
---

When you're studying Kotlin and take a look at abstract classes and interfaces, your first impression is that they are two very similar constructs, or at least that was my first impression. You can use both of them to share state and behavior

So, let's see what are they and what are the difference amongst them.

## Abstract classes
With abstract classes, using the keyword *abstract* you can declare members without implementation. That implementation will be provided by the class that derives from the abstract class, using the keyword override. 

```
abstract class Vehicle {
    abstract fun honk()
}

class Car: Vehicle {
    override fun honk() = prinln("meeeek")
}
```

But you can also, and this is something you can't do with interfaces, define functions that have an implementation and that can't be overriden in the derived classes.

You can't create a new instance of an abstract class, but you can create constructors and init blocks. The order of execution is the same that would be if the class wasn't abstract. Let's take a look. Imagine that we have these classes:

```
abstract class Person(val name: String) {
    var surname: String = ""
    var messages: MutableList<String> = mutableListOf()

    constructor(name: String, surname: String): this(name) {
        this.surname = surname
        this.messages.add("Call to first secondary constructor in Person")
    }

    init {
        this.messages.add("Call to first init block in Person. Primary constructor was already called with $name")
    }

    init {
        this.messages.add("Call to second init block in Person")
    }
}

class Spaniard : Person {
    var secondSurname = ""

    constructor(name: String, surname: String, secondSurname: String) : super(name, surname) {
        this.secondSurname = secondSurname
        this.messages.add("Call to first secondary constructor in Spaniard")
    }

    constructor(name: String) : this(name, "no surname", "no surname") {
        this.messages.add("Call to second secondary constructor in Spaniard")
    }

    init {
        this.messages.add("Call to first init block in Spaniard")
    }

    init {
        this.messages.add("Call to second init block in Spaniard")
    }
}
```

If we now create a new object of the `Spaniard` class with "Jose" as parameter, the message that we'll have in the message list would be:

```
Call to first init block in Person. Primary constructor was already called with Jose
Call to second init block in Person
Call to first secondary constructor in Person
Call to first init block in Spaniard
Call to second init block in Spaniard
Call to first secondary constructor in Spaniard
Call to second secondary constructor in Spaniard
```

As you can see, the primary constructor of the abstract class is called first, then the init blocks of the abstract class in appearance order, then the secondary constructor in the abstract class, and then the same order in the concrete class.

Logically, by default abstract classes are open and their abstract methods too. And functions declared as abstract cannot be declared as final.

## Interfaces
We can do something similar with interfaces. There are some differences though between abstract classes and interfaces.

In an interface, by default everything is open and we can override everything. We can have default method implementations in interfaces. If we do that, we don't need to override those methods. Let's have a look to an example:

```
interface OnlyNamePerson {
    val name: String
    fun sayHello(name: String): String
    fun sayGoodBye() : String {
        return "Bye from $name"
    }
}

class NoFamilyPerson(override val name: String) : OnlyNamePerson {
    override fun sayHello(name: String): String {
        return "Hello $name, my name is ${this.name} and I have no surname"
    }
}
```

As you can see, we need to override the sayHello method because it has no body in the interface declaration, but we don't need to override the sayGoodBye method because it has already an implementation.

Interfaces don't have constructors or fields. They only can have properties and functions.

One big difference between an abstract class and an interface is that, as an abstract class is just another type of class, you can only inherit from one abstract class, but you can implement as many interfaces as you want.

## Traits
One interesting capability of interfaces in Kotlin is that you can implement [traits](https://en.wikipedia.org/wiki/Trait_(computer_programming)) with them. This allow us to compose objects with behaviors defined in interfaces. Let's take a look to an example:

Imagine that we're developing a game and we need a way to create players. Let's start with some generic interfaces:

```
interface Level {
    fun getLevel(): Int
}

interface Enemy {
    fun isEnemy(): Boolean
}

interface Class {
    fun getClass(): String
}
```

Without traits, if we wanted to create a NotDangerousFriendlyWizard we'd needed to do something like this:

```
class NotDangerousFriendlyWizard : Class, Level, Enemy {
    override fun getLevel(): Int = 1
    override fun isEnemy(): Boolean = false
    override fun getClass(): String = "Wizard"
}
```

And what happens if we want to create a NotDangerousFriendlyElf? That we need to create something like this: 

```
class NotDangerousFriendlyElf : Class, Level, Enemy {
    override fun getLevel(): Int = 1
    override fun isEnemy(): Boolean = false
    override fun getClass(): String = "Wizard"
}
```

As you can see, our classes are quite big and there's a lot of code duplication.

Let's use traits (via interfaces with default implementations) in order to have a more concise code. First, we need to create the interfaces with the default implementations:

```
interface NotDangerous : Level { override fun getLevel(): Int { return 1 } }
interface Friend : Enemy { override fun isEnemy(): Boolean { return false } }
interface Wizard : Class { override fun getClass(): String { return "Wizard"}}
```

Now, we can declare our NotDangerousFriendlyWizard like this:

```
class NotDangerousFriendlyWizard : NotDangerous, Friend, Wizard
```

As the class is implementing those interfaces but those interfaces have those methods already declared with a default implementation, we can use those implementations. Nice, uh?

(we can accomplish the same thing with object declaration and object delegation, by the way)

## Summary
In this article, we've seen the differences between abstract classes and interfaces.