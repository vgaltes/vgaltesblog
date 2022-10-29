---
title: 'Learning Kotlin: configuring Kotest with Gradle'
thumbnail: /images/LearningProject1-CreateNewProject.png
date: '2022-10-29'
categories:
- kotlin
tags:
- kotest
- gradle
comments: []
---

I'm about to start a new job. This will be a huge challenge for me, as I'm going to be developing in Kotlin. I've spent all of my professional life working with .Net, so I need to learn LOTS of things. As I believe that writing down what you learn is the best way to solidify it, I'll be documenting all that I learn (maybe not all, but a bing chunk of it :D). Don't expect this to be a guide to learn Kotlin, or Gradle, or whatever, but just a dump of my brain. Yet, I hope you can learn something. Feel free to correct me if I'm doing anything wrong, or suggesting best ways to do it.

## Installing the tools
We'll be using IntelliJ, Gradle, Kotlin, and Kotest. I'll be using a Mac, so if you have Windows some steps won't work for you.

The first thing you need to do is to install brew. You can do this with this command:
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"

```

Once you have brew, you can configure your machine in many ways. I recommend you to use dotfiles. In this tutorial we will install everything by hand so the steps will be clearer.

The next step is to install IntelliJ. You can do this by going to it's [product web site](https://www.jetbrains.com/idea/download/)

After that, we will install sdkman. Sdkman is a nice tool to install diferent versions of different sdks. In our case, we will use it to install Java and Gradle. So, let's install Sdkman first:

```
curl -s "https://get.sdkman.io" | bash

source "$HOME/.sdkman/bin/sdkman-init.sh"
```

Once we have Sdkman installed, we are ready to install Gradle and Java. Let's start with Gradle

```
sdk install gradle
```

And we can continue with Java. In our case, we need Java 17.

```
sdk install java 17.0.4-amzn   
```

## Creating the project
We're now ready to create our **very** simple test project. The goal of the project will be to learn to setup a project with Grale and Kotest.

So, go to IntelliJ and create a new project. Language should be Kotlin, build system should be Gradle, select the Java 17 SDK, and select Kotlin as the Gradle DSL.

![Create New Project](/images/LearningProject1-CreateNewProject.png)

## Configure Gradle
Now we need to configure Gradle so we can run the build. The file that we're going to be working on is the one called `build.gradle.kts`. First we will need to add an import:

```
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile
import org.gradle.api.tasks.testing.logging.TestExceptionFormat.*
import org.gradle.api.tasks.testing.logging.TestLogEvent.*
```

Change the dependencies section so we can use Kotest:

```
dependencies {
    val kotestVersion = "5.4.1"
    testImplementation("io.kotest:kotest-runner-junit5:$kotestVersion")
    testImplementation("io.kotest:kotest-assertions-core:$kotestVersion")
}
```

Now we need to "link" Kotest with JUnit5. To do this, we need to add useJUnitPlatform() inside the tasks with type Test:

```
tasks.withType<Test>{
    useJUnitPlatform()

    testLogging {
        events(FAILED, STANDARD_ERROR, SKIPPED)

        exceptionFormat = FULL
        showExceptions = true
        showCauses = true
        showStackTraces = true
    }
}
```

This will do the job along the junit5 dependency we added earlier. As you can see we're also configuring the output of the Gradle test task, so we will have more information if something goes wrong.

## Writing our first tests
Now it's time to add some simple test. Go to the `src/test/kotlin` and create a new file called `MyFirstTest` with the following content:

```
import io.kotest.matchers.shouldBe
import io.kotest.core.spec.style.StringSpec
import io.kotest.matchers.should
import io.kotest.matchers.string.startWith

class MyFirstTest : StringSpec({
    "length should return size of string" {
        "hello".length shouldBe 6
    }
    "startsWith should test for a prefix" {
        "world" should startWith("wor")
    }
})
```

## Running the tests
We have several options to run the tests. Let's start with the most visual way. Let's go to IntelliJ -> Preferences -> Plugins and search for `kotest`. Install the plugin and you'll be able to run the tests from the file or from the plugin window.

If you want to run the tests from the build you need to run the following command:

```
./gradlew test
```

## Summary
In this article, we've seen how to setup a project to start learning Kotlin and its platform. We now can run tests using Kotest so we can write a lot of them to learn how Kotlin works. 
