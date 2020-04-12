---
author: João Antunes
date: 2019-06-13 18:00:00+01:00
layout: post
title: "Episode 022 - Integrating IdentityServer4 - Part 2 - Auth Service - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we start looking at the code needed to integrate IdentityServer4 in our application, namely with the authentication service we developed previously."
images:
- '/assets/2019/06/13/aspnet-core-from-zero-to-overkill-e022.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- identityserver4
slug: aspnet-022-from-zero-to-overkill-integrating-identityserver4-part2-auth-service
---

In this episode, we start looking at the code needed to integrate IdentityServer4 in our application, namely with the authentication service we developed previously.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< yt EWFsiLSWMjg >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In this first part of the sub-series of posts on integrating IdentityServer - or more precisely, authentication and authorization - into the PlayBall application, we'll see how to configure it to play well with ASP.NET Core Identity, setup the OpenId Connect / OAuth 2.0 bits, as well as making sure its dependencies are taken care of (like a required data store).

Even though it's in the title of the sub-series, this is the only part that explicitly uses IdentityServer. The other parts of this sub-series need only to know that the auth service is an OpenId Connect compliant identity provider. This means it should be easy to replace the auth service without affecting the other services, as long as we use another OpenId Connect implementing identity provider.

## Configuring IdentityServer4
So let's begin configuring IdentityServer. Of course this will be very focused on the example application we're building in this series, so it won't make use of all the configuration options IdentityServer provides us with.

### NuGet packages
We'll need to install some packages to use IdentityServer. The core package is simply named [IdentityServer4](https://www.nuget.org/packages/IdentityServer4/). As expected, it contains the core features of IdentityServer4. IdentityServer4 is however very extensible, so we can add other packages that build upon it (or create our own if needed).

In the next sections, as required, we'll see some other packages be added with extra features.

### First bits of configuration
As many other things that integrate with ASP.NET Core, IdentityServer4 is added to the application through the `Startup` class, in the `ConfigureServices` and `Configure` methods.

Starting with `Startup.Configure`, this is where we add IdentityServer to the request handling pipeline, so it can take of OpenId Connect / OAuth 2.0 endpoints for us. A call to `app.UseIdentityServer();` is all that's needed to get this part done.

In `Startup.ConfigureServices` is where we would configure everything else for IdentityServer, but instead of having the `Startup` class continue to grow like this, I extracted some stuff already there into auxiliary classes with extension methods, in the `IoC` folder - ASP.NET Core Identity configuration was moved to `IdentityExtensions`, localization configuration moved into `LocalizationExtensions` and MVC configurations got their own `MvcExtensions`. Following this approach, IdentityServer's configuration was created in the `IdentityServerExtensions` class.

For quick reference, `ConfigureServices` looks like this now:

`Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services
        .AddConfiguredMvc()
        .AddConfiguredLocalization()
        .AddConfiguredIdentity(_configuration)
        .ConfigureApplicationCookie(options =>
        {
            options.LoginPath = "/Login";
            options.LogoutPath = "/Logout";
            options.AccessDeniedPath = "/AccessDenied";
        })
        .AddConfiguredIdentityServer(_environment, _configuration);
}
```

In the next sections we'll see the implementation of `IdentityServerExtensions.AddConfiguredIdentityServer`.

### AddIdentityServer
In the `IdentityServerExtensions` class, we have a single extension method for `IServiceCollection`, named `AddConfiguredIdentityServer`. With the IdentityServer4 NuGet package installed, when we dot on an `IServiceCollection` we get access to `AddIdentityServer`, the entry point for configuration. This method has a couple of overloads, one that receives a `Action<IdentityServerOptions>` and another that gets a `IConfiguration` that should map to a `IdentityServerOptions`. For now, for simplicity while learning, we're going with the `Action` based overload, but the other should be a better choice to use with configurations.

`AddIdentityServer` with the `IdentityServerOptions` allows us to configure a bunch of stuff, but for now the only thing we're configuring is enabling the raising of events by IdentityServer, so we can better understand what's going on while we build the application. These events are not like logs, these would be logged anyway, depending on the levels configured of course. These events are more specific to the work IdentityServer is doing in regards to the authentication flows, like when a user is authenticated and a token is created, or when a user consents for the requested information, etc. More info [here](http://docs.identityserver.io/en/latest/topics/events.html).

`IoC/IdentityServerExtensions.cs`
```csharp
public static class IdentityServerExtensions
{
    public static IServiceCollection AddConfiguredIdentityServer(/*...*/)
    {
        var builder = services.AddIdentityServer(options =>
            {
                options.Events.RaiseErrorEvents = true;
                options.Events.RaiseInformationEvents = true;
                options.Events.RaiseFailureEvents = true;
                options.Events.RaiseSuccessEvents = true;
            })
            // ...
```

For reference, below we can see an example event printed out to the console:

```bash
info: IdentityServer4.Validation.TokenRequestValidator[0]
      Token request validation success
{
        "ClientId": "WebFrontend",
        "GrantType": "authorization_code",
        "AuthorizationCode": "46eb980eebde8b886b2b0fa3d4da465f9c3bfb374e8adc3b4342eaf0dbe7c04e",
        "Raw": {
          "client_id": "WebFrontend",
          "client_secret": "***REDACTED***",
          "code": "46eb980eebde8b886b2b0fa3d4da465f9c3bfb374e8adc3b4342eaf0dbe7c04e",
          "grant_type": "authorization_code",
          "redirect_uri": "http://localhost:5000/signin-oidc"
        }
      }
```

### Configuring user information, APIs and clients
After invoking `AddIdentityServer`, we get an instance of `IIdentityServerBuilder`, which we use to configure even more things. We'll start by some of the most interesting bits, namely the user information that'll be available to to the clients, the APIs that we'll be accessed using a token provided by IdentityServer and the clients (or relying parties, in OpenId Connect wording) that'll use the auth service as a means of authentication.

While probably not the ideal for production ready applications, we'll use in memory stores for all of these configurations, keeping it simple as we explore.

#### User information
Let's start with the user information that'll be made available to the client applications. We configure this by invoking `AddInMemoryIdentityResources`, passing in a collection of `IdentityResource`. These `IdentityResource`s indicate the claims that'll be made available to the clients. The two main properties of these resources are the name and a list of claims it'll provide. A client can then be configured, indicating the name of the `IdentityResource` as a scope to which it wants to have access, getting the claims associated with that resource.

We can use `IdentityResource`s that are provided out of the box or we can create our own. In this case, we're simply going with a couple of already provided resources, `OpenId` and `Profile`.

The code for this is the following:

`IoC/IdentityServerExtensions.cs`
```csharp
public static class IdentityServerExtensions
{
    public static IServiceCollection AddConfiguredIdentityServer(/*...*/)
    {
        var builder = services.AddIdentityServer(/*...*/)
            .AddInMemoryIdentityResources(GetIdentityResources())
            // ...
    }
    
    private static IEnumerable<IdentityResource> GetIdentityResources()
    {
        return new IdentityResource[]
        {
            new IdentityResources.OpenId(),
            new IdentityResources.Profile { Required = true }
        };
    }
    // ...
```

Just a quick not on the `Required = true` seen above. This makes this scope mandatory, otherwise the user could disable the access of a client application to this scope, and we want to make sure the applications have full access to the required data.

The `OpenId` scope provides claims as `sub`, which is a unique identifier for the user. The `Profile` scope will provide claims like `name`, `nickname`, `picture` and others.

#### APIs
Configuring the API resources is pretty similar to the identity resources. On the `IIdentityServerBuilder` we can invoke `AddInMemoryApiResources`, providing it with the `ApiResource`s representing the APIs we want to secure.

Let's take a quick look at the code:

`IoC/IdentityServerExtensions.cs`
```csharp
public static class IdentityServerExtensions
{
    public static IServiceCollection AddConfiguredIdentityServer(/*...*/)
    {
        var builder = services.AddIdentityServer(/*...*/)
            // ...
            .AddInMemoryApiResources(GetApis())
            // ...
    }
    
    private static IEnumerable<ApiResource> GetApis()
    {
        var apiResource = new ApiResource("GroupManagement", "Group Management");
        apiResource.Scopes.First().Required = true;
        return new[]
        {
            apiResource
        };
    }
    // ...
```

When creating an `ApiResource`, the name we give to it will be the name the client applications will use as the scope, unless we set the `Scopes` property. This can be useful if we want to create multiple scopes of a single API, e.g. to have a readonly scope and another that allows writes.

We have some more things we can setup in an `ApiResource`, like the user claims that should be included in the access token that's sent to the API; secrets, so the API may use the introspection API to validate an access token if it requires to do so - normally when using [JWT](https://jwt.io/) access tokens (which are probably more common), this isn't required as it can be validated without making a request, but if we were to use [reference tokens](http://docs.identityserver.io/en/latest/topics/reference_tokens.html), then we would require this extra request.

#### Clients
Finally, let's configure the client application (relying party). Once again, `IIdentityServerBuilder` gives us access to `AddInMemoryClients`, to which we provide a collection of clients.

Let's check out the code, then go through it:

`IoC/IdentityServerExtensions.cs`
```csharp
public static class IdentityServerExtensions
{
    public static IServiceCollection AddConfiguredIdentityServer(/*...*/)
    {
        var builder = services.AddIdentityServer(/*...*/)
            // ...
            .AddInMemoryClients(GetClients())
            // ...
    }
    
    private static IEnumerable<Client> GetClients()
    {
        return new[]
        {
            new Client
            {
                ClientId = "WebFrontend",
                AllowedGrantTypes = GrantTypes.Code,
                ClientSecrets = {new Secret("secret".Sha256())},
                RedirectUris = new[] {"http://localhost:5000/signin-oidc"},
                RefreshTokenUsage = TokenUsage.OneTimeOnly,
                AllowedScopes =
                {
                    IdentityServerConstants.StandardScopes.OpenId,
                    IdentityServerConstants.StandardScopes.Profile,
                    "GroupManagement"
                },
                AllowOfflineAccess = true,
                AccessTokenLifetime = 60,
                RefreshTokenExpiration = TokenExpiration.Sliding
            }
        };
    }
    // ...
```

As usual in this post, the options we can see are just some of them, there are a bunch more available for configuration. Let's quickly go through the properties we're setting in the `Client`.

- `ClientId` - the client's identifier, which it'll use when initiating an authentication flow.
- `AllowedGrantTypes` - the types of grants the client is allowed. It's with these grants that we specify the kinds of flows the client can use. The `GrantTypes` class provides some common grant types, and by using `GrantTypes.Code` we're saying the client can use the authorization code flow we talked about in the previous episode.
- `ClientSecrets` - client secret used in some interactions between the client and the auth service, for example, when exchanging the authorization code for an access token.
- `RedirectUris` - the URIs that the client application might use as a redirect target after a successful authentication flow.
- `RefreshTokenUsage` - indicates if the refresh token is kept as is after using it or a replacement is provided once used (lifetime is not affected, it's just a different `string` that's returned to represent the same token).
- `AllowedScopes` - the scopes the client may request access to. Notice we're using the two more generic ones, plus the one that'll provide access to the group management API.
- `AllowOfflineAccess` - the name of this one is not very obvious, but having this set to `true` is what will allow the client to use refresh tokens.
- `AccessTokenLifetime` - the amount of time (in seconds) through which the access token is valid. Of course these 60 seconds are a bit too low, but it's just for us to see everything working.
- `RefreshTokenExpiration` - indicates whether the refresh token expires at a specific point in time or its lifetime is extended each time it's used.

### Integrate with ASP.NET Core Identity
Let's continue our look at IdentityServer4 configuration with its integration with ASP.NET Core Identity.

As we want to integrate IdentityServer with the ASP.NET Identity bits we already have in place for authentication, we can add the [IdentityServer4.AspNetIdentity](https://www.nuget.org/packages/IdentityServer4.AspNetIdentity) package, that takes care of that for us. Then, all we need to do is call `AddAspNetIdentity` on the `IIdentityServerBuilder` instance we have.

`IoC/IdentityServerExtensions.cs`
```csharp
public static class IdentityServerExtensions
{
    public static IServiceCollection AddConfiguredIdentityServer(/*...*/)
    {
        var builder = services.AddIdentityServer(/*...*/)
            // ...
            .AddAspNetIdentity<PlayBallUser>()
            // ...
    }
    // ...
```

### Integrate operational store with EF Core
So far we've been using in-memory everything for our configurations. While it works for testing in the cases we've seen so far, there are other things that even if we're just testing out, will be annoying if we don't use persistence. One such case is the operational data.

Operational data in IdentityServer are things like information about the refresh tokens, reference tokens, temporary flow data and so on. If we don't configure a persistent store for all of this, it will be in memory and every time we restart the auth service or if we use multiple instances of it, it won't work well, so it's important we set this up.

We can implement our own operational store, but IdentityServer already provides an implementation based on Entity Framework Core. To use it we need to install the [IdentityServer4.EntityFramework](https://www.nuget.org/packages/IdentityServer4.EntityFramework) NuGet package.

Now to use it, we need to go through a couple of steps: 
- configure its usage in our `IdentityServerExtensions` class
- create migrations so the database can be created and updated

#### Configure operational store implementation
Let's begin with the simplest part, adding the operational store configuration to the startup process.

Again, we make use of the `IIdentityServerBuilder`, where we have the `AddOperationalStore` extension method available. In this method we can setup, among other things, the `DbContext`.

In the below code we're configuring the `DbContext` with a connection string added to the configurations, plus setting the migrations assembly as the current one (the auth service project). This migrations assembly bit is needed as it is a different from the assembly that contains the `DbContext`.

`IoC/IdentityServerExtensions.cs`
```csharp
public static class IdentityServerExtensions
{
    public static IServiceCollection AddConfiguredIdentityServer(/*...*/)
    {
        var builder = services.AddIdentityServer(/*...*/)
            // ...
            .AddOperationalStore(options =>
            {
                options.ConfigureDbContext = b =>
                    b.UseNpgsql(configuration.GetConnectionString("PersistedGrantDbContext"),
                        npgOptions =>
                            npgOptions.MigrationsAssembly(
                                typeof(IdentityServerExtensions).Assembly.GetName().Name));
            })
            .AddInMemoryCaching();
            
        // ...
    }
    // ...
```

Another interesting thing to note, is the call to `AddInMemoryCaching`. This is just to avoid continuos hammering on the database (although I think this isn't used for the operational store, only some of the others). 

#### Create operational store migrations
Creating the migrations for the operational store is a similar task to what we've seen in [episode 011](https://blog.codingmilitia.com/2019/01/16/aspnet-011-from-zero-to-overkill-data-access-with-entity-framework-core).

The command to create the initial migration used for the operational store is the following:

```bash
dotnet ef migrations add InitialIdentityServerPersistedGrantDbMigration -c PersistedGrantDbContext -o "Migrations\IdentityServer\PersistentGrantDb"
```

In the command, we're indicating the `DbContext` we want to create the migrations for, plus the location we want the generated files to be put.

While we're at it, we can also add migration execution to the application startup. To do this we just need to make a small tweak to the `DatabaseExtensions` class, to add the `PersistedGrantDbContext` migration execution next to the `AuthDbContext` ones.

`StartupHelpers\DatabaseExtensions.cs`
```csharp
internal static class DatabaseExtensions
{
    internal static async Task EnsureDbUpToDateAsync(this IWebHost host)
    {
        using (var scope = host.Services.CreateScope())
        {
            var hostingEnvironment = scope.ServiceProvider.GetRequiredService<IHostingEnvironment>();
            if (hostingEnvironment.IsDevelopment() || hostingEnvironment.IsEnvironment("DockerDevelopment"))
            {
                var authDbContext = scope.ServiceProvider.GetRequiredService<AuthDbContext>();
                await authDbContext.Database.MigrateAsync();

                var grantDbContext = scope.ServiceProvider.GetRequiredService<PersistedGrantDbContext>();
                await grantDbContext.Database.MigrateAsync();
            }
        }
    }
}
```

### Signing credentials
One last thing we're (kind of) configuring with IdentityServer are the signing credentials.

The signing credentials are used by IdentityServer to sign the tokens, so they can be checked for tampering. The auth service will sign the tokens with a private key, and the recipients of the token can validate them using a public key.

For now I didn't want to spend time in this (although we'll need to eventually), so in this cases we can just use the `AddDeveloperSigningCredential` extension method, which will create some development time signing credentials.

`IoC/IdentityServerExtensions.cs`
```csharp
public static class IdentityServerExtensions
{
    public static IServiceCollection AddConfiguredIdentityServer(/*...*/)
    {
        var builder = services.AddIdentityServer(/*...*/)
            // ...
            
        if (environment.IsDevelopment())
        {
            builder.AddDeveloperSigningCredential();
        }
        else
        {
            throw new Exception("need to configure key material");
        }
        
        // ...
    }
    // ...
```

In the future we'll need to add some real credentials, using the `AddSigningCredential` extension method, to which we can provide a certificate, a RSA key or other alternatives available through some overloads.

### Consent screen
We won't need a consent screen, as the auth service will be used only by the other components of the PlayBall application and not as a generic SSO service (like Google or Microsoft logins). Anyway, just to see it in action I created a simple (and far from perfect) screen, based on available code from the IdentityServer folks.

There's a repository available with UI elements to use with IdentityServer, [IdentityServer4.Quickstart.UI
](https://github.com/IdentityServer/IdentityServer4.Quickstart.UI). It contains an implementation based on ASP.NET Core MVC. As we're using Razor Pages in the auth service, I based the development on the available code, adapting to Razor Pages as needed.

If you want to check it out, the code is in the `Pages\Consent` folder, but won't really write about it here, as it's a lot of boring code, setting up forms, check the options the user's make, make redirects to carry-on with the flow and so on. Feel free to drop any questions you encounter if you take a look at the code.

## Outro
That's all for this episode, has we have the IdentityServer4 integration ready on the auth service side. In the next episode, we'll take a look at how to configure the group management API to require a token to authenticate each request.

Links in the post:
- [IdentityServer4 Docs](http://docs.identityserver.io/en/latest/index.html)
- [IdentityServer4 Docs - Events](http://docs.identityserver.io/en/latest/topics/events.html)
- [IdentityServer4 Docs - Reference Tokens](http://docs.identityserver.io/en/latest/topics/reference_tokens.html)
- [JWT](https://jwt.io/)
- [IdentityServer4.Quickstart.UI repository](https://github.com/IdentityServer/IdentityServer4.Quickstart.UI)
- [IdentityServer4 NuGet package](https://www.nuget.org/packages/IdentityServer4/)
- [IdentityServer4.AspNetIdentity NuGet package](https://www.nuget.org/packages/IdentityServer4.AspNetIdentity)
- [IdentityServer4.EntityFramework NuGet package](https://www.nuget.org/packages/IdentityServer4.EntityFramework)
- [Episode 011 - Data access with Entity Framework Core - ASP.NET Core: From 0 to overkill](https://blog.codingmilitia.com/2019/01/16/aspnet-011-from-zero-to-overkill-data-access-with-entity-framework-core).

The source code for this sub-series of posts is scattered across a bunch of repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode021`.
- [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode021)
- [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode021)
- [WebFrontend](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode021)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!