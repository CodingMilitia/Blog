---
author: João Antunes
date: 2018-10-07 10:00:00+01:00
layout: post
title: "Episode 002 - Project structure plus first application - ASP.NET Core: From 0 to overkill"
summary: "In this post we finally see some code, although not much 😛 We start by checking out the project structure that'll be used in .NET repositories and then find our way through the command line to create the solution and C# project for the group management component."
images:
- '/assets/2018/10/07/aspnet-core-from-zero-to-overkill-e002.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-002-from-zero-to-overkill-project-structure-plus-first-application
---

In this post/video I’ll just begin setting up the project, describe the repository structure used and initialize the first ASP.NET Core application. Should be quick and simple 😉

Given this, the following video provides an overview, but if you prefer a quick read, skip to the written synthesis.

{{< yt ULUtAAHOyTw >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
As introduced, in this post I’ll just make the initial setup of the project, go through the repository structure I’ll use in .NET repositories (spoiler alert: there will be non .NET repositories along the way 😛) and initialize the first ASP.NET Core application, the group management component of our sample (eventually over-engineered) system.

I'm doing this on a Mac, so some commands may be bash specific. If you're on Windows, adapt accordingly.

## Creating the repository

To begin with I went to GitHub and created a new repository under the umbrella organization for this series ([AspNetCoreFromZeroToOverkill](https://github.com/AspNetCoreFromZeroToOverkill)) for the `GroupManagement` component, as seen on the last post.

As I'll do for the other repositories, I went with the [MIT license](https://opensource.org/licenses/MIT), as it's nice and permissive, I'm just doing this for the ride 🙂

Besides the license, I went with the option for the readme file (still empty, will fill that in eventually) but not the predefined `.gitignore` file, as I have another version derived from the one provided by GitHub, with a couple of added extras.

After this, I just needed to clone the repo to my computer `git clone https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement.git`.

## Repository structure

For the repository structure, I tend to follow something similar to what David Fowler (Microsoft architect for ASP.NET Core) recommends in his gist:

{% gist ed7564297c61fe9ab814 %}

(consider only the folder structure, the `.gitignore` I used has a lot more stuff)

Don't really have much else to add to this, as he explains in the gist, but feel free to ask if something isn't very clear.

## Initialize the application

Now that we know how we want to structure the repository, let's get started with making it happen.

For starters I downloaded the `.gitigore` from a previous project I have on GitHub: `curl https://raw.githubusercontent.com/CodingMilitia/GrpcExtensions/master/.gitignore -o .gitignore`

Then created a .NET solution. In this very simple post, I did it all from the command line and then viewed the code in Visual Studio Code, so creating a solution goes like this (inside the cloned repository): `dotnet new sln -n CodingMilitia.PlayBall.GroupManagement`.

Then, following the structure described, do `mkdir src` to create the folder for the project source code and inside that directory `mkdir CodingMilitia.PlayBall.GroupManagement.Web` to create the directory for the first application, named after the complete namespace plus `Web`, as it'll be the web application. 

Moving into the application folder, we can do a `dotnet new --help` to see what are the available project types:

```
Templates                                         Short Name         Language          Tags
----------------------------------------------------------------------------------------------------------------------------
Console Application                               console            [C#], F#, VB      Common/Console
Class library                                     classlib           [C#], F#, VB      Common/Library
Unit Test Project                                 mstest             [C#], F#, VB      Test/MSTest
NUnit 3 Test Project                              nunit              [C#], F#, VB      Test/NUnit
NUnit 3 Test Item                                 nunit-test         [C#], F#, VB      Test/NUnit
xUnit Test Project                                xunit              [C#], F#, VB      Test/xUnit
Razor Page                                        page               [C#]              Web/ASP.NET
MVC ViewImports                                   viewimports        [C#]              Web/ASP.NET
MVC ViewStart                                     viewstart          [C#]              Web/ASP.NET
ASP.NET Core Empty                                web                [C#], F#          Web/Empty
ASP.NET Core Web App (Model-View-Controller)      mvc                [C#], F#          Web/MVC
ASP.NET Core Web App                              razor              [C#]              Web/MVC/Razor Pages
ASP.NET Core with Angular                         angular            [C#]              Web/MVC/SPA
ASP.NET Core with React.js                        react              [C#]              Web/MVC/SPA
ASP.NET Core with React.js and Redux              reactredux         [C#]              Web/MVC/SPA
Razor Class Library                               razorclasslib      [C#]              Web/Razor/Library/Razor Class Library
ASP.NET Core Web API                              webapi             [C#], F#          Web/WebAPI
global.json file                                  globaljson                           Config
NuGet Config                                      nugetconfig                          Config
Web Config                                        webconfig                            Config
Solution File                                     sln                                  Solution
```

These are the default project templates that ship with the .NET Core 2.1 SDK.

There are a bunch of web project types, but we'll go with the empty one, because I want to start from scratch and build everything, instead of having a lot there already and deleting what's not needed. I think it's better for learning purposes, although the other templates may be great for quick starts.

Given this, `dotnet new web` it is. Besides creating the necessary files, the required dependencies are immediately restored, no need to do it manually as in previous versions of .NET Core.

## The code

Now we can see the code. To do this, I just go into the solution root and type `code .`, and it opens Visual Studio Code in that folder - doing this requires you to have this feature configured, but I think it's configured by default in the latest versions of Visual Studio Code.

Looking at the code we can see that there's not much in there, and it shouldn't, as we specifically used the empty web application template. The most relevant files to look at are `CodingMilitia.PlayBall.GroupManagement.Web.csproj`, `Program.cs` and `Startup.cs`.

The `CodingMilitia.PlayBall.GroupManagement.Web.csproj` file contains information on the project. 
It indicates:
- the used SDK - if you notice, it's different from a console application one, with some specific MSBuild tasks and targets for web applications
- the target framework - in this case `netcoreapp2.1`
- folders that should be copied to the output - `wwwroot` in this case
- package references - where we can ad dependencies to packages outside of our solution - right now only the meta package with the required APIs to create ASP.NET Core applications (I'll talk more about this metapackage in the future)

There's a bunch more that can be done in this file, but upon initialization, that's all there is to it. More info on the `csproj` file [here](https://docs.microsoft.com/en-us/dotnet/core/tools/csproj).

The `Program.cs` file contains code for application bootstrapping. The first thing we can take away from it is that an ASP.NET Core application is just like a console application, with a `Main` method as entrypoint. Then it just calls the `CreateWebHostBuilder` to initialize Kestrel (ASP.NET Core's embedded server) and start listening for requests.

While looking at the `Program` class, we see the reference to the `Startup` class. This is where we'll configure the dependency injection container and the request handling pipeline (more details about these two in the next post). 
One thing to note as well regarding using this `Startup` class, is that it's basically a convention. If you add a dot after `WebHost.CreateDefaultBuilder(args)` to see what intellisense shows, you'll notice there are available `Configure` and `ConfigureServices` methods, so we wouldn't need to use the extra class, it's just that it is a better way to organize things.

Getting back to the `Startup` class, we can see that in the configure method, among other things there is a `app.Run(...)` method call. This is defining a middleware - more detail on this on a later post, but in summary, it allows us to build the request handling pipeline which can be composed of several middlewares, each passing the request to the next until one handles provides the response - that'll respond to every request with an `Hello World!` `string`.

After this quick look, let's see this working. On the project folder (not the solution root) we can do `dotnet run` and the application starts, listening on port 5000 (or 5001 for HTTPS). Going to the browser gets us exactly what we expect, a blank page with `Hello World!`.

## Outro

Now that we prepared the repository structure and the initial project (and not much else really), we can get on with the coding.
In the next post, we'll start with a simple ASP.NET Core MVC application.

Don't hold back on feedback! Hope you'll be back for the next one!

Thanks for reading, cyaz!