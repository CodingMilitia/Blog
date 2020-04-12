---
author: João Antunes
date: 2018-07-30 20:00:02+01:00
layout: post
title: 'Creating a CI/CD pipeline for a .NET library: Part 3 - Building on AppVeyor and Travis CI'
summary: "In this third post we get our build running in the cloud, compiling and testing in Windows, Linux and MacOS."
images:
- '/assets/2018/07/30/ci-post-image.jpg'
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
slug: creating-ci-cd-pipeline-dotnet-library-part-03-building-on-appveyor-and-travis-ci
---

[![CI/CD](/assets/2018/07/30/ci-post-image.jpg)](/assets/2018/07/30/ci-post-image.jpg)

# Intro
On the last post, we defined the build using [Cake](https://cakebuild.net/), including publishing the resulting artifacts to [NuGet](https://www.nuget.org/) and [Coveralls](https://coveralls.io/) (although we'll see more detail on Coveralls later). In this post we'll use [AppVeyor](https://www.appveyor.com/) and [Travis CI](https://travis-ci.org/), which provide free continuous integration hosting for open source projects.

# Adding the project to AppVeyor
AppVeyor is one of the main CI cloud providers used with .NET projects, particularly before the multiplatform .NET Core arrived, but still plenty useful, if not for anything else, to build on Windows.

## Setting up the project on the AppVeyor site
After creating an account, adding a project is really simple. 

On the home page, create new project:

[![AppVeyor Home](/assets/2018/07/30/appveyor-home.jpg)](/assets/2018/07/30/appveyor-home.jpg)

Then choose a source. In my case GitHub, making sure to authorize AppVeyor to access the GitHub repositories:

[![AppVeyor New Project](/assets/2018/07/30/appveyor-new-project.jpg)](/assets/2018/07/30/appveyor-new-project.jpg)

The next screen will be empty, in my case there’s already some information on previous builds (including the logs):

[![AppVeyor Latest Build](/assets/2018/07/30/appveyor-latest-build.jpg)](/assets/2018/07/30/appveyor-latest-build.jpg)

The next step would be to go into the settings section and set everything up, but I didn’t change anything because I’m using the Cake build script and an `appveyor.yml` configuration file in the project repository to configure everything.

## Configure using appveyor.yml

Like mentioned in the previous section, the configuration of the build could be done on the AppVeyor website, but using the `appveyor.yml` file we can do the same things and keep everything together in source control.

Considering the bulk of the work is done in Cake, the AppVeyor configuration is really simple - I’ll just insert it here and then go through it.

{% gist 6ece9e908b165059539aa7b10ddf7518 %}

Starting with the version. I don’t use this at all, as I’m defining the version manually in the library’s `csproj` file, but the default is `1.0.{build}` and it is shown when we are looking at the latest builds, I preferred changing it to `{build}` to just map directly to the build number.

The `image` is the base virtual machine image that’ll be used to execute the build. We can configure more than one, including Linux, but as I’m delegating the other operating systems to Travis CI, I’m just using a Windows image with Visual Studio 2017.

The `branches` configuration part is where we can define the branches we want to trigger a build. This can be a whitelist, like in this example, or a blacklist.

Then, in the `environment` section we can define required environment variables. The first two are just to skip some .NET unneeded steps, to minimize build time. The latter two are the API keys for publishing the package to NuGet and to publish the code coverage report to Coveralls. Notice the API keys are not in plain text - that wouldn’t end well - they’re encrypted using a tool provided by AppVeyor ([over here](https://ci.appveyor.com/tools/encrypt)) so they can be safely stored in source control.

[![AppVeyor Encrypt Data Menu](/assets/2018/07/30/appveyor-encrypt-data-menu.jpg)](/assets/2018/07/30/appveyor-encrypt-data-menu.jpg)

[![AppVeyor Encrypt Data](/assets/2018/07/30/appveyor-encrypt-data.jpg)](/assets/2018/07/30/appveyor-encrypt-data.jpg)

Finally we have the build “definition” in `build_script`, that’s just calling the Cake build script, providing it with the required arguments. As AppVeyor is already very .NET focused, it has some default behaviors that we don’t need as they’re being taken care of by the Cake script, so we must disable the `test` and `deploy` stages.

There is a lot more we can configure, but this is all I need for this project. More info about `appveyor.yml` [here](https://www.appveyor.com/docs/appveyor-yml/).

With all of this configured, the next time we push changes to the repository (as long as it’s either on the development or master branch) a new build is triggered automatically. In the project page we can see the build history and the logs for the builds, among other things.

# Adding the project to Travis CI
The setup for Travis is similar to AppVeyor.

## Setting up the project on the Travis CI site
On the initial page we can click to create a new project, and also see the other existing projects and information about them.

[![Travis CI Home](/assets/2018/07/30/travis-home.jpg)](/assets/2018/07/30/travis-home.jpg)

When creating a new project, the GitHub projects appear front and center to choose from.

[![Travis CI New Project](/assets/2018/07/30/travis-new-project.jpg)](/assets/2018/07/30/travis-new-project.jpg)

On the project page we can see the information about the project, like previous build results and logs, project settings and other things. This is the same page as the initial one, just focused on the selected project.

Regarding the configuration, I’m using the same approach as with AppVeyor, a `.travis.yml` checked into source control.

## Configure using .travis.yml

As you’ll see, the logic is very similar to AppVeyor, just using a different file format. Again, I’ll be focusing on the things I needed in this case, more info about `.travis.yml` file [here](https://docs.travis-ci.com/user/customizing-the-build/).

Using the same approach, complete file, then walk through its parts.

{% gist eb484907eadc0fc828e87defffa4fe4b %}

Starting with the `language`, this basically defines some default steps.

Then we define a `matrix` that will correspond to a build job per entry. In this case I’m only using this for building in different operating systems, but it could also be used for building with different dependency versions, OS versions and other things you may think of.

The `branches` section is exactly the same as in AppVeyor, I’m defining a whitelist of branches I want to trigger the builds.

The `env` section provides the required environment variables. In this case the API keys are not needed as I’m only publishing stuff from AppVeyor, so no need to have them here.

`mono` tells Travis we need [Mono](https://www.mono-project.com/) installed (Cake requires Mono to run when not in Windows) and `dotnet` tells Travis we need the .NET Core SDK.

Finally the `script` is just passing the ball to Cake to do it’s thing, passing it a different target, as we only need to build and test, publishing of code coverage and packages is done elsewhere.

With all this in place, the next push will trigger a build, or in this case two builds - one in Ubuntu and another on MacOS.

Like in AppVeyor, we can go to the project page and see the build logs.

# Outro
With all this in place we have a functional CI/CD pipeline already, with th build defined and AppVeyor and Travis waiting on GitHub changes to trigger new builds.

On the next post we'll put the finishing touches, namely adding the project to Coveralls so we can visualize the status of our tests code coverage.

- [Part 1 - Intro]({% post_url 2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-01-intro %})
- [Part 2 - Defining the build with Cake and publishing to NuGet]({% post_url 2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-02-defining-the-build-with-cake-publish-nuget %})
- [Part 3 - Building on AppVeyor and Travis CI (this post)]({% post_url 2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-03-building-on-appveyor-and-travis-ci %})
- [Part 4 - Code coverage on Coveralls, badges and wrap up]({% post_url 2018-07-30-creating-ci-cd-pipeline-dotnet-library-part-04-coverage-coveralls-badges-wrap-up %})

The accompanying code for these posts is [here](https://github.com/CodingMilitia/GrpcExtensions/tree/july-blog-post) (tagged to be sure the code reflects what's written here in the future).