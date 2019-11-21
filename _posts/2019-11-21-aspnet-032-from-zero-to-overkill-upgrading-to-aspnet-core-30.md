---
author: João Antunes
date: 2019-11-21 18:30:00+01:00
layout: post
title: "Episode 032 - Upgrading to ASP.NET Core 3.0 - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we'll go through the main changes required to upgrade our currently ASP.NET Core 2.2 applications to ASP.NET Core 3.0."
image: '/assets/2019/11/21/aspnet-core-from-zero-to-overkill-e032.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
---

In this episode, we'll go through the main changes required to upgrade our currently ASP.NET Core 2.2 applications to ASP.NET Core 3.0.

For the walk-through you can check out the next video, but if you prefer reading, skip to the written synthesis.

{% youtube "https://youtu.be/pPAMeuoPsUE" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

I guess it's time for a boring episode, as .NET Core 3.0 came out recently, we need to get our projects updated to the latest version.

There aren't too many required changes, but being a new major version, there are a couple of breaking changes and some new recommended practices.

I probably won't go through every change in this post, so if you want to check out all the details, take a look at the diff in the GitHub repositories.

## Basics and other quick stuff

Let's begin with the simplest bits.

The most obvious is of course going through all the `csproj` files and bumping up the .NET Core version, by replacing the target framework, currently `netcoreapp2.2` with `netcoreapp3.0`.

The second most obvious change is upgrading the NuGet packages, namely the ones from Microsoft that have their versions tied to the framework version, but even others (e.g. IdentityServer4) sometimes keep their versions paired with the framework's. In this case, for simplicity, I just used the IDE and updated all the packages to their latest stable version.

Other quick changes were:

- `SetCompatibilityVersion` method, which we called when setting up MVC is no longer needed, at least with 3.0, maybe it'll be back in 3.1.
- In `GroupEntityConfiguration`, when setting the auto increment column, `UseNpgsqlIdentityAlwaysColumn` is deprecated in favor of `UseIdentityAlwaysColumn`.
- `IHostingEnvironment`, used for instance in the `Startup` class is deprecated, now replaced by `IHostEnvironment`. If specific web applications bits are needed, the alternative is `IWebHostEnvironment` (which inherits from `IHostEnvironment`).
- We can remove the reference to the metapackage `Microsoft.AspNetCore.App`, because it's implicit when the project SDK is `Microsoft.NET.Sdk.Web`, as is the case. If we weren't using this SDK, we should move this reference to a `FrameworkReference` instead of a `PackageReference`. For more info, go [here](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30#framework-reference).
- Entity Framework Core 3.0 now targets .NET Standard 2.1 instead of 2.0 (though this will be [reverted in 3.1](https://github.com/aspnet/EntityFrameworkCore/issues/18141)).

## Packages extracted from the ASP.NET Core shared framework

With this release, some packages that were part of ASP.NET Core shared framework (the things we got when referencing `Microsoft.AspNetCore.App`), so we need to reference the ones we're using.

The main package in this situation is Entity Framework Core. The main reasons for this is change seem to be to make the process of targeting EF Core the same whether it's an ASP.NET Core app or not, as well as not forcing the upgrade of EF Core version when upgrading the framework. More info in the migration docs [here](https://docs.microsoft.com/en-us/ef/core/what-is-new/ef-core-3.0/breaking-changes#no-longer).

Other extracted packages that we must now explicitly reference are `Microsoft.AspNetCore.Authentication.OpenIdConnect` and `Microsoft.AspNetCore.Authentication.JwtBearer`.

## IWebHost → IHost

Previously we were using `IWebHost` (along with `IWebHostBuilder`) to setup the host of our web applications. When we wanted to build other kinds of services, for instance a message queue consumer, we would use the more generic `IHost` and its accompanying `IHostBuilder` (we'll eventually implement such services in this series).

With ASP.NET Core 3.0, everything was consolidated under a single interface, the generic `IHost`.

The main class which looked like the following:

`Program.cs`

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateWebHostBuilder(args).Build().Run();
    }

    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseStartup<Startup>();
}
```

Has been adapted to:

`Program.cs`

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateWebHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateWebHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

## Endpoint routing

Endpoint routing isn't something new in ASP.NET Core 3.0, being [introduced in 2.2](https://devblogs.microsoft.com/aspnet/asp-net-core-2-2-0-preview1-endpoint-routing/), but it gains more visibility with this latest release.

The goal of endpoint routing is to make routing usable in ASP.NET Core in general, not just be a feature of MVC.

As an example of the changes this introduces, let's see the `Configure` method of our group management API `Startup` class.

`GroupManagement\src\CodingMilitia.PlayBall.GroupManagement.Web\Startup.cs`

```csharp
public void Configure(IApplicationBuilder app, IHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.Use(async (context, next) =>
    {
        context.Response.OnStarting(() =>
        {
            context.Response.Headers.Add("X-Powered-By", "ASP.NET Core: From 0 to overkill");
            return Task.CompletedTask;
        });

        await next.Invoke();
    });

    app.UseRouting();
    app.UseAuthentication();
    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        // we'll take a another look at this in the next section
        endpoints
            .MapControllers()
            .RequireAuthorization();
    });

    app.Run(async (context) =>
    {
        await context.Response.WriteAsync("No middlewares could handle the request");
    });
}
```

The new bits are the calls to `UseRouting` and `UseEndpoints`:

- `UseRouting` marks the point in the pipeline in which the route will be resolved, so any middleware that comes after it knows what endpoint will eventually run.
- `UseEndpoints` is where we configure the "routable" components, in this specific case we're configuring our controllers (more on why no MVC later). If we take a look at the options autocomplete gives us when we dot on `endpoints`, we'll see things like `MapHub` (for SignalR) or `MapBlazorHub` (for Blazor), which are taking advantage of the new routing capabilities.

We'll certainly make more use of these new routing features in the future, but for now we'll stick to adapting our existing code to the new recommendations - `UseMvc` still works, but we might as well start using the new bits.

One thing that we can immediately see, that's taking advantage of this new feature is the call to `RequireAuthorization` inside `UseEndpoints`. Previously we would enforce that the controllers required authorization through an MVC configuration, as we can see below.

`GroupManagement\src\CodingMilitia.PlayBall.GroupManagement.Web\IoC\ServiceCollectionExtensions.cs`

```csharp
// ...

var mvcBuilder = services.AddMvcCore(options =>
{
    var policy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .RequireClaim("scope", "GroupManagement")
        .Build();

    options.Filters.Add(new AuthorizeFilter(policy));
});

// ...
```

While this still works, so we could have left it, now having routing information available outside of MVC allows to do some things in a more generic way. In this case we're not getting any particular advantage out of it, but if we wanted to use the same authorization infrastructure we're used to in MVC with other kinds of endpoints, we could.

`RequireAuthorization` has some overloads to further configure the authorization requirements of the target endpoint, but we don't really need it here as we can just setup a default policy for the group management API. This can be done when calling `AddAuthorization` in `ConfigureServices`.

```csharp
services.AddAuthorization(options =>
{
    var policy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .RequireClaim("scope", "GroupManagement")
        .Build();

    options.DefaultPolicy = policy;
});
```

## Segregation of MVC components

From the previous section you might have already noticed that MVC components are now better segregated when configuring the application in the `Startup` class, as we can see in both `ConfigureServices` and `Configure` methods.

On the `ConfigureServices` side, we can still call `AddMvc` to get everything setup, but we have some new options if we don't want to use everything: `AddControllers`, `AddControllersWithViews` and `AddRazorPages`.

Similarly, on the `Configure` side, inside `UseEndpoints` we can map things individually, by calling methods such as `MapControllers` or `MapRazorPages`.

Now we can use what we need in our applications. In the group management API and the BFF we only need the controller bits, in the auth service we'll use everything.

## Breaking change in route resolution of async actions

Quick but important shout-out to a breaking change in route resolution of async methods, is that the `Async` suffix is removed from the action names.

As an example, the group management API's `GroupsController.AddAsync` action returns a 201 with a link to the `GetByIdAsync` route. Previously we would do this with:

```csharp
return CreatedAtAction("GetByIdAsync", new {id = group.Id}, group.ToModel());
```

With ASP.NET Core 3.0, we need to get rid of the `Async` suffix, like so:

```csharp
return CreatedAtAction("GetById", new {id = group.Id}, group.ToModel());
```

## Outro

That should be it for our upgrade to .NET and ASP.NET Core 3.0. There are a lot more things going on, so be sure to check out the [docs](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30), we just went through the main things that affected what we developed so far in the series.

For the full diff be sure to look at the repositories on GitHub, and feel free to ask any question. I took the opportunity to do some unrelated changes, as I was touching some code I wanted to improve. but most of the diff should be upgrade related.

Links in the post:

- [Migrate from ASP.NET Core 2.2 to 3.0](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30)
- [.NET Generic Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-3.0)
- [Endpoint Routing in 2.2](https://devblogs.microsoft.com/aspnet/asp-net-core-2-2-0-preview1-endpoint-routing/)
- [Comparing Startup.cs between the ASP.NET Core 3.0 templates](https://andrewlock.net/comparing-startup-between-the-asp-net-core-3-templates/)
- [Understanding ASP.NET Core Endpoint Routing](https://aregcode.com/blog/2019/dotnetcore-understanding-aspnet-endpoint-routing/)

The source code for this post is spread across the repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode032`.

- [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode032)
- [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode032)
- [WebFrontend](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode032)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!