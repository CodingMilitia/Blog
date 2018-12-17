---
author: johnny
date: 2018-12-05 18:30:00+00:00
layout: post
title: "Episode 008 - Middlewares - ASP.NET Core: From 0 to overkill"
summary: "In this episode we take a look at ASP.NET Core's middlewares, how to create them and use them in the request handling pipeline."
image: '/assets/2018/12/05/aspnet-core-from-zero-to-overkill-e008.jpg'
categories:
- fromzerotooverkill
tags:
- dotnet
- aspnetcore
---

In this episode we take a look at ASP.NET Core's middlewares, how to create them and use them in the request handling pipeline.
For the walk-through you can check the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/YU28TJWARg0" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />
## Intro
We had a brief introduction to middlewares in the fourth episode, when we talked about the `Program` and `Startup` classes. In this post we’ll be revisiting the middleware topic, and going through the options we have to build middlewares.

As we’ve seen, the basic idea of middlewares is that they work like in a chain - a middleware is invoked, it can do something, then pass the request along to the next middleware and finally do something else when the next middleware is done. There can be cases where the next middleware isn't invoked, for instance in a authorization middleware, if the user isn't authenticated, it can immediately stop the request, returning a response by itself.

The following is a good image to illustrate the behavior, from the official [docs](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/index?view=aspnetcore-2.1).

![request handling pipeline](/assets/2018/12/05/request-delegate-pipeline.png)

Middlewares can range from simple things just to enrich the request handling pipeline - adding some headers, filter unauthorized access, collect some metrics, etc - to implementing a complete request handling logic - an endpoint for health checks, serving static files, MVC 🙂, etc. 

To create middlewares, we have a couple of options to get started:
- Implement the logic in an inline anonymous function (like the example we saw in episode 4)
- Implement the logic in a class dedicated to it

In this post, we'll take a look at some simple examples of implementing middlewares with both of these options.

## app.Use
Given we already saw a simple middleware implemented and registered in the pipeline with `app.Use`, won't be going into much more detail than adding the code here for reference 🙂

`Startup.cs`
```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    //...
    
    app.Use(async (context, next) =>
    {
        context.Response.OnStarting(() =>
        {
            context.Response.Headers.Add("X-Powered-By", "ASP.NET Core: From 0 to overkill");
            return Task.CompletedTask;
        });

        await next.Invoke();
    });

    //...
}
```
As we can see, this registers an anonymous function that receives the `HttpContext` and a `Func` to invoke the next middleware in the chain. Then the implementation, well, we do whatever we want, in this case it's just adding a response header, while letting the rest of the pipeline work as normal, by invoking the next middleware.

Note that the order in which we register the middleware (during the `Configure` method), defines the order in which they'll be run when handling a request, so **order is not irrelevant**.

## app.UseMiddleware<T>
With `app.Use`, we register a lambda based middleware. To register a class based middleware we can use `app.UseMiddleware<T>` extension method.

When creating class based middlewares, we have a couple of options:
- Convention-based activated - where we follow a convention to implement the class, and all will work kinda automagically
- Factory-based activated - the middleware class implements `IMiddleware`, so we get a more "normal" experience, like when we implement a controller inheriting from the `Controller` class (which, by the way, is also not mandatory, we can also implement controllers by following some conventions)

### Convention-based activated middleware
For a convention-based activated middleware, we should follow a couple of rules and we're good to go. A basic implementation would be something like the following.

```csharp
public class SampleConventionActivatedMiddleware
{
    private readonly RequestDelegate _next;

    public SampleConventionActivatedMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // do stuff before the next middleware executes
        await _next(context);
        // do stuff after the next middleware executes
    }
}
```

The constructor receives a delegate to invoke the next middleware in the chain, and we're storing it to use it when handling the requests.

Then we have an `InvokeAsync` method that gets the `HttpContext` as an argument. This might already tells us something, that's probably not completely clear at first look - the instance of the middleware used in the pipeline is basically a singleton, as it's constructed at application startup.

Let's take a look at a middleware with a little more going on.

```csharp
public class RequestTimingAdHocMiddleware
{
    private readonly RequestDelegate _next;
    private int _requestCounter; 

    public RequestTimingAdHocMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context, ILogger<RequestTimingAdHocMiddleware> logger)
    {
        var watch = Stopwatch.StartNew();
        await _next(context);
        watch.Stop();
        Interlocked.Increment(ref _requestCounter);
        logger.LogWarning("Request {requestNumber} took {requestTime}ms", _requestCounter, watch.ElapsedMilliseconds);
    }
}
```
As we saw in the previous example, the middleware instance is a singleton, so if we have some dependencies, instead of getting them in the constructor as usual, we declare what we need in the `InvokeAsync` method, so they can be resolved per request.

With all this in mind, what happens in this case is the `_requestCounter` field will have a count of all the requests that went through the middleware during the application lifetime, because it's a singleton. Additionally, we get a logger as dependency injected on `InvokeAsync`.

Sample log output:
```
11:38:09 CodingMilitia.PlayBall.GroupManagement.Web.Demo.Middlewares.RequestTimingAdHocMiddleware Warn Request 1 took 1ms
11:38:15 CodingMilitia.PlayBall.GroupManagement.Web.Demo.Middlewares.RequestTimingAdHocMiddleware Warn Request 2 took 0ms
11:38:19 CodingMilitia.PlayBall.GroupManagement.Web.Demo.Middlewares.RequestTimingAdHocMiddleware Warn Request 3 took 0ms
```

For all of this to work, of course we need to register the middleware in the request handling pipeline:

`Startup.cs`
```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    //...

    app.UseMiddleware<RequestTimingAdHocMiddleware>();

    //...
}
```

### Factory-based activated middleware
Another option for class based middlewares is to implement the `IMiddleware` interface. Continuing with the example we used for the convention-based approach, we get:

```csharp
public class RequestTimingFactoryMiddleware : IMiddleware
{
    private readonly ILogger<RequestTimingFactoryMiddleware> _logger;
    private int _requestCounter; 

    public RequestTimingFactoryMiddleware(ILogger<RequestTimingFactoryMiddleware> logger)
    {
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var watch = Stopwatch.StartNew();
        await next(context);
        watch.Stop();
        Interlocked.Increment(ref _requestCounter);
        _logger.LogWarning("Request {requestNumber} took {requestTime}ms", _requestCounter, watch.ElapsedMilliseconds);
    }
}
```

Right off the bat, we can see a more traditional class layout, with a dependency injected in the constructor and the `InvokeAsync` method to implement the interface.

In this type of middleware, the lifetime is up to us, as we need to register it in DI, and not only in the request handling pipeline like we did for the convention-based middleware.

`Startup.cs`
```csharp
public class Startup
{
    //...
    
    public void ConfigureServices(IServiceCollection services)
    {
        //...

        services.AddTransient<RequestTimingFactoryMiddleware>();

        //...
    }

    //...

    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        //...

        app.UseMiddleware<RequestTimingFactoryMiddleware>();

        //...
    }
}
```
In this case, because the class is being registered as transient, the request counter will not persist across requests, so we get the following log output:

```
11:52:06 CodingMilitia.PlayBall.GroupManagement.Web.Demo.Middlewares.RequestTimingFactoryMiddleware Warn Request 1 took 0ms
11:52:07 CodingMilitia.PlayBall.GroupManagement.Web.Demo.Middlewares.RequestTimingFactoryMiddleware Warn Request 1 took 0ms
11:52:08 CodingMilitia.PlayBall.GroupManagement.Web.Demo.Middlewares.RequestTimingFactoryMiddleware Warn Request 1 took 0ms
```

## app.Run
Back to using anonymous functions as middlewares, besides the `app.Use` extension method, we have `app.Run`. It is basically the same, but to register a function as a terminal middleware, which means it won't flow the request through to any middleware that comes after it in the chain. It's basically the same as `app.Use` without ever calling the `next` argument 😛

Just for examples sake, in the request handling pipeline configuration, we can add something at the end:

`Startup.cs`
```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    //...
    
    app.Run(async (context) =>
    {
        await context.Response.WriteAsync("No middlewares could handle the request");
    });    
}
```
In this case, if no middleware is capable of completely handle the request, particularly the MVC middleware, we send out a (pretty barebones) response informing of this fact.

## app.Map and app.MapWhen
Another interesting couple of extension methods we have available are `Map` and `MapWhen`. These methods allow us to create branches in the request handling pipeline given some conditions.

Let's take a look at some examples.

### app.Map
`Map` allows us to branch the pipeline for a given path.

`Startup.cs`
```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    //...
    
    app.Map("/ping", builder =>
    {
        builder.UseMiddleware<RequestTimingFactoryMiddleware>();
        builder.Run(async (context) => { await context.Response.WriteAsync("pong from path"); });
    }); 

    //... 
}
```

Taking the example above, for any request to `/ping`, instead of following the usual pipeline, the middlewares configured in the lambda argument will be used, do we get the previously developed `RequestTimingFactoryMiddleware` executing, plus a terminal middleware responding with a simple message.

### app.MapWhen
`MapWhen` works in a very similar fashion to `Map`, but instead of using forcibly a path, we can define other conditions. To define the condition, we provide a function that gets an `HttpContext` argument and returns a boolean, indicating if the branch should be used.

`Startup.cs`
```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    //...
    
    app.MapWhen(
        context => context.Request.Headers.ContainsKey("ping"),
        builder =>
        {
            builder.UseMiddleware<RequestTimingAdHocMiddleware>();
            builder.Run(async (context) => { await context.Response.WriteAsync("pong from header"); });
        });

    //... 
}
```

This example is pretty similar to the previous one, but using as condition a check to see if there's a request header named `ping`. If the header exists, than the branch is used and, in this case,  the previously implemented `RequestTimingAdHocMiddleware` is used, plus another terminal middleware to output a simple response message.

## Outro
That does it for a quick look at implementing middlewares. It's simple enough, and I think there isn't much more to it. To build more complex middlewares, we just need to add more logic upon these initial steps we followed to set them up.

Like I said in other posts in this series, the official [docs](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-2.1) will most likely have much more information when compared to these rather quick walk-throughs I'm putting out 😉.

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode008).

Please send any feedback so I can improve and adjust the next episodes.

Thanks for stopping by, cyaz!