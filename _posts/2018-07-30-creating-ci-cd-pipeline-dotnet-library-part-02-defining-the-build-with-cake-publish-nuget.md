---
author: johnny
date: 2018-07-30 20:00:01+01:00
layout: post
title: 'Creating a CI/CD pipeline for a .NET library: Part 2 - Defining the build with Cake and publishing to NuGet'
summary: 'In this second post we get started with the CI/CD pipeline, defining the build steps in C# using Cake.'
image: '/assets/2018/07/30/ci-post-image.jpg'
categories:
- dotnet
tags:
- ci
- cd
- continuous integration
- continuous delivery
- appveyor
- travis
- coveralls
- cake
---

[![CI/CD](/assets/2018/07/30/ci-post-image.jpg)](/assets/2018/07/30/ci-post-image.jpg)

# Intro
In this post I'll talk about defining the build steps using [Cake](https://cakebuild.net/) and publishing the result to [NuGet](https://www.nuget.org). From the official site: "Cake (C# Make) is a cross-platform build automation system with a C# DSL for tasks such as compiling code, copying files and folders, running unit tests, compressing files and building NuGet packages."

# Why Cake
Cake is one of several options available to define what needs to be done to build a project. 

The first thing that comes to mind to define the CI/CD tasks would probably be to use the tools already provided by something like AppVeyor, VSTS, Travis CI and all the other options. 

The first problem with this approach is in a case like this post introduces, where one wants to build in more than one CI provider, to test multiple operating systems, and that would mean repeating the tasks in each provider (and the same applies if we want to migrate to a different provider).

One more advantage of using Cake instead of directly depending on the CI providers is that we can run the build script locally. This is not only useful to test, but also if our build is not super straightforward, we can just use the scripts while in development.

Another option would be to create a PowerShell or Shell script defining the tasks. This would be provider agnostic but probably not operating system agnostic (although PowerShell now also runs on other platforms, so it would be possible).

Cake solves these problems and adds a couple of extra bonuses:
- We write things in C#, which is probably easier for many C# developers
- There are lots of builtin and pluggable helpers
- Also, worst case scenario where there’s something missing, it’s C#, we can just code what’s missing! (and I have an example of that in this post)

# Bootstrapping Cake
Before we start defining the build, we need to bootstrap Cake. This is done simply by downloading the `build.ps1` and `build.sh` from its resources [repository](https://github.com/cake-build/resources). Then executing one of these files (depending on your development operating system) will download the required dependencies and create a `build.cake` file, where the build will be defined.

# Defining the build tasks
Now we can open our `build.cake` file and start defining the required tasks. I’ll walk through my project’s build definition file (which was better organized after reading [this post](https://dev.to/jandedobbeleer/dotnet-core-served-with-a-slice-of-cake-5972)).

At the start of the file I’m declaring the dependencies on plugins and tools that’ll be used. There’s also a `using NuGet;` statement, because like I mentioned earlier, this is C# and I wanted to do some shenanigans.

{% gist f258b468659ff8318104e7b9f74db754 %}

Next there’s a bunch of variable definitions that’ll be used when defining the tasks. Some are just hardcoded constants, like project and file paths, others are created using arguments that are passed to the build script.

{% gist e70562d472399dcad5cc9617e8bff09b %}

Now let’s get into the tasks. There are seven of them: `Clean`, `Restore`, `Build`, `Test`, `UploadCoverage`, `Package` and `Publish`. Even if a bit overkill, I’ll go through each.

The `Clean` task deletes the artifacts directory and creates a new empty one to put everything that'll be generated in the next steps in there. It also runs `dotnet clean` at the solution level.

{% gist 78f5d67c6a8fe1ab3d0e93240a7253ea %}

The `Restore` task simply calls `dotnet restore` at the solution level to restore all required dependencies of the projects.

{% gist 13a863de87902bb04310dbc5e6ea2961 %}

The `Build` task basically runs `dotnet build -c Release` at the solution level. The task also defines it’s dependent on the `Clean` and `Restore` tasks, so those are executed before this one.

{% gist 70f9bf46a4c3b365863fb1ca716c9db4 %}

The `Test` task executes the tests in the respective project but adds some other settings pertaining to the code coverage report generation - `CollectCoverage`, `CoverletOutputFormat` and `CoverletOutput`.

[Coverlet](https://github.com/tonerdo/coverlet) is a tool used to generate the coverage reports in .NET Core.
After the tests are run, the generated code coverage report is moved to the artifacts directory. This is more for organization and to avoid having loose files in the development environment, as in the CI servers a new build is initiated from a completely clean environment.

{% gist 9f5981511efc922076dd0502e6ef135c %}

The `UploadCoverage` task, well… uploads the coverage report 😛<br/>
It uses a plugin to upload to Coveralls, getting as input the coverage report file path and token to authenticate itself with the service. The token is passed as an argument to the build script, being stored as an environment variable by the CI provider - I’ll talk about in the next post, on the section about AppVeyor.

{% gist 1cdc107f1798d44f6962af937702a34f %}

The `Package` task packs the library project into a pretty little NuGet package we can share on the interwebs.

{% gist dbf845be7693c289738613592b34b80c %}

The `Publish` task publishes the generated NuGet package into [nuget.org](https://www.nuget.org).

This is probably a questionable decision, as the usual approach would be to just create the package and publish afterwards if all is well. But as this is more of a didactic project than something serious and I’m kind of lazy to be publishing packages manually, I publish it directly in the build script, but only in release builds (done on the master branch).

It’s here the C# shenanigans come into play. To avoid trying to publish when the package version is the same - for instance when I’m just making adjustments to the build script and haven’t changed the project code - I’m importing the `Nuget.Core` NuGet package to use the API to check if this package version is already published.

{% gist f6bc515548a3bf83f88577bece5074e0 %}

To push the package to NuGet we need an API key. We can use a general key or create one for each project, which I would say it’s the ideal for security reasons. To create one you can go [over here](https://www.nuget.org/account/apikeys), hit create, give it a name, the owner of the package it pushes (for instance you may want an organization instead of yourself), set some options and the packages this key has access to.

[![Create NuGet API Key](/assets/2018/07/30/nuget-api-key-creation.jpg)](/assets/2018/07/30/nuget-api-key-creation.jpg)

# Defining build targets

The final thing in the `build.cake` file is the definition of build targets, which are tasks like the others, I’m just using them to try and make clear what should be used as argument when invoking the build script.

To target a specific task we could add the argument `--target NAME_OF_TASK` when invoking the build script. If nothing is passed then `Default` is assumed, which does a complete build, depending on the environment/branch - the development branch goes only ‘till the `UploadCoverage` task and the master branch goes all the way to the `Publish` one.

{% gist 2f651ec5dbc4425d8a1dfd04b21fd343 %}

# Running the build locally
Now that we defined the Cake build script, we can run it locally, just as it is going to be run in the cloud CI providers.

On Windows we can do:

`.\build.ps1 --currentBranch=develop --nugetApiKey=NUGET_API_KEY --coverallsToken=COVERALLS_TOKEN`

On Linux/MacOS the only difference would be the use of `./build.sh` instead of `.\build.ps1`.

Like I mentioned in the previous section, not passing a target, `Default` is used. Alternatively we could do, for example:

`.\build.ps1 --currentBranch=develop --nugetApiKey=NUGET_API_KEY --coverallsToken=COVERALLS_TOKEN --target BuildAndTest`

**Note**: while running on MacOS, the Coveralls publication fails with a weird error: 

[![MacOS Coverage Upload Error](/assets/2018/07/30/coverage-upload-error-mac.jpg)](/assets/2018/07/30/coverage-upload-error-mac.jpg)

As I only want the coverage to be published from one environment and I’m using AppVeyor as the primary one, it doesn’t annoy me too much and haven’t wasted time investigating the issue, but I’ll leave the note for future reference.

# Outro
On the Cake and NuGet part we're good to go, on the next post, we'll play around with AppVeyor and Travis CI to run the build in the cloud.

- [Part 1 - Intro]({% post_url 2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-01-intro %})
- [Part 2 - Defining the build with Cake and publishing to NuGet (this post)]({% post_url 2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-02-defining-the-build-with-cake-publish-nuget %})
- [Part 3 - Building on AppVeyor and Travis CI]({% post_url 2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-03-building-on-appveyor-and-travis-ci %})
- [Part 4 - Code coverage on Coveralls, badges and wrap up]({% post_url 2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-04-coverage-coveralls-badges-wrap-up %})

The accompanying code for these posts is [here](https://github.com/CodingMilitia/GrpcExtensions/tree/july-blog-post) (tagged to be sure the code reflects what's written here in the future).