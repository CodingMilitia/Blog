---
author: João Antunes
date: 2019-08-31 12:30:00+01:00
layout: post
title: "Episode 028 - Multiple service instances tweaks - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we'll take a look at some things we need to keep in mind when we want to run multiple instances of an ASP.NET Core application, namely data protection configuration and how it impacts our application even without us using it directly."
image: '/assets/2019/08/31/aspnet-core-from-zero-to-overkill-e028.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- docker
---

In this episode, we'll take a look at some things we need to keep in mind when we want to run multiple instances of an ASP.NET Core application, namely data protection configuration and how it impacts our application even without us using it directly.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/_-eV5ZcCwag" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
As we have our application working as a whole, with an easy way to get running thanks to Docker and Docker Compose, we can take advantage of it to check out some scenarios that we didn't consider until now, as we were just running all the components in development mode.

One important scenario to consider, is having multiple instances of each component running in parallel. It's a very important topic to consider, particularly in production, not only to share the load across instances, but also to have redundancy, in case one of the instances goes down.

In this context of multiple instances of an application, the topic of ASP.NET Core's data protection APIs comes afloat, but is something we really didn't bother with so far. Time to change that 🙂.

## What is data protection in ASP.NET Core and why should I care?
In ASP.NET Core, the concept of data protection (although the term is a big vague) is tied to the need to ensure that a certain piece of information isn't tampered with even when it leaves the security of our server.

A practical example of this need are cookies. When the framework sets the authentication cookie, which might include the permissions the user has, it needs to be sure that no one will go and change the cookie, adding more permissions than there are in reality. To ensure this, the cookie is signed (not to confuse with encrypted), using a secret that only the server knows. If someone changes the cookie, the server will be able to detect that the signature no longer matches, so the cookie is invalid.

We can use these data protection APIs on our own if we want (we haven't so far in our application), but even if we don't, the framework is using them under the hood, for things like cookies or CSRF tokens.

You can read much more about it in the official [ASP.NET Core docs](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/introduction?view=aspnetcore-2.2).

## Why has it been working well until now?
Out of the box, the data protection APIs will work just fine, as the keys will be generated automatically. The place where the keys are stored depends on some conditions. In our scenario, which are Linux Docker containers, the keys are generated but only stay in memory, so when the application goes down, that set of keys is lost (more details [here](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/default-settings?view=aspnetcore-2.2)).

So far this hasn't caused us much trouble, but we can change that easily. Let's head to our Docker Compose file, add another instance of the backend for frontend and try it out.

`Deployment\docker\docker-compose.dev.yml`
```yml
# ...
bff:
    build: ../../WebFrontend/server/.
    image: codingmilitia/webfrontend/bff:latest
    networks:
        - internal-network
    depends_on:
        - groupmanagement
    environment:
        - ASPNETCORE_ENVIRONMENT=DockerDevelopment
    # this is not the right way to implement multiple nodes with compose, but it's the quickest for the demo we're doing
    another-bff:
    build: ../../WebFrontend/server/.
    image: codingmilitia/webfrontend/bff:latest
    networks:
        - internal-network
    depends_on:
        - groupmanagement
    environment:
        - ASPNETCORE_ENVIRONMENT=DockerDevelopment
# ...
```

We also need to tweak the HAProxy configuration, to also route the requests to the new BFF instance.

`Deployment\docker\reverse-proxy\haproxy.cfg`
```cfg
# ...
backend bff
    option forwardfor
    server bff1 bff:80
    server bff2 another-bff:80
# ...
```

Now if we try to run the application, at first glance it'll seem to be running fine, but as soon as we try to login, when the auth service redirects back to the BFF, things aren't working as expected. That's because, as part of the [OpenID Connect flow we're using](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest), some parameters passed in (like the state) are signed, plus some cookies are created for correlation when we get the result back. This worked fine before, because we only had one instance of the BFF running, so it would always use the same set of in memory keys to handle it all. Now that we have two instances, if the flow starts in one instance but ends in another, it all goes south because the instances use different data protection keys.

## Using a shared directory to store data protection keys
We have a [bunch of options](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/implementation/key-storage-providers?view=aspnetcore-2.2) to share the keys across instances of a service, like file system, EF Core, Redis and more. For our current scenario, we'll go with a simple approach and use a shared directory.

As we're running in Docker, to have a directory shared by containers we need to create a volume. Let's get back to the Docker Compose file and do that.

`Deployment\docker\docker-compose.dev.yml`
```yml
# ...
services:
    # ...
    bff:
        # ...
        volumes:
            - bff-data-protection-keys:/var/lib/bff/dataprotectionkeys
    another-bff:
        # ...
        volumes:
            - bff-data-protection-keys:/var/lib/bff/dataprotectionkeys
    # ...
volumes:
    bff-data-protection-keys:
# ...
```

At the end of the file, we declared a named volume. Then, for the `bff` and `another-bff` services, we mapped that volume to a specific folder.

Now we need to configure the application to store the keys in shared directory. Heading to the BFF project, we can go to the `appsettings.DockerDevelopment.json` file and add the following:

`WebFrontend\server\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web\appsettings.DockerDevelopment.json`
```json
// ...
 "DataProtectionSettings": {
    "Location":  "/var/lib/bff/dataprotectionkeys"
  }
// ...
```

Then in the `Startup` class:
`WebFrontend\server\src\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web\Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ...

    var dataProtectionKeysLocation = _configuration.GetSection<DataProtectionSettings>(nameof(DataProtectionSettings)).Location;
    if (!string.IsNullOrWhiteSpace(dataProtectionKeysLocation))
    {
        services
            .AddDataProtection()
            .PersistKeysToFileSystem(new DirectoryInfo(dataProtectionKeysLocation));
            // TODO: encrypt the keys
    }
}
```

With this in place, if we run the application again, the two instances of the BFF will be able to share the keys, so the login process will now work as expected.

**Note:** just a quick note, as you might have noticed from the TODO comment, as it is, the keys will be stored in plain text in the configured folder. Ideally we should encrypt them, but I don't feel like messing with certificates at this point, so it'll be a topic for another time.

## IdentityServer signing credentials
Since we're in the topic of shared keys, there's also the case for the signing credentials used by IdentityServer in the auth service, that has the exact same problem as what we've seen.

For starters, if we were actually deploying this, we would need to have actual signing credentials, which we would store safely somewhere. Even so, if we would like to use the development signing credentials in a Docker development environment, we could use the exact same strategy - create a volume, map to a folder in the container and configure it, something like:

`Auth\src\CodingMilitia.PlayBall.Auth.Web\IoC\IdentityServerExtensions.cs`
```csharp
// ...
if (environment.IsDevelopment() || environment.IsEnvironment("DockerDevelopment"))
{
    builder.AddDeveloperSigningCredential(
        filename: configuration
            .GetSection<SigningCredentialSettings>(nameof(SigningCredentialSettings))
            .DeveloperCredentialFilePath);
}
else
{
    throw new Exception("need to configure key material");
}
// ...
```

## Outro
That's all for this episode. We took a quick look at ASP.NET Core data protection, not from the point of view of using the APIs (maybe something to do in the future), but from a configuration point of view, as even if we're not using it directly, the framework certainly is, and it is crucial for our application to run smoothly.

Links in the post:
- [ASP.NET Core Data Protection](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/introduction?view=aspnetcore-2.2)
- [Data Protection key management and lifetime in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/default-settings?view=aspnetcore-2.2)
- [Key storage providers in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/implementation/key-storage-providers?view=aspnetcore-2.2)
- [Key encryption at rest in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/implementation/key-encryption-at-rest?view=aspnetcore-2.2)
- [OpenID Connect - authorization code flow - authentication Request](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest)

The source code for this post is spread across the repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode028`.
- [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode028)
- [WebFrontend](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode028)
- [Deployment](https://github.com/AspNetCoreFromZeroToOverkill/Deployment/tree/episode028)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!