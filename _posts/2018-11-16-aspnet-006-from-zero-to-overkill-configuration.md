---
author: João Antunes
date: 2018-11-16 19:00:00+00:00
layout: post
title: "Episode 006 - Configuration - ASP.NET Core: From 0 to overkill"
summary: "In this episode we take a look at configuration in ASP.NET Core, the possible sources, how to read from them, the options pattern, wrapping up with development time secrets."
image: '/assets/2018/11/16/aspnet-core-from-zero-to-overkill-e006.jpg'
categories:
- fromzerotooverkill
tags:
- dotnet
- aspnetcore
---

In this episode we take a look at configuration in ASP.NET Core, the possible sources, how to read from them, the options pattern, wrapping up with development time secrets.

For the walk-through you can check the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/2x6tMPcBJqY" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In ASP.NET Core the way configuration is managed really changed when comparing with ASP.NET, where we had the `web.config` file (and others with the same format). In this post, we'll take a look at this new configuration, and see how much more powerful it is.

For much more in depth information about this topic, be sure to check out the docs [over here](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-2.1).

## Configuration providers
One of the great things introduced with ASP.NET Core configuration, is the ability to have multiple configuration providers, but being able to then use those configurations in the code in a consistent manner, regardless of their source.

So for instance, we can have configurations done through environment variables, command-line arguments and a JSON file, and then use them as if they all came from the same place.

(Copied from the docs) we currently have the following providers available:

| Provider | Provides configuration from |
| -------- | ----------------------------------- |
| Azure Key Vault Configuration Provider | Azure Key Vault |
| Command-line Configuration Provider | Command-line parameters |
| Custom configuration provider | Custom source |
| Environment Variables Configuration Provider | Environment variables |
| File Configuration Provider | Files (INI, JSON, XML) |
| Key-per-file Configuration Provider | Directory files |
| Memory Configuration Provider | In-memory collections |
| User secrets (Secret Manager) | File in the user profile directory |

As you can see in the table above, we can also create our own provider, to access a different configuration format or location, but that's out of scope for this post, we'll just play a bit with some of the existing providers.

To provide a glimpse of what we can do with ASP.NET Core's configuration infrastructure, in this post we'll play a little bit with the command-line, file and user secrets providers.

### Configuring providers
The first thing we need to do is configure the configuration providers we want to use (or as we'll see, not really 😛). 

To configure the providers, we go into the `Program` class and where we're creating the `IWebHostBuilder`, we can call the `ConfigureAppConfiguration` method to setup the providers we want.

As we saw in previous episodes, the `WebHost` `CreateDefaultBuilder` method already comes with a lot of things setup out of the box, and configuration providers are one of them. If we take a look at the method's source, we can see what's already there.

```csharp
//...
.ConfigureAppConfiguration((hostingContext, config) =>
{
    var env = hostingContext.HostingEnvironment;

    config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
            .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: true);

    if (env.IsDevelopment())
    {
        var appAssembly = Assembly.Load(new AssemblyName(env.ApplicationName));
        if (appAssembly != null)
        {
            config.AddUserSecrets(appAssembly, optional: true);
        }
    }

    config.AddEnvironmentVariables();

    if (args != null)
    {
        config.AddCommandLine(args);
    }
})
//...
```
(complete source on [GitHub](https://github.com/aspnet/MetaPackages/blob/62d9794c633a9a2c502334d525d81c454ac29264/src/Microsoft.AspNetCore/WebHost.cs#L165))

By looking at the source, we can see that 5 providers are configured:
- 2 for JSON files, the first one named `appsettings.json` and the other the same plus the name of the current environment
- a user secrets provider, only when in a development environment
- environment variables provider
- command line arguments provider

An important thing to point out is that the order in which the providers are configured is relevant. An added provider overrides any previous one's configuration with the same key. For instance, if we set a value in `appsettings.json` but then we want a different one in development mode, we simply put a configuration with the same key in `appsettings.development.json`. Likewise for other providers.

Given these providers are already setup, for this post's objective, no need to configure anything else.

## Creating some configurations
Before we see how to use the configurations in the code, it's probably a good idea to start by creating some 😀

Let's start by creating a couple of JSON files in the root of the project - `appsettings.json` and `appsettings.development.json` - and add the following contents to them.

`appsettings.json`
```json
{
  "SomeRoot": {
    "SomeSubRoot": {
      "SomeKey": 12345,
      "AnotherKey": "QWERTY"
    }
  }
}
```

`appsettings.development.json`
```json
{
  "SomeRoot": {
    "SomeSubRoot": {
      "SomeKey": 67890
    }
  }
}
```

To pass in some configurations through the command line we can do `dotnet run -- --SomeRoot:SomeSubRoot:CmdLineKey 13579`. The colon is used to define a hierarchy, so the equivalent in JSON would be:
```json
{
  "SomeRoot": {
    "SomeSubRoot": {
      "CmdLineKey": 13579
    }
  }
}
```

If you're using an IDE to run the application, there's probably a menu somewhere that lets you pass in command line arguments. 
- In Visual Studio 2017 you would -> right click project -> "Properties" -> "Debug" -> "Application Arguments" (note that this doesn't work when using IIS Express)
- In JetBrains Rider, which is what I'm using, you go into the run configurations, and add the arguments into the "Program arguments" input

Now that we have some configurations to access, let's get to it.

## Accessing configuration
Let's start with the simplest (but not great) way to access the configuration, using `IConfiguration`.

In our `Startup` class, or even in our controllers or services, we can get an `IConfiguration` injected, and use it to access the configurations. Then we can, for instance, do the following to get some values:

```csharp
config.GetValue<int>("SomeRoot:SomeSubRoot:SomeKey"); //returns 67890
config.GetValue<int>("SomeRoot:SomeSubRoot:CmdLineKey"); //returns 13579
//or
var section = config.GetSection("SomeRoot:SomeSubRoot");
section.GetValue<string>("AnotherKey"); //returns "QWERTY"
```

Even though this works, and for really simple stuff may be enough, it's not great, and spreading strings like `"SomeRoot:SomeSubRoot:SomeKey"` around the application isn't very nice (even if we put it in constants). 

What would be much nicer than this way of accessing our configurations, would be to have classes that represent them, providing a much cleaner and type safe way to access those values. To solve this problem, enter the options pattern.

## Options pattern
The options pattern introduced in ASP.NET Core allows us to easily bind the configurations to POCOs. To map what we setup previously, we create a couple of classes:

`SomeRootConfiguration.cs`
```csharp
public class SomeRootConfiguration
{
    public SomeSubRootConfiguration SomeSubRoot { get; set; }
}
```

`SomeSubRootConfiguration.cs`
```csharp
public class SomeSubRootConfiguration
{
    public int SomeKey { get; set; }
    public string AnotherKey { get; set; }
    public int CmdLineKey { get; set; }
}
```

Then to bind the configurations to these classes, we can add the following line to `Startup`'s `ConfigureServices` method:

`Startup.cs`
```csharp
public class Startup
{
    private readonly IConfiguration _config;

    public Startup(IConfiguration config)
    {
        _config = config;
    }

    public IServiceProvider ConfigureServices(IServiceCollection services)
    {
        //...
        services.Configure<SomeRootConfiguration>(_config.GetSection("SomeRoot"));
        //...
    }
    //...
}
```

Then to use it, we can go into the `GroupsController` and add `IOptions<SomeRootConfiguration>` to use our nicely structured configuration class. Alternatively, we can inject `IOptionsSnapshot<SomeRootConfiguration>`, which allows the application to load configurations changed at runtime.

## Avoiding IOptions injection
Even though the options pattern is pretty great improvement to what we saw before it, it can be further improved.

Unless for the reloading capabilities, injecting `IOptions` isn't really useful, and adds an extra dependency that's really not required. When injecting into controllers, this extra `using` isn't a big deal, as we're already in ASP.NET Core MVC land. But what about injecting it into other services/classes? Forcing a dependency just to inject configurations in other libraries, that may be used in a non ASP.NET Core application (e.g. a console application, a web application developed using a different web framework) is a harder pill to swallow.

Luckily, it's an easy problem to solve. For a more detailed discussion on this you can check [this excellent article](https://www.strathweb.com/2016/09/strongly-typed-configuration-in-asp-net-core-without-ioptionst/), but I'll quickly walk through a solution here.

Instead of using the `services.Configure` method, we can bind the configuration directly to the class, and then inject it as a singleton. We can do that as follows:

`Startup.cs`
```csharp
public IServiceProvider ConfigureServices(IServiceCollection services)
{
    //...    
    var someRootConfiguration = new SomeRootConfiguration();
    _config.GetSection("SomeRoot").Bind(someRootConfiguration);
    services.AddSingleton(someRootConfiguration); 
    //...
}
```

This solves our problem and now, in the controller, we can simply inject a `SomeRootConfiguration`, no `IOptions` needed.

To avoid repeating this code for all types of configurations we want to add, we can create an extension method to handle this logic.

`ServiceCollectionExtensions.cs`
```csharp
public static class ServiceCollectionExtensions
{
    //...
    
    public static TConfig ConfigurePOCO<TConfig>(this IServiceCollection services, IConfiguration configuration) 
      where TConfig : class, new()
    {
        if (services == null) throw new ArgumentNullException(nameof(services));
        if (configuration == null) throw new ArgumentNullException(nameof(configuration));

        var config = new TConfig();
        configuration.Bind(config);
        services.AddSingleton(config);
        return config;
    }
}
```

And we can replace the previous configuration registration with:

`Startup.cs`
```csharp
public IServiceProvider ConfigureServices(IServiceCollection services)
{
    //...    
    services.ConfigurePOCO<SomeRootConfiguration>(_config.GetSection("SomeRoot"));
    //...
}
```

Which now looks pretty much like the usual configuration registration, but without the `IOptions` burden.

## User secrets
Another interesting configuration provider we should take a look at is the user secrets provider.

The user secrets provider is used during development to keep secrets from ending up in source control, like API keys, connection strings and things like that. In production you would probably use other means of providing those kinds of configurations, like environment variables or services to keep secrets (like Azure Key Vault).

User secrets are not encrypted, they're simply stored in a different path to avoid adding them to source control by accident. For a lot more info on user secrets, head on to the [docs](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-2.1).

Before adding secrets to the application, we need to add an identifier to it, so when starting the application can fetch from the secrets store the correct configurations. To do this, head into the application's `csproj` and add a GUID inside an `UserSecretsId` element below the `TargetFramework`.

In this case, we go into `CodingMilitia.PlayBall.GroupManagement.Web.csproj` and do:
```xml
<!-- ... -->
<PropertyGroup>
    <TargetFramework>netcoreapp2.1</TargetFramework>
    <UserSecretsId>A1CD619E-A374-4208-888B-42F3DC489F14</UserSecretsId>
</PropertyGroup>
<!-- ... -->
```

Now we can add some secrets by doing `dotnet user-secrets set "DemoSecrets:SomeKey" "02468"`.

To access the configured secrets, we do exactly the same as in the other types of providers, so I just created another POCO and configured it like we saw in the previous section.

`DemoSecretsConfiguration.cs`
```csharp
public class DemoSecretsConfiguration
{
    public int SomeKey { get; set; }
}
```

`Startup.cs`
```csharp
public IServiceProvider ConfigureServices(IServiceCollection services)
{
    //...    
    services.ConfigurePOCO<DemoSecretsConfiguration>(_config.GetSection("DemoSecrets"));
    //...
}
```

Pretty neat I would say 😀

## Outro
That's a wrap for this intro to ASP.NET Core configuration. As usual, there's a lot more to explore, but I think this covers the bases. Be sure to head on to the docs for a lot more information (the documentation for ASP.NET Core is pretty great in general).

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode006).

Please send any feedback you have, so the next posts/videos can be better and even adjusted to more interesting topics.

Thanks for stopping by, cyaz!

<br/>

___
### Linked articles

[Strongly typed configuration in ASP.NET Core without IOptions&lt;T&gt;](https://www.strathweb.com/2016/09/strongly-typed-configuration-in-asp-net-core-without-ioptionst/)
