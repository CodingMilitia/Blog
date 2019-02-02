---
author: João Antunes
date: 2018-10-27 21:45:00+01:00
layout: post
title: "Episode 004 - The Program and Startup classes - ASP.NET Core: From 0 to overkill"
summary: "In this episode we take a step back to look at some important infrastructure parts of an ASP.NET Core application - the Program and the Startup classes."
image: '/assets/2018/10/27/aspnet-core-from-zero-to-overkill-e004.jpg'
categories:
- fromzerotooverkill
tags:
- dotnet
- aspnetcore
---

In this post/video we take a step back to look at some important infrastructure parts of an ASP.NET Core application - the `Program` and the `Startup` classes. Real advancements on our reference project will be a bit on hold for some episodes, as we focus on core concepts that help understanding what we will be doing going forward.

For the walk-through you can check the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/kc4PzBHK2fA" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In the last episode we started our group management application using MVC. Today however, we'll take a step back to look at some core concepts in ASP.NET Core, so we can better understand what's coming in the future, and not be overwhelmed with too many concepts at the same time.

## The Program class
### Just a console application
An ASP.NET Core application is basically a console application that hosts a web server (Kestrel). Knowing this, there's no surprise to find a `Program` class with the classic `Main` method we see on traditional console applications.

This means we can do pretty much anything as we would on a console application before running the server, mainly things that we don't want to do in the context of the ASP.NET Core application per se. For instance, something I've seen being done here is to check if the database is up do date and apply any required migrations before starting the application, although one could argue that:
1. Shouldn't be doing this kind of database changes automatically on application start (at least in production)
2. Even if we really want to make these database changes, we can do it in the `Startup` class (or another step of preparing the web host).

### Building the web host
To get our application listening to HTTP requests, we must create a `WebHost`. Prior to ASP.NET Core 2.0, there was more configuration we needed to do to get it running, but with that version some helper methods were added we just need one line to get the `WebHost` started: `WebHost.CreateDefaultBuilder(args).UseStartup<Startup>();` (being `args` the arguments passed in when starting the application through the command line).

Of course this helper methods just abstracts away what was needed to do every time we wanted to create a new application. If we're not happy with the default, we can bypass this helper and do it ourselves. For reference, we can just go and see the source code for the helper method on GitHub - [here's the link to the version on ASP.NET Core 2.1](https://github.com/aspnet/MetaPackages/blob/96be626c87c3ca325b18aa6653602f5e7087497f/src/Microsoft.AspNetCore/WebHost.cs#L150).

Looking at the `CreateDefaultBuilder` code we see somethings that one who worked with ASP.NET Core < 2.0 will recognize, like configuring the use of Kestrel, defining the application's root content directory, setting up configuration sources, setting up logging and a couple other things. Having this abstracted away is nice, because many times it will suffice, but it's important to know it's there, and we can override it to match our needs.

Besides the `CreateDefaultBuilder` we see the call to `UseStartup<Startup>()`. We'll take a closer look at the `Startup` class in a moment, but this method call basically tells the `WebHostBuilder` the class that should be use for initialization purposes. Doing this is optional, as we could, instead of using the `Startup` class, call `ConfigureServices` and `Configure` directly on the `WebHostBuilder`.

To get our application running, we call `Build` to get the instance of `WebHost` out of the `WebHostBuilder`, and then `Run` on it to start.

## The Startup class
Like I mentioned earlier, the `Startup` class contains some initialization code for our ASP.NET Core application, namely in the form of the `ConfigureServices` and `Configure` methods.

The names of the methods are not great, but to make it clear, `ConfigureServices` is where we configure dependency injection - as we saw in the last episode, `services.AddMvc()` adds MVC's required services to the DI container - and `Configure` is where we configure the request handling pipeline - as we've seen, `app.UseMvc()` adds MVC's request handling middleware to the application's pipeline.

Now let's play a little more with this methods, to get a better feel of their uses as a preparation to dive a little bit deeper in the next episodes.

### Configure method
Like we talked, the `Configure` method is where we configure our request handling pipeline. Given we're building an MVC application, we added `app.UseMvc()` to register MVC's middleware to handle the requests. The MVC middleware should be the last one registered, as we want it to be the final handler for our requests in most cases, but we can add some middlewares earlier in the pipeline to enrich the request/response or even handle things that we don't want MVC to handle.

#### Creating a simple middleware
Let's start with creating a very simple middleware that adds an header to every response. To keep it really simple we'll use the `app.Use(...)` method (extension method of `IApplicationBuilder`) and pass it a lambda function that represents the middleware. There are more ways to create middlewares, we'll look at those in a future episode.

As we've talking, the basic idea of middlewares is that they work like in a chain - a middleware is invoked, it can do something, then pass the request along to the next middleware and finally do something else when the next middleware is done. There can be cases where the next middleware isn't invoked, for instance in a authorization middleware, if the user isn't authenticated, it can immediately stop the request, returning a response by itself. Given this, the base of the middleware will be as follows:

{% gist 5ca2d3f5dc469e96ca427051c57a2f88 %}

With this in mind, our middleware for adding a response header will be really simple, having only logic before invoking the next middleware.

{% gist d66d1ece902024d833d4ac9ab3be52fe %}

As you can see, the first thing the middleware does is register a callback on `context.Response.Starting()`, so the lambda passed in is executed before the response starts being delivered to the client. Next and final thing the middleware does is invoking `next()`, so the next middleware in the chain can start to do its thing (in this case, the MVC middleware).

If we now make a request to the application and check the headers, among other that are already put there by ASP.NET Core, we can see the added `X-Powered-By`.

![Response header added by middleware](/assets/2018/10/27/response-header-added-by-middleware.jpg)

#### Using static files
Now that we made a really simple middleware, let's take a look at another builtin one, the static files middleware. In the `Configure` method, before our custom response header middleware, we simply add the line `app.UseStaticFiles();`. Now we can add a `wwwroot` folder to the root of the web application project, and any static file we put there. This is just the default place from which the middleware retrieves files from, we can configure other folders or even providers that fetch images from a database, external image providers, etc.

Remember, the order in which the middlewares are registered is important, so given that we used the static files middleware before the custom response header one, the header won't be present when we fetch static files, only when the request reaches MVC.

In the end, the `Configure` method looks like this:

{% gist 7c96f7b2f6bd55be83b6d7c2236e0bef %}

I haven't really talked about the development exception page middleware that's present in the beginning of the method (part of the project template), but by now, knowing how all of this works, you can guess what it does - if we're in development environment, a page with exception details is shown when an unhandled error occurs, when not in development, this information is not shown.

### ConfigureServices method
Taking a look at the other method present in the `Startup` class, we have `ConfigureServices`. As we talked about, `ConfigureServices` is where we register our applications dependencies, so they can be injected where needed. Until now we only had registered the MVC dependencies, now let's add a dependency for our own usage.

In the root of the project, I'm adding a a new folder `Demo` and in there creating 2 files: `IGroupIdGenerator` and `GroupIdGenerator` - you can guess that the first will be an interface and the second a class that implements the former.

{% gist 824f113de5bdd3e61a503178531d17d4 %}

Pretty simple stuff, just to have something to inject 😛 The interface has a single method, to retrieve the next id. The implementation stores the last used id in an instance field, which is incremented in every method call.

To use this id generator, we head to the `GroupsController`, remove the `currentGroupId` field we had there and inject - as in receive as a constructor argument - the newly created `IGroupIdGenerator`, storing it in an instance variable.

{% gist 3a00d47b55c1a2b9d86e16673c110653 %}

Now we can use the id generator on the action responsible for creating a new group, instead of the id field we had previously.

{% gist fbf2988811c922a276b493ab3592d6d6 %}

Even though if we compile and run right now, it seems to work correctly, as soon as we make a request to our groups controller, we get the following error:
```
An unhandled exception occurred while processing the request.

InvalidOperationException: Unable to resolve service for type 'CodingMilitia.PlayBall.GroupManagement.Web.Demo.IGroupIdGenerator' while attempting to activate 'CodingMilitia.PlayBall.GroupManagement.Web.Controllers.GroupsController'.
```

That's because we prepared everything, but didn't register the group id generator in the dependency injection container. When registering a service in the container, one decision we need to make is what's the life cycle of the instances created by the DI container:

- Transient - every time a dependency on the registered service is needed, a new instance is created
- Scoped - every time a dependency on the registered service is needed in a request, the same instance is used, but across different requests the instance is different
- Singleton - every time a dependency on the registered service is needed, the same instance is used

The life cycle will always depend on the service we're registering. In our case, given that we're storing the last generated id in an instance field of `GroupIdGenerator`, and we want it to last at least as long as the application is running (given we don't have persistence yet), we'll use a singleton.

Our `ConfigureServices` end result will be as follows:

{% gist be37997078286174a894793dc1f22f3e %}

Now if we run it, it works as it was working before, just with a different way to getting the next id when creating a new group.

## Outro

These are the basics of the Program and Startup classes. There's of course much more that can be said and done, but I feel this is enough as an intro as we prepare to go a little bit deeper in the next episodes.

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode004).

Please send any feedback you have, so the next posts/videos can be better and even adjusted to more interesting topics.

Thanks for stopping by, cyaz!