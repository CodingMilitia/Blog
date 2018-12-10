---
author: johnny
date: 2018-12-10 17:25:00+01:00
layout: post
title: "Episode 009 - MVC filters - ASP.NET Core: From 0 to overkill"
summary: "Following up on the previous episode on ASP.NET Core middlewares, in this episode we take a look at MVC's filters, an MVC specific way to add behaviors to our request handling pipeline, and how we can use them to implement cross-cutting concerns in our web applications."
image: '/assets/2018/12/10/aspnet-core-from-zero-to-overkill-e009.jpg'
categories:
- fromzerotooverkill
tags:
- dotnet
- aspnetcore
---

Following up on the previous episode on ASP.NET Core middlewares, in this episode we take a look at MVC's filters, an MVC specific way to add behaviors to our request handling pipeline, and how we can use them to implement cross-cutting concerns in our web applications.
For the walk-through you can check the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/Dms0HcPAEcY" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
Like introduced, given the previous episode focused on ASP.NET Core's middlewares, taking a look at MVC filters right after it, makes sense to me, as it provides some different options to add behaviors to the request handling pipeline, even if in this case more specific to MVC features.

Going back to the [docs](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/filters?view=aspnetcore-2.1) for an image that illustrates how the filters fit in the overall picture.

![filter pipeline overview](/assets/2018/12/10/filter-pipeline-overview.png)

There are 5 types of filters, as seen in the [docs](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/filters?view=aspnetcore-2.1):

> - Authorization filters run first and are used to determine whether the current user is authorized for the current request. They can short-circuit the pipeline if a request is unauthorized.
> - Resource filters are the first to handle a request after authorization. They can run code before the rest of the filter pipeline, and after the rest of the pipeline has completed. They're useful to implement caching or otherwise short-circuit the filter pipeline for performance reasons. They run before model binding, so they can influence model binding.
> - Action filters can run code immediately before and after an individual action method is called. They can be used to manipulate the arguments passed into an action and the result returned from the action.
> - Exception filters are used to apply global policies to unhandled exceptions that occur before anything has been written to the response body.
> - Result filters can run code immediately before and after the execution of individual action results. They run only when the action method has executed successfully. They are useful for logic that must surround view or formatter execution.

The following image (from the [docs](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/filters?view=aspnetcore-2.1) again) shows some more details of how the filters play together. 

![filter pipeline closeup](/assets/2018/12/10/filter-pipeline-closeup.png)

In this post we'll only use action and exception filters, as well as the options we have on how to implement and use them. The other filters should be similar in terms of implementation and usage, having of course different reasons to go with them.

## Application wide filter
Let's start simple, with an action filter registered to intercept all requests that find their way to an MVC action.

To do this, we start by creating a class that implements `IActionFilter` (or `IAsyncActionFilter` if we need to do some async work on there).

`DemoActionFilter.cs`
```csharp
public class DemoActionFilter : IActionFilter
{
    private readonly ILogger<DemoActionFilter> _logger;

    public DemoActionFilter(ILogger<DemoActionFilter> logger)
    {
        _logger = logger;
    }

    public void OnActionExecuting(ActionExecutingContext context)
    {
        _logger.LogInformation("Before executing action {action} with arguments \"{@arguments}\" and model state \"{@modelState}\"",
            context.ActionDescriptor.DisplayName,
            context.ActionArguments,
            context.ModelState);
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        _logger.LogInformation("After executing action {action}.", context.ActionDescriptor.DisplayName);
    }
}
```

Rather straightforward stuff. We implement the interface methods `OnActionExecuting` and `OnActionExecuted` (which run before and after the action executes respectively). 

As usual in my post examples, I'm logging stuff 🤣 The content is not that important I would say, it's just serving as an example of some of the information we have access to in the context of the filter.

We can also see that we can get dependencies injected, as we're getting a logger in the constructor.

To register the filter, so it intercepts all the actions, we head on to the `Startup` class `ConfigureServices` method, and register the filter in MVC options as follows:

`Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc(options =>
    {
        options.Filters.Add<DemoActionFilter>();
    });

    //...
}
```

Another alternative (and probably a better idea) would be to use `options.Filters.AddService<DemoActionFilter>()` to register the filter, but if we do this, we must also register the filter in the DI container.

A sample log output for this filter would be as follows:

```
19:05:24 CodingMilitia.PlayBall.GroupManagement.Web.Demo.Filters.DemoActionFilter Info Before executing action "CodingMilitia.PlayBall.GroupManagement.Web.Controllers.GroupsController.CreateReally (CodingMilitia.PlayBall.GroupManagement.Web)" with arguments "{"model":{"Id":0, "Name":"Some Group"}}" and model state "[{"Key":"Name", "Value":{"Key":"Name", "SubKey":{"Buffer":"Name", "Offset":0, "Length":4, "Value":"Name", "HasValue":true}, "IsContainerNode":false, "RawValue":"Some Group", "AttemptedValue":"Some Group", "Errors":[], "ValidationState":"Valid"}}]"
19:05:24 CodingMilitia.PlayBall.GroupManagement.Web.Demo.Filters.DemoActionFilter Info After executing action CodingMilitia.PlayBall.GroupManagement.Web.Controllers.GroupsController.CreateReally (CodingMilitia.PlayBall.GroupManagement.Web).
```

## Decorating a controller or action with a filter attribute
Having a filter applied globally is nice, and may suffice for a great amount of cases, but sometimes we really need more control over when the filter should in fact execute. A good way to achieve this is by using attributes, with which we can decorate a controller or an action where we want the filter to be used.

Let's make another silly sample action filter to check this out 🙂

`DemoActionFilterAttribute.cs`
```csharp
public class DemoActionFilterAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (context.ActionArguments.TryGetValue("model", out var model)
            && model is GroupViewModel group
            && group.Id == 1)
        {
            group.Name += $" (Added on {nameof(DemoActionFilterAttribute)})";
        }
    }
}
```

In this example, we're checking if there is an action argument named `model` and then if it is of type `GroupViewModel` with id `1`. If it is a match, we alter the contents of the object, just to show we can 🙂

You might notice we don't have any dependencies being injected, and that's because by rolling an attribute like this, we can't have them, because we would need to pass them when applying the attribute, which isn't really doable (but we'll see in a bit how we can have a filter applied using an attribute that is able to have dependencies injected).

To apply the attribute, we can go into our `GroupsController` and apply it to the class directly, or to any method - but to see it working we need to apply it to the `Edit` method, the others will not make use of it.

`GroupsController.cs`
```csharp
[DemoActionFilter]
[HttpPost]
[Route("{id}")]
[ValidateAntiForgeryToken]
public IActionResult Edit(long id, GroupViewModel model)
{
    var group = _groupsService.Update(model.ToServiceModel());

    if (group == null)
    {
        return NotFound();
    }

    return RedirectToAction("Index");
}
```

To see the end result, we can create a group, then edit it, and we'll see the added text.

## Filter attribute with dependencies
Applying a filter using an attribute is nice, but as mentioned, doing it like shown in the previous section doesn't allow us to do much, as we can't get any dependencies in the filter class.

Let's look at some options to have the cake and eat it too. Let's start with creating a sample exception filter.

`DemoExceptionFilter.cs`
```csharp
public class DemoExceptionFilter : IExceptionFilter
{
    private readonly ILogger<DemoExceptionFilter> _logger;

    public DemoExceptionFilter(ILogger<DemoExceptionFilter> logger)
    {
        _logger = logger;
    }

    public void OnException(ExceptionContext context)
    {
        if (context.Exception is ArgumentException)
        {
            _logger.LogError("Transforming ArgumentException in 400");
            context.Result = new BadRequestResult();
        }
    }
}
```

Simple stuff, any time an exception is thrown (and not caught) in an action, it'll end up in the `DemoExceptionFilter`, and if it's an `ArgumentException` we respond with a 400. Now let's use the filter.

### Using ServiceFilterAttribute
The first option we have is to use the `ServiceFilterAttribute`. Instead of applying a filter as attribute directly, we apply `ServiceFilterAttribute` with the type of filter we want as an argument.

`GroupsController.cs`
```csharp
[ServiceFilter(typeof(DemoExceptionFilter))]
[Route("groups")]
public class GroupsController : Controller
{
    //...
}
```

Besides adding the attribute, we also need to register the `DemoExceptionFilter` in DI, so the `ServiceFilterAttribute` can fetch it.

`Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    //...

    services.AddTransient<DemoExceptionFilter>();

    //...
}
```

### Using a custom filter factory
An alternative to `ServiceFilterAttribute`, if we require more control over things, is to create an attribute that implements `IFilterFactory`. 

`DemoExceptionFilterFactoryAttribute.cs`
```csharp
public class DemoExceptionFilterFactoryAttribute : Attribute, IFilterFactory
{
    public IFilterMetadata CreateInstance(IServiceProvider serviceProvider)
    {
        var filter = serviceProvider.GetRequiredService<DemoExceptionFilter>();
        return filter;
    }

    public bool IsReusable { get; } = false;
}
```

When implementing `IFilterFactory.CreateInstance` we get an `IServiceProvider` instance as argument, so we can fetch anything we need from the dependency injection container. In this case we're simply getting a filter instance from DI, but we could also complicate things if needed.

The `IsReusable` property is usd to tell the runtime if the filter instances returned by the factory can be reused across requests. If it was a singleton filter, sure, but when it's not the case, `IsReusable` should be `false`.

To use it, we can simply go into `GroupsController` and replace the `ServiceFilterAttribute` with this one.

`GroupsController.cs`
```csharp
[DemoExceptionFilterFactory]
[Route("groups")]
public class GroupsController : Controller
{
    //...
}
```

## Outro
Like I said in the beginning, this is just a quick look at some of the stuff we can do it MVC filters, just so we are aware of our options when developing an application, maybe we recognize that some pattern would map perfectly to an MVC filter (or maybe a middleware like we saw in the previous post).

As always, the [docs](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-2.1) are a great place to learn more about all these topics, with lots of info I'm not able to cram into these quick posts.

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode009).

Please send any feedback so I can improve and adjust the next episodes.

Thanks for stopping by, cyaz!