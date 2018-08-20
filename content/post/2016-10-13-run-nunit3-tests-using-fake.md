---
title: 'Running NUnit3 tests using fake'
date: '2016-10-13'
categories:
- FSharp
tags:
- F#
- Unit Testing
- CI
---

When you have some unit tests developed using NUnit 2.x your [FAKE](http://fsharp.github.io/FAKE) script looks like something like this:

    Target "RunUnitTests" (fun _ ->
        !! (testDir + "/*.Tests.dll")
        |> NUnit (fun p ->
              {p with ToolPath = "packages/NUnit.Runners/tools/"})
)

But [NUnit3](https://github.com/nunit/nunit) works slightly different. Instead of having a single NUnit.Runners package, that package references some other packages (runner, extensions, etc). One of those packages is NUnit.ConsoleRunner that has the exe inside the tools folder.

But changing the ToolPath in the [FAKE](http://fsharp.github.io/FAKE) task is not enough. You should use the new [NUnit3 action](http://fsharp.github.io/FAKE/apidocs/fake-testing-nunit3.html), which is inside the Fake.Testing module. So, we need to open the module and change the task definition:

    let nunitRunnerPath = "packages/NUnit.ConsoleRunner/tools/nunit3-console.exe"
    
    Target "RunUnitTests" (fun _ ->
        !! (testDir + "/*.Tests.dll")
        |> NUnit3 (fun p ->
            {p with ToolPath = nunitRunnerPath})
)

And hopefully when you run the [FAKE](http://fsharp.github.io/FAKE) script  you'll have something like this:

![NUnit3 Results](/images/nunit3_results.png)

