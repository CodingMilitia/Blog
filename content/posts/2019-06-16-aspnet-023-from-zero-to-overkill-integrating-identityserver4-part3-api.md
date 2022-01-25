---
author: João Antunes
date: 2019-06-16 20:00:00+01:00
layout: post
title: "Episode 023 - Integrating IdentityServer4 - Part 3 - API - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we look at the group management service, and the changes required for it to enforce the requests authentication using an access token (JWT)."
images:
- '/images/2019/06/16/aspnet-core-from-zero-to-overkill-e023.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- identityserver4
slug: aspnet-023-from-zero-to-overkill-integrating-identityserver4-part3-api
---

In this episode, we look at the group management service, and the changes required for it to enforce the requests authentication using an access token (JWT).

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< yt _DDJKFW8LxQ >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In the last post, we've seen how to configure IdentityServer4 in the auth service. In this post, we take advantage of ASP.NET Core built-in features to authenticate requests to the group management API using [JWT (JSON Web Tokens)](https://jwt.io/) provided by the auth service to a client application, after a successful authentication.

## Configuring authentication and authorization
Configuring the application to require JWT authentication is rather simple, almost like magic 😛, because ASP.NET Core comes prepared to handle such scenarios out of the box. We'll start by seeing how to perform these configurations, but later on we'll try to shed some light on some of the magic.

### Setting up JWT authentication
Configuring authentication will require the usual 2 steps in the `Startup` class (or methods called by it): configuring services and configuring the request handling pipeline.

Starting with the simplest part, the request handling pipeline, we just need to add `app.UseAuthentication()`, much like we did when configuring authentication in the auth service.

Now let's move to the service configuration. We have a `ServiceCollectionExtensions` class to where we have been extracting the methods that configure the services, so we'll create a new one named `AddConfiguredAuth` in there (although this should be refactored later, to avoid having all the unrelated configurations in the same place).

`IoC\ServiceCollectionExtensions.cs`
```csharp
public static class ServiceCollectionExtensions
{
    // ...
    public static IServiceCollection AddConfiguredAuth(this IServiceCollection services)
    {
        services
            .AddAuthentication("Bearer")
            .AddJwtBearer("Bearer", options =>
            {
                options.Authority = "https://localhost:5005";
                options.Audience = "GroupManagement";
            });

        return services;
    }
    // ...
}
```

Looking at the code above, I think it's easy to see why I said it was simple to implement, 4 or 5 lines of code take care of everything (because ASP.NET Core implements the hard parts of course).

Let's quickly go through what's going on:
- We call `AddAuthentication` to add the core required services for authentication, along with the default scheme we want to use for authentication.
- We call `AddJwtBearer` to add the required services for JWT authentication. We provide the scheme name, so it can be matched by what we passed to `AddAuthentication` and also provide some options for handling the JWTs.
    - `Authority` is the url of the auth service, so the application can fetch some required information to validate the tokens.
    - `Audience` is used in the token validation to ensure the token was intended to be used by this specific application.

At first glance one might ask, how is this enough to validate the tokens? Even though it might not seem a lot, the `Authority` plays a huge role in this, as OpenId Connect specs include [discovery of information about the provider](https://openid.net/specs/openid-connect-discovery-1_0.html), meaning the OpenId Connect provider exposes well known endpoints where the relying parties can fetch information. We'll see an example in a coming section.

### Setting up authorization based on the authenticated user
In [episode 019](https://blog.codingmilitia.com/2019/04/29/aspnet-019-from-zero-to-overkill-roles-claims-policies) (and even on [016](https://blog.codingmilitia.com/2019/02/24/aspnet-016-from-zero-to-overkill-authentication-with-identity-and-razor-pages)) we took a look at authorization in ASP.NET Core. What we need in this application is very similar. 

Basically we require a user to be authenticated and have a scope named `GroupManagement` to be authorized to access the API. We could use different scopes for different endpoints, but no need to go down that path for now.

To perform these configurations we go to the `ServiceCollectionExtensions.AddRequiredMvcComponents` method and make some additions:

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddRequiredMvcComponents(this IServiceCollection services)
    {
        // ...
        var mvcBuilder = services.AddMvcCore(options =>
        {
            // ...
            var policy = new AuthorizationPolicyBuilder()
                .RequireAuthenticatedUser()
                .RequireClaim("scope", "GroupManagement")
                .Build();

            options.Filters.Add(new AuthorizeFilter(policy));
        });
        
        // ...
        mvcBuilder.AddAuthorization();
        return services;
    }
    // ...
}
```

We saw similar code in episode 019, but for a quick overview, we're building an authorization filter/policy to add to MVC's filters. Building the policy we indicate what we require, which is the user to be authenticated and have a claim named `scope` with the value `GroupManagement`. Given that in the JWT validation we're ensuring `GroupManagement` is a target audience, this scope check could probably be skipped in this simple scenario, using it if we want to have multiple scopes in the same API like mentioned previously.

## OpenId Connect configuration discovery endpoint
I won't even try to go into much detail on OpenId Connect discovery, as I don't really have a complete grasp of the whole protocol, but I think it's interesting to have some understanding about what's going on and not just rely on the "magic" that happens with 4 or 5 lines of code.

As I mentioned previously, providing the auth service url to the JWT validation services goes a long way because OpenId Connect specifies ways of using well known endpoints to gather required information.

If we grab the auth service url and append to it `/.well-known/openid-configuration` we get a `JSON` response with the aforementioned information.

`https://AUTH-SERVICE/well-known/openid-configuration`
```json


{
  "issuer": "http:\/\/localhost:5005",
  "jwks_uri": "http:\/\/localhost:5005\/.well-known\/openid-configuration\/jwks",
  "authorization_endpoint": "http:\/\/localhost:5005\/connect\/authorize",
  "token_endpoint": "http:\/\/localhost:5005\/connect\/token",
  //...
  "frontchannel_logout_supported": true,
  //...
  "scopes_supported": [
    "openid",
    "profile",
    "GroupManagement",
    "offline_access"
  ],
  "claims_supported": [
    "sub",
    "name",
    //...
  ],
  "grant_types_supported": [
    "authorization_code",
    "client_credentials",
    //...
  ],
  "response_types_supported": [
    "code",
    "token",
    //...
  ],
  "response_modes_supported": [
    "form_post",
    "query",
    "fragment"
  ],
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic",
    "client_secret_post"
  ],
  "subject_types_supported": [
    "public"
  ],
  "id_token_signing_alg_values_supported": [
    "RS256"
  ],
  "code_challenge_methods_supported": [
    "plain",
    "S256"
  ]
}
```

I cut off a bit of content to avoid a massive wall of `JSON`, but wanted to point out some interesting bits:
- the endpoints the provider exposes are indicated here
- the `jwks_uri` endpoint provides a set of public keys that the applications can use to validate the JWTs created by the auth service, signed by its private key
- besides endpoints, there are indications of capabilities supported, like `frontchannel_logout_supported`, the scopes and claims supported, etc

## Sample JWT
Since we're checking out some "behind the scenes" stuff that we could ignore, as it's mostly handled for us, but it's always good we know what's going on in our applications, we can take a look at a sample JWT. 

Remember the JWT is not encrypted, just signed so any tampering is detected, so we can grab it and check out what's in there. If we grab a JWT and put it in [jwt.io](https://jwt.io/), we can see its data.

```json
{
  "nbf": 1560702568,
  "exp": 1560702628,
  "iss": "http://localhost:5005",
  "aud": [
    "http://localhost:5005/resources",
    "GroupManagement"
  ],
  "client_id": "WebFrontend",
  "sub": "bbfbc630-6209-43e9-abeb-d26b92d7c76a",
  "auth_time": 1560702553,
  "idp": "local",
  "scope": [
    "openid",
    "profile",
    "GroupManagement",
    "offline_access"
  ],
  "amr": [
    "pwd"
  ]
}
```

Won't go through the whole thing, but we can take a look at some familiar things:
- with `client_id` we can see that the token was emitted for usage by the `WebFrontend` client
- we can see a familiar audience in the `aud` property, `GroupManagement`
- in the `scope` array we see the scopes we discussed in the previous post

## Outro
That should do it of integration of JWT based authentication in the group management API. As we saw, being IdentityServer the provider or something else is irrelevant from the point of view of this API, as long as it provides the required features.

Links in the post:
- [JWT](https://jwt.io/)
- [OpenId Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html)
- [Episode 019 - Roles, claims and policies - ASP.NET Core: From 0 to overkill](https://blog.codingmilitia.com/2019/04/29/aspnet-019-from-zero-to-overkill-roles-claims-policies)
- [Episode 016 - Authentication with Identity and Razor Pages - ASP.NET Core: From 0 to overkill](https://blog.codingmilitia.com/2019/02/24/aspnet-016-from-zero-to-overkill-authentication-with-identity-and-razor-pages)

The source code for this sub-series of posts is scattered across a bunch of repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode021`.
- [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode021)
- [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode021)
- [WebFrontend](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode021)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!