---
author: João Antunes
date: 2018-07-30 20:00:00+01:00
layout: post
title: 'Creating a CI/CD pipeline for a .NET library: Part 1 - Intro'
summary: 'Setting up a complete CI/CD pipeline for a .NET library, from building and testing in different platforms, visualizing code coverage and publishing the binaries to NuGet.'
images:
- '/images/2018/07/30/ci-post-image.jpg'
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
slug: creating-ci-cd-pipeline-dotnet-library-part-01-intro
---

{{< embedded-image "/images/2018/07/30/ci-post-image.jpg" "CI/CD" >}}

In this series of posts I want to talk about creating a CI/CD (continuous integration/continuous delivery) pipeline for a .NET library that’s distributed as a NuGet package. As reference, I’ll use the [GrpcExtensions](https://github.com/CodingMilitia/GrpcExtensions) library that I talked about in a previous [post]({{< ref "2018-05-19-dotnetifying-grpc-sane-edition " >}}).

Given we’re in this new world of cross platform .NET, I want to build and test stuff in more than just Windows, so I’m using [AppVeyor](https://www.appveyor.com/) and [Travis CI](https://travis-ci.org/), where the former handles building on Windows and the latter handles the Linux and MacOS work.

Considering the use of more than one provider for the CI/CD pipeline, [Cake](https://cakebuild.net/) is useful to define the required tasks in a provider and system agnostic manner. It’s also cool that we can define the build tasks using C#, which is rather easy to understand given the rest of the project is already C#.

Finally, to have visibility on the tests code coverage I’m using [Coveralls](https://coveralls.io/) to host this information.

It would be also good to add some static code analysis to the mix, like [SonarQube](https://www.sonarqube.org/), but haven’t done it yet, maybe a good topic for another post.

**Disclaimer**:
I’m assuming the reader understands what I’m talking about from the very small intros I provide, but if I’m assuming wrongly (which is very possible) feel free to ask and I’ll answer and eventually improve the posts to take into account.

Going with a series of posts instead of a giant one to be (hopefully) easier to read. 
The following is the list of posts for easy navigation.

- [Part 1 - Intro (this post)]({{< ref "2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-01-intro " >}})
- [Part 2 - Defining the build with Cake and publishing to NuGet]({{< ref "2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-02-defining-the-build-with-cake-publish-nuget " >}})
- [Part 3 - Building on AppVeyor and Travis CI]({{< ref "2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-03-building-on-appveyor-and-travis-ci " >}})
- [Part 4 - Code coverage on Coveralls, badges and wrap up]({{< ref "2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-04-coverage-coveralls-badges-wrap-up " >}})

The accompanying code for these posts is [here](https://github.com/CodingMilitia/GrpcExtensions/tree/july-blog-post) (tagged to be sure the code reflects what's written here in the future).

Cyaz