---
author: João Antunes
date: 2018-10-02 22:40:00+01:00
layout: post
title: 'ASP.NET Core: From 0 to overkill - Intro'
summary: "This post serves as an introduction to a new series of videos (and I'll try to accompany them with posts) on creating a complex application using on ASP.NET Core."
images:
- '/assets/2018/10/02/aspnet-core-from-zero-to-overkill-post-image.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-000-from-zero-to-overkill-intro
---

This post serves as an introduction to a new series of videos (and I'll try to accompany them with posts) on creating a complex application using on ASP.NET Core.

You can go through the intro video here, or scroll past it and read the written synthesis.

{{< yt 4FXUrEY9PIQ >}}
<br />
## Intro

As introduced, the goal of this series will be to build a complex application using ASP.NET Core, starting simple and complicating along the way. 

The end result will probably be on the over-engineered side (hence the "overkill" on the title), because the application will be developed as an excuse to try a whole lot of technologies and concepts. If the application wasn't being developed with educational purposes in mind, the outcome would certainly be simpler.

I’m saying this series will be about ASP.NET Core, but that’s not 100% correct, as I’ll probably do some stuff not specific to ASP.NET, but it will still be the central piece of the application.

I’m assuming C# understanding, but feel free to ask for some explanations on the language if you find it valuable in this context.

## Reference project

As a reference project I’ll be creating an application that handles organizing sports events between friends, keeping statistics and some other features we’ll discuss along the way. I’ll try not to make it focused on football (or soccer for our US friends) which was my initial idea, but we’ll see along the way how hard is it to make something like this generic (yeah, I don’t know yet, I didn’t invest that much time planning this whole thing in that regard 😝).

Didn't want to go with the usual blog or shopping samples, and I think this sample will allow for a lot of concepts, tools and techs to be tried out.

As already mentioned, the end result will very likely be over-engineered, considering the problem being solved is not that complex, but given the goal is to learn and try new stuff, I don't think anyone minds.

## Tools

Will start with [.NET Core](https://www.microsoft.com/net) and ASP.NET Core 2.1, but will upgrade as new versions appear. To install .NET Core, you can download from Microsoft’s site, but as a maybe simpler alternative you can use [Homebrew](https://brew.sh/) on MacOS or [Chocolatey](https://chocolatey.org/) on Windows (and probably apt-get on Linux, but haven’t tried it).

Regarding tooling:

- As the IDE, I’ll use [JetBrains Rider](https://www.jetbrains.com/rider/) when on MacOS or Linux (not free though, unless for open source and student accounts). When on Windows, [Visual Studio 2017](https://visualstudio.microsoft.com/vs) will probably be the choice. Another alternative for MacOS would be Visual Studio for Mac or [Visual Studio Code](https://code.visualstudio.com/) for all platforms.
- [Visual Studio Code](https://code.visualstudio.com/), for non C# stuff.
- [Docker](https://www.docker.com/) - for starters only for supporting stuff, like databases and stuff like that, eventually will put the app in there as well. Alternatively you can install the databases directly on the computer, but I prefer not installing everything I want to test if I can, to keep the system clean.
- [Git](https://git-scm.com) for source control. The project will be public on [GitHub](https://github.com).
- When not using Git from the command line, I’ll use [GitKraken](https://www.gitkraken.com/), [Git Extensions](https://github.com/gitextensions/gitextensions) or [Sublime Merge](https://www.sublimemerge.com/) - this last one is a new tool from the [Sublime Text](https://www.sublimetext.com/) folks and I want to try it out
- [Postman](https://www.getpostman.com/) for testing HTTP requests
- [iTerm2](https://www.iterm2.com/) to do command line stuff on the Mac, just for a couple of extra features, the default terminal would also suffice. When on Windows I use [Cmder](http://cmder.net/). If you want bash on Windows, you can also go with WSL (Windows Subsystem for Linux) in Windows 10. On Linux, the default terminal will do the trick.

## Outro

Now let's get this started, if I mess something up or if you have any questions, please let me know!
Also, the project will be open for pull requests, do feel free to help me with this.

Hope this is useful, cyaz!