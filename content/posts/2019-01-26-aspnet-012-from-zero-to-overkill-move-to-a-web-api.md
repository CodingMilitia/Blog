---
author: João Antunes
date: 2019-01-26 17:30:00+00:00
layout: post
title: "Episode 012 - Move to a Web API - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we start transforming the current application into a Web API, so it can be used in the single page application we'll be developing."
images:
- '/assets/2019/01/26/aspnet-core-from-zero-to-overkill-e012.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-012-from-zero-to-overkill-move-to-a-web-api
---

In this episode, we start transforming the current application into a Web API, so it can be used in the single page application we'll be developing.

For the walk-through you can check the next video, but if you prefer a quick read, skip to the written synthesis.

{{< yt vxDvAGP9Bh8 >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
So far we've been building a server side rendered MVC application. The goal of this project however, is to end up with a single page application that handles most of the UI needs. To achieve this, we will transition this MVC application into a Web API, so it and other components we build in the future can be used by the SPA (or other components that may require it).

Also, a note on me calling this a Web API or an HTTP API and not a REST API. I don't want to annoy anyone by calling it a REST API while not implementing it following all the requirements that implies, including, but not only, hypermedia. Let's say it's a RESTish API (😛), as it implements what most expect from it, but is not fully compliant to be called REST.

## Remove unneeded bits
Since we're moving to a Web API, there are some things that we have no more need and can cleanup/adjust namely:
- Remove views
- Replace `AddMvc` with `AddMvcCore`, adding only needed MVC features
- Inherit from `ControllerBase` instead of `Controller`

Regarding the first one, there's not much to say, just delete the folder with the views as we won't be needing them anymore. Regarding the others, let's get into more detail.

### Replace AddMvc with AddMvcCore
`AddMvc` registers in the DI container MVC's services, the ones it surely requires and some that it might. To do this it calls `AddMvcCore` and adds some more things. We can avoid registering unneeded services by calling `AddMvcCore` directly, plus any other MVC services we know we need.

Will this have an impact on performance? Probably not that much (didn't measure it though), but since we're at it, we remove some unused bits and take a better look at some of the available MVC features. Most of the time though, keep `AddMvc` and don't worry about it 🙂

Let's take a look at the [`MvcServiceCollectionExtensions.cs`](https://github.com/aspnet/AspNetCore/blob/v2.2.1/src/Mvc/src/Microsoft.AspNetCore.Mvc/MvcServiceCollectionExtensions.cs) file on GitHub, where `AddMvc` is defined.

```csharp
// ...

public static class MvcServiceCollectionExtensions
{
    /// <summary>
    /// Adds MVC services to the specified <see cref="IServiceCollection" />.
    /// </summary>
    /// <param name="services">The <see cref="IServiceCollection" /> to add services to.</param>
    /// <returns>An <see cref="IMvcBuilder"/> that can be used to further configure the MVC services.</returns>
    public static IMvcBuilder AddMvc(this IServiceCollection services)
    {
        // ...

        var builder = services.AddMvcCore();

        builder.AddApiExplorer();
        builder.AddAuthorization();

        AddDefaultFrameworkParts(builder.PartManager);

        // Order added affects options setup order

        // Default framework order
        builder.AddFormatterMappings();
        builder.AddViews();
        builder.AddRazorViewEngine();
        builder.AddRazorPages();
        builder.AddCacheTagHelper();

        // +1 order
        builder.AddDataAnnotations(); // +1 order

        // +10 order
        builder.AddJsonFormatters();

        builder.AddCors();

        return new MvcBuilder(builder.Services, builder.PartManager);
    }

    private static void AddDefaultFrameworkParts(ApplicationPartManager partManager)
    {
        var mvcTagHelpersAssembly = typeof(InputTagHelper).GetTypeInfo().Assembly;
        if (!partManager.ApplicationParts.OfType<AssemblyPart>().Any(p => p.Assembly == mvcTagHelpersAssembly))
        {
            partManager.ApplicationParts.Add(new FrameworkAssemblyPart(mvcTagHelpersAssembly));
        }

        var mvcRazorAssembly = typeof(UrlResolutionTagHelper).GetTypeInfo().Assembly;
        if (!partManager.ApplicationParts.OfType<AssemblyPart>().Any(p => p.Assembly == mvcRazorAssembly))
        {
            partManager.ApplicationParts.Add(new FrameworkAssemblyPart(mvcRazorAssembly));
        }
    }
    
    // ...
}
```

I cleaned up some parts of the file that aren't really needed for what we're looking for right now.

Looking at `AddMvc`, we see that the first thing it does is calling `AddMvcCore`, storing the returned `IMvcBuilder` in a variable for further configuration, so that's something we'll also need to do. Now let's look at the other things that are configured using the builder.

- AddApiExplorer - used to expose information about the MVC application. It's useful, for instance, for creating Swagger documentation endpoints. For more info check out [Andrew Lock's post](https://andrewlock.net/introduction-to-the-apiexplorer-in-asp-net-core/). We'll use Swagger eventually, but since we aren't using it yet... out with it!
- AddAuthorization - adds the necessary authorization services that we'll certainly need in the future, but not right now, so... out!
- AddDefaultFrameworkParts - looking at `AddDefaultFrameworkParts` implementation below `AddMvc`, we can see it's registering services from Razor and TagHelpers assemblies, so we can also bypass this one.
- AddFormatterMappings - I had to take a look at the implementation of this one to figure out what it does, and it basically registers a `FormatFilter` that checks the request's route data and query string for the presence of a format argument, to be used like the `Accept` header is used. Another one that we're not going to need, so skip it.
- AddViews - views... next!
- AddRazorViewEngine - more Razor stuff, also next!
- AddRazorPages - and more Razor... let's keep on skipping.
- AddCacheTagHelper - more TagHelpers, skip it.
- AddDataAnnotations - we're not using data annotations so far, and I'm also not expecting to use them in this API in the future (we'll use [Fluent Validation](https://github.com/JeremySkinner/FluentValidation)), so we can also safely ignore it.
- AddJsonFormatters - now this we need, as our API will need to handle JSON.
- AddCors - I'm not expecting we'll be accessing the API from different domains, so we can safely ignore this one as well.

So, what do we end up with? In the `Startup` class the `services.AddMvc();` is replaced by `services.AddRequiredMvcComponents();`, a new extension method we add to the `ServiceCollectionExtensions` class we already created. In this file we add the following:

`ServiceCollectionExtensions.cs`
```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddRequiredMvcComponents(this IServiceCollection services)
    {
        var mvcBuilder = services.AddMvcCore();
        mvcBuilder.AddJsonFormatters();
        return services;
    }

    // ...
}
```

Considering what we had, we only replaced the `AddMvc` with `AddMvcCore` and also added a call to `AddJsonFormatters`. We'll get back to this method later, as we'll need some extra configurations for the changes we'll be making.

### Inherit from ControllerBase instead of Controller
Our `GroupsController` class so far inherited from `Controller`. It can still inherit from `Controller`, but since the only thing `Controller` has that `ControllerBase` doesn't (from which it inherits by the way) is support for views, we can skip the unnecessary extra bits.

Because of this change, since we are still calling the `View` method, we'll get compilation errors. We'll get right to it.

## Adjust endpoints
Now let's rework our `GroupsController` into an API controller. In summary, it'll end up with methods with the signatures pretty much the same as the `IGroupsService`, calling the service's methods and adding some extra "controller things" into the mix, namely, routing, HTTP methods and HTTP responses.

`GroupsController.cs`
```csharp
[Route("groups")]
public class GroupsController : ControllerBase
{
    private readonly IGroupsService _groupsService;

    public GroupsController(IGroupsService groupsService)
    {
        _groupsService = groupsService;
    }

    [HttpGet]
    [Route("")]
    public async Task<IActionResult> GetAllAsync(CancellationToken ct)
    {
        var result = await _groupsService.GetAllAsync(ct);
        return Ok(result.ToModel());
    }


    [HttpGet]
    [Route("{id}")]
    public async Task<IActionResult> GetByIdAsync(long id, CancellationToken ct)
    {
        var group = await _groupsService.GetByIdAsync(id, ct);

        if (group == null)
        {
            return NotFound();
        }

        return Ok(group.ToModel());
    }

    [HttpPut]
    [Route("{id}")]
    public async Task<IActionResult> UpdateAsync(long id, GroupModel model, CancellationToken ct)
    {
        model.Id = id; //not needed when we move to MediatR
        var group = await _groupsService.UpdateAsync(model.ToServiceModel(), ct);
        
        return Ok(group.ToModel());
    }

    [HttpPut]
    [HttpPost]
    [Route("")]
    public async Task<IActionResult> AddAsync(GroupModel model, CancellationToken ct)
    {
        model.Id = 0; //not needed when we move to MediatR
        var group = await _groupsService.AddAsync(model.ToServiceModel(), ct);

        return CreatedAtAction(nameof(GetByIdAsync), new { id = group.Id }, group.ToModel());
    }

    [HttpDelete]
    [Route("{id}")]
    public async Task<IActionResult> RemoveAsync(long id, CancellationToken ct)
    {
        await _groupsService.RemoveAsync(id, ct);

        return NoContent();
    }
}
```

I think it's really straightforward to understand what's going on here, but I'll try to point out some small details.

- The read actions, including their routes remain basically the same, with some method name adjustments and returning the models directly instead of views (JSON or any other format serialization is handled by the framework).

- The update action is bound to HTTP PUT, the usual in HTTP APIs, as we're expecting full replacement of the stored entity.

- The add action is bound to both HTTP POST and PUT methods, differing from the update action by not expecting an id. This is of course dependent on this implementation, that generates an id automatically. Other implementations might allow for the client to provide an id, in which case we would probably replace these two actions with a single one that would add if the entity doesn't exist, update otherwise.

- The add action returns an HTTP 201 Created status code, including the `Location` header to the url to fetch the added entity, but also including it in the response body.

On a final note in this section, notice that in the add and update methods I'm overriding the ids that come from the client. I shouldn't need to be doing this, but am doing to ensure the service doesn't get inconsistent/unexpected ids - defining an id on the add that should have no id yet (at least in the way we're doing it right now, allowing the id to be defined could also be valid) and a different id than what was put on the route in the case of the update.

The problem mainly stems from the fact we're reusing the same model for all operations. The best way to do it would be to use specific models for each, which we'll get to when we add [MediatR](https://github.com/jbogard/MediatR) to the project.

## Use ApiController attribute
The API is mostly ready, but if we try to run it now, the read actions will work as expected but not the writing ones.

Lets make an attempt at adding a new group, by POSTing to `http://localhost:5000/groups` the info `{ "name": "Test Group" }`. The response we get is the following:

```json
{
    "id": 1,
    "name": null,
    "rowVersion": "611"
}
```
The `id` and the `rowVersion` are correct, but not the `name`. The former are good because they are generated server side, but the name is not getting to the action method. If we put a break point in the `AddAsync` method we'll see that the `model` argument is empty. Same is happening with `UpdateAsync`.

The problem is that the actions don't know that the `model` should be deserialized from the body. We can easily solve this by adding the `[FromBody]` attribute to the `model` argument.

This is acceptable, but we can do better. In ASP.NET Core 2.1 the `ApiController` was introduced, allowing us to decorate a controller, making it follow some conventions that save us work, like inferring that complex objects come from the request body. This can be overridden of course, but it's a good default to avoid us some typing. You can read more about this attribute [here](https://docs.microsoft.com/en-us/aspnet/core/web-api/?view=aspnetcore-2.2#annotation-with-apicontroller-attribute).

To use this attribute we must go to our dependency injection MVC configuration and add the following:

`ServiceCollectionExtensions.cs`
```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddRequiredMvcComponents(this IServiceCollection services)
    {
        var mvcBuilder = services.AddMvcCore();
        mvcBuilder.SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
        // ...
    }

    // ...
}
```

Because using this attribute requires the use of new ASP.NET Core MVC bits from version 2.1 and up, which might cause breaking changes when compared to 2.0, we must explicitly tell it we want to use the new features. Read more about compatibility versions [here](https://docs.microsoft.com/en-us/aspnet/core/mvc/compatibility-version?view=aspnetcore-2.2).

I'm using compatibility version 2.2 because in this one we can use the `ApiController` attribute as an assembly attribute that is applied automatically to all controllers, instead of having to decorate each of them.

In the `Startup` class file I just added the attribute as `[assembly: ApiController]`, and we now have a correctly working controller 🙂

## Create an exception filter
Building on what we learned in past episodes, we can create an `ExceptionFilter`. In this case we'll create a filter that returns an HTTP 409 Conflict status code when an update to a group fails due to it being outdated - an optimistic concurrency exception like we saw in the last episode. We can improve the filter later to handle more kinds of errors.

`ApiExceptionFilter.cs`
```csharp
public class ApiExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        if (context.Exception is DbUpdateConcurrencyException)
        {
            context.Result = 
                new ConflictObjectResult(
                    new
                    {
                        Message = "The updated entity has changed, please refresh your current copy."
                    });
        }
    }
}
```

We can improve this in the future, by making the business layer abstract the exception that reaches the API to a more generic concurrency exception, instead of being tied to an Entity Framework specific exception. For now, it's more than good enough.

Now we go back to our `ServiceCollectionExtensions` class and wrap-up the changes (for today) to MVC's configuration, by adding this filter to it.

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddRequiredMvcComponents(this IServiceCollection services)
    {
        services.AddTransient<ApiExceptionFilter>();

        var mvcBuilder = services.AddMvcCore(options =>
        {
            options.Filters.AddService<ApiExceptionFilter>();
        });
        mvcBuilder.SetCompatibilityVersion(CompatibilityVersion.Version_2_2);   
        mvcBuilder.AddJsonFormatters();
        return services;
    }
    
    // ...
}
```

## Outro
That's about it for this post. We migrated our MVC application to a Web API - which is build on top of MVC anyway, but you get the point!

In future episodes we will build upon this API, which is still very simple but has served us well so far, and we were able to explore a lot of ASP.NET Core building blocks and features.

In the next episode though, we'll take a break from ASP.NET Core and start creating a single page application with Vue.js, which will be our PlayBall project frontend, and will use this API to implement its features.

Links in the post:
- Andrew Lock's [Introduction to the ApiExplorer in ASP.NET Core](https://andrewlock.net/introduction-to-the-apiexplorer-in-asp-net-core/)
- [Build web APIs with ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/web-api/?view=aspnetcore-2.2)
- [Compatibility version for ASP.NET Core MVC](https://docs.microsoft.com/en-us/aspnet/core/mvc/compatibility-version?view=aspnetcore-2.2)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode012).

If you want to see the changes from the previous code to the changes of the episode, don't forget you can take a look at the commits and pull requests in GitHub, for instance, you can see the PR for this episode [here](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/pull/5).

Feel free to ask any questions and don't hesitate to provide feedback.

Thanks for stopping by, cyaz!