---
author: João Antunes
date: 2019-09-11 18:30:00+01:00
layout: post
title: "Episode 029 - Simplifying the BFF with ProxyKit - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we'll revise our current backend for frontend implementation, introducing ProxyKit to simplify request routing to backing APIs, foregoing the need to implement everything manually."
images:
- '/images/2019/09/11/aspnet-core-from-zero-to-overkill-e029.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- proxykit
slug: aspnet-029-from-zero-to-overkill-simplifying-the-bff-with-proxykit
---

In this episode, we'll revise our current backend for frontend implementation, introducing ProxyKit to simplify request routing to backing APIs, foregoing the need to implement everything manually.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< youtube Wgu97TKaRiI >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
At this point, we have created a stable (yet overkill) base infrastructure for future developments, but in terms of actual application features, we're still on a very basic level, so it's about time we get back to implementing them.

One problem we have on our hands though, is the current implementation of the backend for frontend, which will be a pain to maintain.

When we introduced the BFF in [episode 020](https://blog.codingmilitia.com/2019/05/05/aspnet-020-from-zero-to-overkill-backend-for-frontend-httpclient), we did it with one main goal: have a place to implement features specific to a given frontend, allowing for some level of decoupling between it and the backing APIs that may be used by other client applications.

This still makes sense, but the thing is, not every interaction with the backing APIs needs to be customized. As we saw so far with the group management API, some things can just flow directly to the APIs, the BFF doesn't do much besides adding the authorization token to the request.

In episode 020 we took the opportunity to play around with the HTTP client, but if we continue to implement everything like this, every change we do in the APIs, requires us to change the BFF. An alternative name we can give to a BFF is [API gateway](https://microservices.io/patterns/apigateway.html), and if we think about that for a second, we'll start imagining the amount of work we'll get ourselves into if we need manually configure all the interactions between the frontend and the backend, even when no customization is required.

With this in mind, in this episode we'll simplify the BFF by introducing [ProxyKit](https://github.com/damianh/ProxyKit), which will take care of routing any API requests to the correct backing services.

> **Side note:** Won't really talk about them in this post, but just for information of anyone who's following along the series and checking out the code, I did some minor adjustments to the routes of the BFF, namely the authentication endpoints are no longer on `api/auth`, but just `auth`. Due to this change, also adjusted the reverse proxy configuration, the auth service configured redirect endpoints and the single page application development server proxy settings.

## Introducing ProxyKit to the BFF
The best way to introduce [ProxyKit](https://github.com/damianh/ProxyKit) is probably just to copy the description from its [GitHub page](https://github.com/damianh/ProxyKit):

> A toolkit to create code-first HTTP Reverse Proxies hosted in ASP.NET Core as middleware. This allows focused code-first proxies that can be embedded in existing ASP.NET Core applications or deployed as a standalone server.

I think this description allows us to understand why it's a nice solution for our problem, but lets dissect it a bit.

By being just another middleware we can introduce to ASP.NET Core, it means it's just another thing present in the pipeline, not making everything else useless. In short, everything else will work as usual, so we can have additional endpoints in our BFF to implement specific features, like it's the case of the authentication process.

### Quick look at ProxyKit vs Ocelot
Another option we had to implement this was [Ocelot](https://github.com/ThreeMammals/Ocelot). Even though we could use them both to implement similar behaviors, they're focus is different.

ProxyKit is focused on being a reverse proxy, like HAProxy we've been using, but hosted inside an ASP.NET Core application, which allows us to customize things taking advantage of other framework features and our programming language of choice.

Ocelot is focused on being an API gateway, so out of the box, besides similar routing capabilities for instance, has other features that are useful for this specific scenario, like request aggregation, caching, retries and [others](https://github.com/ThreeMammals/Ocelot#features).

At first glance Ocelot would probably be a better choice for our BFF, with all these extra features. Where it comes short in comparison to ProxyKit is in performance in simple proxying scenarios, which is exactly what we have at this point. In fact, that's even better explained in ProxyKit's [GitHub page](https://github.com/damianh/ProxyKit#7-comparison-with-ocelot).

> Ocelot is an API Gateway that also runs on ASP.NET Core. A key difference between API Gateways and general Reverse Proxies is that the former tend to be message based whereas a reverse proxy is stream based. That is, an API Gateway will typically buffer every request and response message to be able to perform transformations. This is fine for an API Gateway but not suitable for a general reverse proxy performance wise nor for responses that are chunked-encoded. See Not Supported Ocelot docs.

> Combining ProxyKit with Ocelot would give some nice options for a variety of scenarios.

This last phrase is particularly interesting, but not at all unexpected. Being that both work as middlewares in an ASP.NET Core application, we are free to combine the best of each. For now though, we'll go with ProxyKit, in the future we might also introduce the Ocelot to the application.

### Installing and configuring
Installing ProxyKit is just a matter of installing a NuGet package.

```sh
dotnet add package ProxyKit
```

Now to use it, given it's a middleware, you probably can already imagine the steps: go to the `Startup` class, configure the required services in `ConfigureServices` and the pipeline in `Configure`.

To configure DI, it's a simple line:

`WebFrontend\server\src\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web\Startup.cs`
```csharp
services.AddProxy();
```

Now we have to configure the pipeline, so we go down a bit to the `Configure` method. We'll add it to the end of the pipeline, so after `app.UseMvc()`. This allows us to have controllers to handle some requests, but if none matches, it'll proceed through the pipeline, reaching the proxy middleware.

`WebFrontend\server\src\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web\Startup.cs`
```csharp
app.Map("/api/groups", api =>
{
    api.RunProxy(async context =>
    {
        var forwardContext = context
            .ForwardTo("http://groupmanagement:80/groups")
            .CopyXForwardedHeaders();

        var token = await context.GetAccessTokenAsync();
        forwardContext.UpstreamRequest.SetBearerToken(token);

        return await forwardContext.Send();
    });
});
```

As we can see from the above code, we don't require a lot of configuration to get it working. We use `app.Map`, with the base route of the group management API, so it only routes the correct requests, not all of them.

After we branch the pipeline, we can use the `RunProxy` extension method to configure the proxy middleware, using the `ForwardTo` extension method to start the forwarding of the request, where we indicate the target endpoint. This method returns a `ForwardContext` instance, with which we can make additional configurations and finally proceed to forward the request.

Besides the required target endpoint, we're also configuring it to copy any `X-Forwarded-*` HTTP headers that our external reverse proxy (HAProxy) might have included in the request. Finally, before actually forwarding the request, we need to include the JWT token the API needs to identify the user. Previously we did it at the `HttpClient` level (with a `DelegatingHandler`) but with ProxyKit we can just do it directly in the middleware. Note that we can also configure the `HttpClient` used by ProxyKit, so the previous approach could also be used (more info [here](https://github.com/damianh/ProxyKit#24-configuring-proxykits-httpclient)).

With this in place, the BFF can now automagically forward requests to the group management API.

### Preparing for more backing APIs
One thing you might noticed is that we're hardcoding configuration just for the group management API. If we just to this, we'll have to add one middleware in the pipeline for each backing service, which is far from ideal.

Instead, let's rework the middleware configuration to make it easier to proxy more services.

For starters, we'll add something to the `appsettings` that allows us to map an API to its endpoint.

`appsetings.DockerDevelopment.json`
```json
"ApiRoutes": {
    "groups": "http://groupmanagement:80"
}
```

Now we need a way to match request routes to the endpoint, as we configured in the settings file. To do this, I created a new class named `ProxiedApiRouteEndpointLookup`. I won't go into much detail about the implementation, as I think it's an interesting topic for the next episode (love it when the idea for the next episode comes as I prepare the previous one 🙂), but I'll drop the code in here for reference anyway.

`WebFrontend\server\src\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web\ApiRouting\ProxiedApiRouteEndpointLookup.cs`
```csharp
public class ProxiedApiRouteEndpointLookup
{
    // The double dictionary strategy can be simplified if we're able to lookup directly with a ReadOnlySpan<char>
    // Work in progress here -> https://github.com/dotnet/corefx/issues/31942
    
    private readonly Dictionary<string, string> _routeToEndpointMap;
    private readonly Dictionary<int, string[]> _routeMatcher;

    public ProxiedApiRouteEndpointLookup(Dictionary<string, string> routeToEndpointMap)
    {
        _routeToEndpointMap = routeToEndpointMap ?? throw new ArgumentNullException(nameof(routeToEndpointMap));
        _routeMatcher = _routeToEndpointMap
            .Keys
            .GroupBy(
                r => r.GetHashCode(),
                r => r)
            .ToDictionary(
                g => g.Key,
                g => g.ToArray());
    }

    public bool TryGet(PathString path, out string endpoint)
    {
        endpoint = null;
        var pathSpan = path.Value.AsSpan();
        var basePathEnd = pathSpan.Slice(1, pathSpan.Length - 1).IndexOf('/');
        var basePath = pathSpan.Slice(1, basePathEnd > 0 ? basePathEnd : pathSpan.Length - 1);

        // when we upgrade to .NET Core 3.0, we can use string.GetHashCode(basePath)
        // to get the hashcode directly from the span, which will be much better for allocations
        if (_routeMatcher.TryGetValue(basePath.ToString().GetHashCode(), out var routes))
        {
            var route = FindRoute(basePath, routes);
            return !(route is null) && _routeToEndpointMap.TryGetValue(route, out endpoint);
        }
        
        return false;
    }
    
    private static string FindRoute(ReadOnlySpan<char> route, string[] routes)
    {
        for (var i = 0; i < routes.Length; ++i)
        {
            var currentRoute = routes[i];
            if (MemoryExtensions.Equals(route, currentRoute, StringComparison.InvariantCultureIgnoreCase))
            {
                return currentRoute;
            }
        }
        return null;
    }
}
```

Again, we'll see it in more detail in episode 030, but for now the important is that the `ProxiedApiRouteEndpointLookup` receives the route map in the constructor, then is able to map a request path to an API route with a call to the `TryGet` method.

To use it in the middleware, we can configure it in the dependency injection container:

`WebFrontend\server\src\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web\Startup.cs`
```csharp
services.AddSingleton(
    new ProxiedApiRouteEndpointLookup(
        _configuration.GetSection<Dictionary<string, string>>("ApiRoutes")));
```

Now we can adjust the middleware configuration:

`WebFrontend\server\src\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web\Startup.cs`
```csharp
app.Map("/api", api =>
{
    api.RunProxy(async context =>
    {
        var endpointLookup = context.RequestServices.GetRequiredService<ProxiedApiRouteEndpointLookup>();
        if (endpointLookup.TryGet(context.Request.Path, out var endpoint))
        {
            var forwardContext = context
                .ForwardTo(endpoint)
                .CopyXForwardedHeaders();

            var token = await context.GetAccessTokenAsync();
            forwardContext.UpstreamRequest.SetBearerToken(token);

            return await forwardContext.Send();
        }

        return new HttpResponseMessage(HttpStatusCode.NotFound);
    });
});
```

Notice the `app.Map` now maps to any `/api` based route that gets that far in the pipeline, so we can match any API service, not just group management.

Then we use `ProxiedApiRouteEndpointLookup` to try and match the request path to a backing service. If there is a match, we do the same as we did before, just using the provided endpoint instead of an hardcoded one. If there is no match, we respond with a 404.

## Adjustments to MVC based behaviors
The proxying part is done, but we're still not done. Because previously we were basically proxying the requests manually in a controller (`GroupsController`), we had things in place to work with MVC, namely enforcing authorization by using an MVC filter (`AuthorizeFilter`), as well as anti-forgery token validation (`AutoValidateAntiforgeryTokenAttribute`).

With this move to do things out of MVC, this won't work anymore, so we need to pull these into a middleware as well, so it applies to everything in the BFF, MVC or not.

### Enforce authentication at middleware level
To enforce authentication at middleware level we... create a middleware! 🙃

`WebFrontend\server\src\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web\Security\EnforceAuthenticatedUserMiddleware.cs`
```csharp
public class EnforceAuthenticatedUserMiddleware : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        if (!context.User.Identity.IsAuthenticated)
        {
            await context.ChallengeAsync();
            return;
        }

        await next(context);
    }
}
```

Simple stuff I guess. We learned about implementing middlewares in [episode 008](https://blog.codingmilitia.com/2018/12/05/aspnet-008-from-zero-to-overkill-middlewares), so implementing `IMiddleware` and the `InvokeAsync` method should look familiar.

As for the the code that runs on each request, we're simply checking if the user is authenticated. If the user isn't authenticated, we call the `ChallengeAsync` extension method on the `HttpContext`, to issue the login request, which might result in a 401 or a redirect to the auth service, depending on the target endpoint of the request.

All we have to do to use the middleware is adding it to DI and to the pipeline:

`WebFrontend\server\src\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web\Startup.cs`
```csharp
// in ConfigureServices
services.AddSingleton<EnforceAuthenticatedUserMiddleware>();

// ...

// in Configure, between app.UseAuthentication() and app.UseMvc()
app.UseMiddleware<EnforceAuthenticatedUserMiddleware>();
```

### Validate anti-forgery tokens at middleware level
To validate the anti-forgery token, we implement another middleware.

`WebFrontend\server\src\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web\Security\ValidateAntiForgeryTokenMiddleware.cs`
```csharp
public class ValidateAntiForgeryTokenMiddleware : IMiddleware
{
    private readonly IAntiforgery _antiforgery;

    public ValidateAntiForgeryTokenMiddleware(IAntiforgery antiforgery)
    {
        _antiforgery = antiforgery;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        if (ShouldValidate(context))
        {
            await _antiforgery.ValidateRequestAsync(context);
        }

        await next(context);
    }
    
    private static bool ShouldValidate(HttpContext context)
    {
        // as seen on https://github.com/aspnet/AspNetCore/blob/release/3.0/src/Mvc/Mvc.ViewFeatures/src/Filters/AutoValidateAntiforgeryTokenAuthorizationFilter.cs
        
        var method = context.Request.Method;
        return !(HttpMethods.IsGet(method)
                 || HttpMethods.IsHead(method)
                 || HttpMethods.IsTrace(method)
                 || HttpMethods.IsOptions(method));
    }
}
```

Again, not much to it. On each request, if it should be validated, we do it, otherwise we just let it go. Requiring or not the validation depends on the request's method, as some are considered safe and don't require validation.

To use it, the same as for `EnforceAuthenticatedUserMiddleware`:

`WebFrontend\server\src\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web\Startup.cs`
```csharp
// in ConfigureServices
services.AddSingleton<ValidateAntiForgeryTokenMiddleware>();

// ...

// in Configure, after the use of EnforceAuthenticatedUserMiddleware
app.UseMiddleware<ValidateAntiForgeryTokenMiddleware>();
```

## Outro
The end of another episode. If we now run the application, making use of our Docker Compose environment, we should see the application running as before. 

We didn't advance in terms of adding more features to the application, but we made it easier to do so in the future, without the need to mess with the BFF for every little thing we do on the backend services.

We do however retain the ability to implement things at the BFF level if we so desire, so it's a nice middle ground between agility and flexibility.

As usual in this series, I didn't explore everything ProxyKit has to offer, but only what I found was needed for what I wanted to do. For more detail, check out its GitHub page.

Links in the post:
- [ProxyKit](https://github.com/damianh/ProxyKit)
- [Ocelot](https://github.com/ThreeMammals/Ocelot)
- [Pattern: API Gateway / Backends for Frontends](https://microservices.io/patterns/apigateway.html)
- [An alternative way to secure SPAs (with ASP.NET Core, OpenID Connect, OAuth 2.0 and ProxyKit)](https://leastprivilege.com/2019/01/18/an-alternative-way-to-secure-spas-with-asp-net-core-openid-connect-oauth-2-0-and-proxykit/)
- [Episode 020 - The backend for frontend and the HttpClient - ASP.NET Core: From 0 to overkill](https://blog.codingmilitia.com/2019/05/05/aspnet-020-from-zero-to-overkill-backend-for-frontend-httpclient)
- [Episode 008 - Middlewares - ASP.NET Core: From 0 to overkill](https://blog.codingmilitia.com/2018/12/05/aspnet-008-from-zero-to-overkill-middlewares)

The source code for this post is spread across the repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode029`.
- [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode029)
- [WebFrontend](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode029)
- [Deployment](https://github.com/AspNetCoreFromZeroToOverkill/Deployment/tree/episode029)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!
