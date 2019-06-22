---
author: João Antunes
date: 2019-06-22 23:00:00+01:00
layout: post
title: "Episode 024 - Integrating IdentityServer4 - Part 4 - Back for Front - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we look at the backend for frontend, and the changes required for it to handle the users authentication, redirection to the identity provider (the IdentityServer4 powered auth service), the inclusion of an access token when making API calls, the refresh of said token and handling CSRF tokens."
image: '/assets/2019/06/22/aspnet-core-from-zero-to-overkill-e024.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- identityserver4
---

In this episode, we look at the backend for frontend, and the changes required for it to handle the users authentication, redirection to the identity provider (the IdentityServer4 powered auth service), the inclusion of an access token when making API calls, the refresh of said token and handling CSRF tokens.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/bK1N-C5zI1Q" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In the past couple of episodes, we saw how to integrate IdentityServer4 into our auth service, then prepared the group management API to make use of the access tokens (particularly, JWT) it gets on each request to authenticate and authorize the user.

In this episode, we'll start adapting the frontend, namely its back for front component, to delegate authentication to the auth service, then include the user's access token in each request to the group management API.

Given we'll use cookies to authenticate each SPA (single page application) request to the BFF, we'll also include support for [cross-site request forgery](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)) tokens.

## Configure the BFF to authenticate/authorize the user
We've seen some ways to implement authentication in ASP.NET Core, and we're going to see yet another one with the OpenID Connect integration.

The gist is the same as we've seen in the previous post on the [group management API](https://blog.codingmilitia.com/2019/06/16/aspnet-023-from-zero-to-overkill-integrating-identityserver4-part3-api) - we'll use `AddAuthentication` with some specific OpenID Connect configurations in `Startup.ConfigureServices` and call `app.UseAuthentication` in the request handling pipeline definition.

`AddAuthentication` has a bit more configurations this time around, so let's jump right into it.

### AddAuthentication
Let's start by looking at the code, then go through it step by step.

`Startup.cs`
```csharp   
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services.AddAuthentication(options =>
    {
        options.DefaultScheme = "Cookies";
        options.DefaultChallengeScheme = "oidc";
    })
    .AddCookie("Cookies", options =>
    {
        options.EventsType = typeof(CustomCookieAuthenticationEvents);
    })
    .AddOpenIdConnect("oidc", options =>
    {
        options.SignInScheme = "Cookies";

        options.Authority = "https://localhost:5005";

        options.ClientId = "WebFrontend";
        options.ClientSecret = "secret";
        options.ResponseType = OidcConstants.ResponseTypes.Code;

        options.SaveTokens = true;
        options.GetClaimsFromUserInfoEndpoint = true;

        options.Scope.Add("GroupManagement");
        options.Scope.Add(OidcConstants.StandardScopes.OfflineAccess);

        options.Events.OnRedirectToIdentityProvider = context =>
        {
            if (!context.HttpContext.Request.Path.StartsWithSegments("/auth/login"))
            {
                context.HttpContext.Response.StatusCode = 401;
                context.HandleResponse();
            }
            return Task.CompletedTask;
        };
    });
    // ...
}
```

The actual call to `AddAuthentication` doesn't to that much, configuring the default authentication scheme and challenge scheme. After this call, on the returned `AuthenticationBuilder` instance we configure the schemes to match with what we configured.

The `Cookies` authentication scheme is configured with `AddCookie` and even though not a lot is going on, it's an important bit. We'll see how this works in more detail in one of the next sections, but setting the `options.EventsType` allows us to provide an alternative implementation of the `CookieAuthenticationEvents` class. By doing this we can, as the name implies, define what happens at certain moments of the cookie authentication flow. In this case, as we'll see later, we're hooking up to an event in order to ensure the tokens we have in our possession (namely access and refresh tokens) are still valid and the user can continue to use the app without re-authenticating.

The challenge scheme, named `oidc`, is configured with the call to `AddOpenIdConnect`. A lot of this should look familiar, as it matches what we configured in the auth service, when we looked at adding `IdentityServer4` to it ([in episode 022](https://blog.codingmilitia.com/2019/06/13/aspnet-022-from-zero-to-overkill-integrating-identityserver4-part2-auth-service)).

We start by setting the `SignInScheme`, which is responsible for persisting the user info we'll obtain from the authentication process.

Then we set the `Authority` with the auth service endpoint, plus `ClientId`, `ClientSecret` and `ResponseType` with the same values we set when configuring the client in the auth service.

`SaveTokens = true` means the tokens will be stored in the cookie. By default it's `false`, to keep the cookie smaller, so in the future we probably should extract it to some server side session, but it works ok for now.

Setting `GetClaimsFromUserInfoEndpoint` to `true`, means that after the OpenID Connect flow is complete and the tokens retrieved, an additional request is made to the user info endpoint to fetch additional user claims.

We add some extra scopes (besides the default OpenID Connect ones), `GroupManagement` and `offline_access` so we can access the group management API and use the refresh token.

Finally, we're registering on the `OnRedirectToIdentityProvider` event. As this application will be used as an API by the SPA and not as a full-on MVC application, it doesn't make sense to redirect API requests to the auth service when the user isn't logged in, but rather responding with a 401 (unauthorized). To achieve this, in the event handler we're providing, all unauthenticated requests that go to an endpoint that's not `/auth/login`, we set the status code to 401 and mark the response has handled, otherwise letting it proceed as usual.

### Requiring the user to be authenticated
We already saw in the previous episode how to ensure all requests to an API are authenticated, so this will be exactly the same, but I'll put it here for reference anyway 🙂.

`Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services
        .AddMvc(options =>
        {
            var policy = new AuthorizationPolicyBuilder()
                .RequireAuthenticatedUser()
                .Build();
            options.Filters.Add(new AuthorizeFilter(policy));
            // ...
        })
        // ...
}
```

As we saw previously, we go to our `AddMvc` call, and in the configuration lambda we create and add an `AuthorizeFilter` with the policies we need, in this case for the user to be authenticated.

### Logging in
As you might have noticed, when we registered on the `OnRedirectToIdentityProvider` for the OpenID Connect authentication challenge scheme, we made it respond with a 401 for all requests except for the `/auth/login` route. That's because we'll use this route to put a login link in the SPA.

In summary, the login strategy will be the following:
- We expose an `/auth/info` endpoint that the SPA can invoke. If it gets a 401, the SPA shows the link for logging in, otherwise it makes use of the returned information.
- We expose an `/auth/login` endpoint, with the sole purpose of redirecting to the auth service and then, when the authentication is completed, redirect to the desired SPA url.

Let's see the `AuthController`'s code, developed to implement the aforementioned login strategy.

`Features/Auth/AuthController.cs`
```csharp
[Route("auth")]
public class AuthController : ControllerBase
{
    // ...

    [HttpGet]
    [Route("info")]
    public ActionResult<AuthInfoModel> GetInfo()
    {
        // CSRF token stuff we'll see later ...
        
        return new AuthInfoModel
        {
            Name = User.FindFirst("name").Value
        };
    }

    [HttpGet]
    [Route("login")]
    public IActionResult Login([FromQuery] string returnUrl)
    {
        /*
        * No need to do anything here, as the auth middleware will take care of redirecting to IdentityServer4.
        * When the user is authenticated and gets back here, we can redirect to the desired url. 
        */
        return Redirect(string.IsNullOrWhiteSpace(returnUrl) ? "/" : returnUrl);
    }
}
```

The `/auth/info` endpoint simply returns some of the current user's information (plus some CSRF token logic we'll see later). It returning 401 on unauthenticated users is handled automatically by the framework, as we saw when we added the authorization filter and transformed the redirect in a 401 in the authentication challenge scheme.

The `/auth/login` endpoint, gets a return url, so it can redirect back the user to the correct place in the SPA. Again, the handling of the unauthenticated users, redirecting to the auth service, is done automatically by the framework.

## Support cross-site request forgery token
To add support for cross-site request forgery token, we have a wee bit more work than what we saw in previous episodes with MVC and Razor Pages. That's because since we're using a SPA instead of Razor generated HTML, so we got some more manual work to do.

Let's begin with `Startup.ConfigureServices`. In here we need to make a couple of changes: add a filter to validate the presence of an anti-forgery token in all requests that should have it (so, all except `GET`, `HEAD`, `OPTIONS`, and `TRACE`) and add helper anti-forgery services.

`Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services
        .AddMvc(options =>
        {
            // ...
            options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute());
        })
    
    // ...
    
    services.AddAntiforgery(options => options.HeaderName = "X-XSRF-TOKEN");
}
```

In the lambda that configures MVC, we can see the new filter being added to validate the anti-forgery tokens. 

After the filter, we can see the configuration of the anti-forgery services, with the call to the `AddAntiforgery` extension method. The `HeaderName` property we're setting, is the name of the header that the SPA will send with the anti-forgery token, so it can be validated server side. If we didn't set this, the server would only try to find the token in the submitted form data, which is the typical behavior in MVC/Razor Pages applications.

## Provide the access token on API calls
Probably the first thing that comes to mind when we think that we must send the access token in each request we'll make to the API, is going to the place we're making the requests and add the header. This would certainly work, but we can use a cleaner solution, which is very reusable: create a class inheriting from `DelegatingHandler` and configure the `HttpClient`s we want to use it.

A `DelegatingHandler` allows us to intercept the `HttpClient`'s requests, so we can add some logic before, after or even instead of the request. In this case, we want to add the access token header before sending the request. The following class `AccessTokenHttpMessageHandler` implements this logic.

`AuthTokenHelpers/AccessTokenHttpMessageHandler.cs`
```csharp
public class AccessTokenHttpMessageHandler : DelegatingHandler
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public AccessTokenHttpMessageHandler(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
        
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        var accessToken = await _httpContextAccessor.HttpContext.GetTokenAsync("access_token");
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
        return await base.SendAsync(request, cancellationToken);
    }
}
```

Not a lot going on, but let's go through it quickly. We're simply overriding the `SendAsync` method, where we can implement the logic we require. We have an `IHttpContextAccessor` instance injected, that we use in `SendAsync` to grab the access token from the `HttpContext` (remember we stored it in the cookies, and can fetch it in this fashion).

With the token in hand, we can add it to the request headers, then pass it on to the base implementation of `SendAsync`, letting the rest of the request flow as normal.

Now we just need to make a couple of changes in `Startup.ConfigureServices`, to use the handler: register it in the dependency injection container and configure it with the desired `HttpClient`s.

`Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services
        .AddTransient<AccessTokenHttpMessageHandler>()
        .TryAddSingleton<IHttpContextAccessor, HttpContextAccessor>();
    //...
    services
        .AddHttpClient<GroupsController>(/*...*/)
        // ...
        .AddHttpMessageHandler<AccessTokenHttpMessageHandler>();
    //...
}
```

As you can see, not a lot to do, we just need to remember to do it 😛. Register the `AccessTokenHttpMessageHandler` with the DI container, while also registering the `IHttpContextAccessor`, as it's not there by default in ASP.NET Core. After that, we can make use of the handler, by calling the `AddHttpMessageHandler` extension method on the `IHttpClientBuilder`.

> Note: a better way to register the `IHttpContextAccessor` is to call the `AddHttpContextAccessor` extension method, but I didn't know about it when I originally wrote this code 🙂.

## Refresh the access token
Now the final thing we need to do to wrap this up, is to make sure we refresh the access token when its expired (or about to).

To handle this, as I briefly mentioned, we have an alternative implementation of `CookieAuthenticationEvents` and some extra helper classes. This means that in each request received, we'll check for the validity of the access token, and if it's expired we refresh it.

Another possible approach to checking the validity of the access token, would be to do it in the `HttpClient`'s delegating handler we implemented. The main difference is, with the cookie authentication events, we make this check once per request to the BFF, while if we do it in the delegating handler, we do it once per request to the backing API. Both have pros and cons, but it seemed like a good opportunity to check out more ways in which we can hook up to the framework's extensibility points, so we'll go with this one 😉.

Let's start with the `CustomCookieAuthenticationEvents` class. When inheriting from `CookieAuthenticationEvents`, we have a bunch of methods we can override. In this case, we want to override `ValidatePrincipal` - if the access token is valid or we are able to refresh it, we can let it flow as normal (with maybe some tweaks), if not, we reject the principal and force a logout, so the user needs to re-authenticate.

`AuthTokenHelpers/CustomCookieAuthenticationEvents.cs`
```csharp
public class CustomCookieAuthenticationEvents : CookieAuthenticationEvents
{
    private readonly ITokenRefresher _tokenRefresher;

    public CustomCookieAuthenticationEvents(ITokenRefresher tokenRefresher)
    {
        _tokenRefresher = tokenRefresher;
    }

    public override async Task ValidatePrincipal(CookieValidatePrincipalContext context)
    {
        var result = await _tokenRefresher.TryRefreshTokenIfRequiredAsync(
            context.Properties.GetTokenValue("refresh_token"),
            context.Properties.GetTokenValue("expires_at"),
            CancellationToken.None);

        if (!result.IsSuccessResult)
        {
            context.RejectPrincipal();
            await context.HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
        }
        else if (result.TokensRenewed)
        {
            context.Properties.UpdateTokenValue("access_token", result.AccessToken);
            context.Properties.UpdateTokenValue("refresh_token", result.RefreshToken);
            context.Properties.UpdateTokenValue("expires_at", result.ExpiresAt);
            context.ShouldRenew = true;
        }
    }
}
```

There are still a couple of missing pieces, but we're able to understand what's going on in the `CustomCookieAuthenticationEvents` anyway. We get an instance of `ITokenRefresher` in the constructor, which we'll see in a bit in more detail, but as the name implies, takes care of refreshing (if needed) an access token. If the token refresh fails, we reject the principal and force logout. If the token is refreshed, then we need to update the values that are stored in the cookie and renew the cookie for the user (hence the `context.ShouldRenew = true`). Another scenario is that the token might still be valid, so we don't need to do anything (hence no specific code for that scenario).

Before looking at the `ITokenRefresher` and its implementation, let's see the `TokenRefreshResult` class which is returned by `TryRefreshTokenIfRequiredAsync`.

`AuthTokenHelpers/TokenRefreshResult.cs`
```csharp
public class TokenRefreshResult
{
    private static readonly TokenRefreshResult NoRefreshNeededResult =
        new TokenRefreshResult(true, false, null, null, null);

    private static readonly TokenRefreshResult FailedResult =
        new TokenRefreshResult(false, false, null, null, null);

    protected TokenRefreshResult(
        bool isSuccessResult,
        bool tokensRenewed,
        string accessToken,
        string refreshToken,
        string expiresAt)
    {
        IsSuccessResult = isSuccessResult;
        TokensRenewed = tokensRenewed;
        AccessToken = accessToken;
        RefreshToken = refreshToken;
        ExpiresAt = expiresAt;
    }

    public bool IsSuccessResult { get; }
    public bool TokensRenewed { get; }
    public string AccessToken { get; }
    public string RefreshToken { get; }
    public string ExpiresAt { get; }

    public static TokenRefreshResult Success(
        string accessToken,
        string refreshToken,
        string expiresAt)
    {
        return new TokenRefreshResult(true, true, accessToken, refreshToken, expiresAt);
    }

    public static TokenRefreshResult Failed() => FailedResult;

    public static TokenRefreshResult NoRefreshNeeded() => NoRefreshNeededResult;
}
```

As we can see, besides having the necessary properties to check the results in the `CustomCookieAuthenticationEvents` class, we have some helper methods that the `ITokenRefresher` implementation can use to achieve nicer readability.

The `ITokenRefresher` interface exposes a single method, `TryRefreshTokenIfRequiredAsync` which... does what the name says 😛

`AuthTokenHelpers/ITokenRefresher.cs`
```csharp
/// <summary>
/// Provides an easy way to ensure the user's access token is up to date. 
/// </summary>
public interface ITokenRefresher
{
    /// <summary>
    /// Tries to refresh the current user's access token if required.
    /// </summary>
    /// <param name="refreshToken">The current refresh token.</param>
    /// <param name="expiresAt">The current token expiration information.</param>
    /// <param name="ct">The async cancellation token.</param>
    /// <returns><code>True</code> if refresh is not needed or executed successfully, <code>False</code> otherwise.</returns>
    Task<TokenRefreshResult> TryRefreshTokenIfRequiredAsync(
        string refreshToken,
        string expiresAt,
        CancellationToken ct);
}
```

Now let's see its implementation.

`AuthTokenHelpers/TokenRefresher.cs`
```csharp
/// <inheritdoc />
public class TokenRefresher : ITokenRefresher
{
    private static readonly TimeSpan TokenRefreshThreshold = TimeSpan.FromSeconds(30);

    private readonly HttpClient _httpClient;
    private readonly IDiscoveryCache _discoveryCache;
    private readonly ILogger<TokenRefresher> _logger;

    public TokenRefresher(
        HttpClient httpClient,
        IDiscoveryCache discoveryCache,
        ILogger<TokenRefresher> logger)
    {
        _httpClient = httpClient;
        _discoveryCache = discoveryCache;
        _logger = logger;
    }

    /// <inheritdoc />
    public async Task<TokenRefreshResult> TryRefreshTokenIfRequiredAsync(
        string refreshToken,
        string expiresAt,
        CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(refreshToken))
        {
            return TokenRefreshResult.Failed();
        }
            
        if (!DateTime.TryParse(expiresAt, out var expiresAtDate) || expiresAtDate >= GetRefreshThreshold())
        {
            return TokenRefreshResult.NoRefreshNeeded();
        }

        var discovered = await _discoveryCache.GetAsync();
        var tokenResult = await _httpClient.RequestRefreshTokenAsync(
            new RefreshTokenRequest
            {
                Address = discovered.TokenEndpoint,
                ClientId = "WebFrontend",
                ClientSecret = "secret",
                RefreshToken = refreshToken
            }, ct);

        if (tokenResult.IsError)
        {
            _logger.LogDebug(
                "Unable to refresh token, reason: {refreshTokenErrorDescription}",
                tokenResult.ErrorDescription);
            return TokenRefreshResult.Failed();
        }

        var newAccessToken = tokenResult.AccessToken;
        var newRefreshToken = tokenResult.RefreshToken;
        var newExpiresAt = CalculateNewExpiresAt(tokenResult.ExpiresIn);

        return TokenRefreshResult.Success(newAccessToken, newRefreshToken, newExpiresAt);
    }

    private static string CalculateNewExpiresAt(int expiresIn)
    {
        // TODO: abstract usages of DateTime to ease unit tests
        return (DateTime.UtcNow + TimeSpan.FromSeconds(expiresIn)).ToString("o", CultureInfo.InvariantCulture);
    }

    private static DateTime GetRefreshThreshold()
    {
        // TODO: abstract usages of DateTime to ease unit tests
        return DateTime.UtcNow + TokenRefreshThreshold;
    }
}
```

The code isn't super complex, but let's go though it carefully, as there are some interesting bits in it.

Taking a look at the constructor, 2 of the dependencies injected are well known by now, but the other isn't. `IDiscoveryCache` is a helper provided in the `IdentityModel` NuGet package, allowing us to get the info from the discovery endpoint of the OpenID Connect provider (and cache it) so we can use it in other requests to the provider, in this case, for token refresh.

Getting into the `TryRefreshTokenIfRequiredAsync` method implementation, we have:
- Check if we have a refresh token, if not, it's a fail.
- Check if the token expiration date is below a threshold, if it isn't we can return a `NoRefreshNeeded` immediately.
- Fetch the provider information with the `_discoveryCache`.
- Make a request to refresh the token, using the info we configured in `Startup` class as well (yes it should come from configuration and not be hardcoded here 🙃). Note that the `RequestRefreshTokenAsync` is an extension method from the `IdentityModel` NuGet package, simplifying the creation of the refresh token request.
- If the token refresh was successful, return success with the required information, otherwise return a failure.

And that should do it for the token refresh logic. It's not a lot, but a good chunk of code nonetheless. Maybe we can put it in a shared package if we need to reuse it in other components of the PlayBall application.

> PS: don't forget to register all the required services in the DI container.

## Outro
Wrapping up, we've seen how to delegate the user authentication to the auth service, while getting and maintaining the tokens to access the backing APIs (right now, only the group management API). We also saw how to get the CSRF token infrastructure ready to work with a SPA, instead of a the usual server side rendering with Razor. In the next episode, we'll wrap up the sub-series, by seeing the changes made to the frontend to play well with the BFF and integrate everything we developed so far.

As a reminder (and I'll probably repeat myself in the next episode), all of this is just one of (probably) many possible ways of tackling this problem, so if you have a different idea and think it's better, go for it 🙂 (and let me know so I can improve mine).

Links in the post:
- [Cross-Site Request Forgery (CSRF)](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF))
- [Episode 023 - Integrating IdentityServer4 - Part 3 - API - ASP.NET Core: From 0 to overkill](https://blog.codingmilitia.com/2019/06/16/aspnet-023-from-zero-to-overkill-integrating-identityserver4-part3-api)

The source code for this sub-series of posts is scattered across a bunch of repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode021`.
- [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode021)
- [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode021)
- [WebFrontend](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode021)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!