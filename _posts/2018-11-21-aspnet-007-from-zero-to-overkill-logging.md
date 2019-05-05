---
author: João Antunes
date: 2018-11-21 19:30:00+00:00
layout: post
title: "Episode 007 - Logging - ASP.NET Core: From 0 to overkill"
summary: "In this episode we take a look at logging in ASP.NET Core, the builtin support and providers, as well as using third party implementations, namely NLog."
image: '/assets/2018/11/21/aspnet-core-from-zero-to-overkill-e007.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
---

In this episode we take a look at logging in ASP.NET Core, the builtin support and providers, as well as using third party implementations, namely NLog. 

For the walk-through you can check the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/jEx2PgUkLAE" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
Logging is one of the basics when creating an application, as it allows us to have a glimpse of its behavior, not only in terms of problems it might run into, but also some information about what's going on.

So far in this project, our "logging" has basically been `Console.WriteLine`, which really isn't a very good idea, as we want more control over logging behavior, like log levels and providers.

So let's add some real logging to the project.

## Using the builtin logging facilities
### Setting up logging
Pretty similar to what we saw in the previous episode regarding configuration, the first thing we need to do is configure the logging providers we want to use (or as we'll see, not really... again! 😛).

Like we did for configuration, we go into the `Program` class and where we're creating the `IWebHostBuilder`, we can call the `ConfigureLogging` method to setup the providers we want.

Going back to (by now well known) `WebHost`'s `CreateDefaultBuilder` method, we can see it already configures some logging providers by default.

```csharp
//...
.ConfigureLogging((hostingContext, logging) =>
{
    logging.AddConfiguration(hostingContext.Configuration.GetSection("Logging"));
    logging.AddConsole();
    logging.AddDebug();
})
//...
```
(complete source on [GitHub](https://github.com/aspnet/MetaPackages/blob/975011071b02f6937261c604fcde48d7e43030ce/src/Microsoft.AspNetCore/WebHost.cs#L181), for ASP.NET Core 2.1)

By looking at the code we now know that:
- Configuration in the logging section will be used if existent (we'll take a look at this in a bit)
- Console and debug providers are pre-configured

In this post we'll mostly stick to the console (and later on, a file), so the default builder's configuration is good to go.

Now lets replace the `Console.WriteLine` calls we have in our code.

### Injecting ILogger<T>
To log stuff, we must get an `ILogger<T>` - or use an `ILoggerFactory<T>`, but we'll stick with `ILogger<T>` for now - in the classes where we want to log. So if, for example, in the `GroupsController` we wanted to add logs, we would add an `ILogger<GroupsController>` and store it in a private field for usage in the actions. Right now, where we have already something being written to the console is in the `GroupsServiceDecorator`, created in the `AutofacModule`, so let's take a look at that.

`AutofacModule.cs`
```csharp
public class AutofacModule : Module
{
    protected override void Load(ContainerBuilder builder)
    {
        builder.RegisterType<InMemoryGroupsService>().Named<IGroupsService>("groupsService").SingleInstance();
        builder.RegisterDecorator<IGroupsService>((context, service) => new GroupsServiceDecorator(service, context.Resolve<ILogger<GroupsServiceDecorator>>()), "groupsService");
    }

    private class GroupsServiceDecorator : IGroupsService
    {
        private readonly IGroupsService _inner;
        private readonly ILogger<GroupsServiceDecorator> _logger;

        public GroupsServiceDecorator(IGroupsService inner, ILogger<GroupsServiceDecorator> logger)
        {
            _inner = inner;
            _logger = logger;
        }
            
        public IReadOnlyCollection<Group> GetAll()
        {
            _logger.LogWarning($"######### Helloooooo from {nameof(GetAll)} #########", );
            return _inner.GetAll();            
        }

        public Group GetById(long id)
        {
            _logger.LogWarning($"######### Helloooooo from {nameof(GetById)} #########");
            return _inner.GetById(id);
        }

        public Group Update(Group group)
        {
            _logger.LogWarning($"######### Helloooooo from {nameof(Update)} #########");
            return _inner.Update(group);
        }

        public Group Add(Group group)
        {
            _logger.LogWarning($"######### Helloooooo from {nameof(Add)} #########");
            return _inner.Add(group);
        }
    }
}
```

Rather simple stuff. In `GroupsServiceDecorator` we get an `ILogger<GroupsServiceDecorator>`, then use it in the methods. Just needed to add the dependency resolution in the `builder.RegisterDecorator` above to pass the logger along. 

If we run the application now, we get mostly the same we had, but with some extra info that comes from using a logger:
```
warn: CodingMilitia.PlayBall.GroupManagement.Web.IoC.AutofacModule.GroupsServiceDecorator[0]
      ######### Helloooooo from GetById #########
```

### Structured logging
Although we won't get much advantage out of it right now, one thing we can do to improve the output of our logs is to make it structured logging friendly.

Structured logging (or semantic logging) means that besides the log message, some metadata is attached to each log, which may help in getting the most out of the logs, without lots of regular expressions or similar voodoos to extract the info needed. Let's take the previous example of the `GroupsServiceDecorator`, what if instead of just logging the message as is now, we use a provider that stores a JSON for each log, and besides the message, it stores the name of the decorated method with its semantic meaning, something like:

```json
{
    "message": "######### Helloooooo from GetAll #########",
    "data": {
        "decoratedMethod": "GetAll"
    }
}
```

This would make it easier to find by decorated method name for example, count the number of times it was called and other things like that.

To achieve this, we replace the string interpolation used previously with the formatting capabilities provided by the logger's methods:

`AutofacModule.cs`
```csharp
//...
private class GroupsServiceDecorator : IGroupsService
{
    //...
        
    public IReadOnlyCollection<Group> GetAll()
    {
        _logger.LogWarning("######### Helloooooo from {decoratedMethod} #########", nameof(GetAll));
        return _inner.GetAll();            
    }

    public Group GetById(long id)
    {
        _logger.LogWarning("######### Helloooooo from {decoratedMethod} #########", nameof(GetById));
        return _inner.GetById(id);
    }

    public Group Update(Group group)
    {
        _logger.LogWarning("######### Helloooooo from {decoratedMethod} #########", nameof(Update));
        return _inner.Update(group);
    }

    public Group Add(Group group)
    {
        _logger.LogWarning("######### Helloooooo from {decoratedMethod} #########", nameof(Add));
        return _inner.Add(group);
    }
}
//...
```

Now, when in the future we configure the logger to output to someplace that supports it, we'll have some better structured data to work with.

### Changing log levels
So far the logs we're creating are all warnings, but probably, for the kind of logs they are, they're level should probably be `Trace` or something like that. Haven't used this yet because the default log level is `Information`, so a `Trace` wouldn't be shown. Let's change that.

In the `appsettings.development.json`, let's add a section for logging, where we can configure 

```json
{
  "_commentWannabe_": "...",

  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Warning",
      "CodingMilitia.PlayBall.GroupManagement.Web.IoC": "Trace"
    }
  }
}
```
We can immediately see that not only we define a log level, but we can define it per namespace. So, the default log level will be `Debug`, but namespaces starting with `System` will only log `Information`, namespaces starting with `Microsoft` only allow `Warning` or above and, finally, for the namespace `CodingMilitia.PlayBall.GroupManagement.Web.IoC` (where the decorator exists) everything is logged, having `Trace` as minimum level.

Now we can replace our `LogWarning` with `LogTrace` and see it still showing up on the console.

```
trce: CodingMilitia.PlayBall.GroupManagement.Web.IoC.AutofacModule.GroupsServiceDecorator[0]
      ######### Helloooooo from GetById #########
```

### Logging scopes
Another interesting feature we can use when logging are scopes. Scopes allow us to group a set of log calls, to which we can attach some data that'll be always part of the log, without the need to add that information in every log invocation.

For us to see scopes in action, we need to enable then in the configuration for the console provider:
`appsettings.development.json`
```json
{
  "_commentWannabe_": "...",

  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Warning",
      "CodingMilitia.PlayBall.GroupManagement.Web.IoC": "Trace"
    },
    "Console":
    {
      "IncludeScopes": true
    }
  }
}
```
Just by doing this, we can see a difference when looking at the console logs, because ASP.NET Core makes use of this functionality:
```
trce: CodingMilitia.PlayBall.GroupManagement.Web.IoC.AutofacModule.GroupsServiceDecorator[0]
      => ConnectionId:0HLIF66I6VVM5 => RequestId:0HLIF66I6VVM5:00000001 RequestPath:/groups/1 => CodingMilitia.PlayBall.GroupManagement.Web.Controllers.GroupsController.Details (CodingMilitia.PlayBall.GroupManagement.Web)
      ######### Helloooooo from GetById #########
```

Comparing with previous logs, we can see the extra couple of lines, with information about the request and controller that handles it.

Now let's try adding a scope of ours.

`AutofacModule.cs`
```csharp
public class AutofacModule : Module
{
    //...

    private class GroupsServiceDecorator : IGroupsService
    {
        //...
        
        public IReadOnlyCollection<Group> GetAll()
        {
            using (var scope = _logger.BeginScope("Decorator scope: {decorator}", nameof(GroupsServiceDecorator)))
            {
                _logger.LogTrace("######### Helloooooo from {decoratedMethod} #########", nameof(GetAll));
                var result = _inner.GetAll();
                _logger.LogTrace("######### Goodbyeeeee from {decoratedMethod} #########", nameof(GetAll));
                return result;
            }
        }
        
        //...
    }
}
```

Using the `GetAll` method as an example, we create the scope using `_logger.BeginScope` inside a `using` statement, ensuring that only the logs created inside of it make use of the scope, including logs that are not directly inside the `using`, but in code invoked there. 

The scope might have a message as well as extra info associated with it, that'll go along with every log in its context.

Also added another `_logger.LogTrace` to the method, so we can see that both take the scope's information along with them. The console output turns out as follows.

```
trce: CodingMilitia.PlayBall.GroupManagement.Web.IoC.AutofacModule.GroupsServiceDecorator[0]
      => ConnectionId:0HLIF66I6VVM6 => RequestId:0HLIF66I6VVM6:00000001 RequestPath:/groups => CodingMilitia.PlayBall.GroupManagement.Web.Controllers.GroupsController.Index (CodingMilitia.PlayBall.GroupManagement.Web) => Decorator scope: GroupsServiceDecorator
      ######### Helloooooo from GetAll #########
trce: CodingMilitia.PlayBall.GroupManagement.Web.IoC.AutofacModule.GroupsServiceDecorator[0]
      => ConnectionId:0HLIF66I6VVM6 => RequestId:0HLIF66I6VVM6:00000001 RequestPath:/groups => CodingMilitia.PlayBall.GroupManagement.Web.Controllers.GroupsController.Index (CodingMilitia.PlayBall.GroupManagement.Web) => Decorator scope: GroupsServiceDecorator
      ######### Goodbyeeeee from GetAll #########
```

## Using third party loggers
Just as is the case for dependency injection, where we can use third party containers, we can also use third party logging libraries that may have some features we don't get out of the box.

To see how we can use third party loggers, we'll add [NLog](https://nlog-project.org/) to our project, and replace the builtin logging providers.

### Remove builtin providers and add NLog
To get the NLog dependency on our project, `dotnet add package NLog.Web.AspNetCore` does the trick.

To replace the default logger with NLog, we head onto the `Program` class and make a couple of changes to the `CreateWebHostBuilder` method.

`Program.cs`
```csharp
public class Program
{
    //...
    
    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .ConfigureLogging(builder =>
            {
                builder.ClearProviders(); //clear default providers, NLog will handle it
                builder.SetMinimumLevel(Microsoft.Extensions.Logging.LogLevel.Trace); //set minimum level to trace, NLog rules will kick in afterwards
            })
            .UseNLog()
            .UseStartup<Startup>();
    //...
}
```

As we see above, we use `ConfigureLogging` so we can clear the providers `WebHost.CreateDefaultBuilder` added - we could keep them to use along with NLog, but that's maybe a little more confusing than letting NLog handle everything. We also set the minimum log level to trace, so everything goes through to NLog, so its targets (we'll see what they are in a moment) will handle logging only the things that should be logged.

Additionally, we must call `UseNLog` on the `WebHostBuilder` to setup NLog.

### Configure NLog
Now to configure NLog! Ideally we would create an `nlog.config` file, but I'm just going to configure everything programmatically right now. For the recommended way to configure everything, check out [NLog's documentation](https://github.com/NLog/NLog.Web/wiki/Getting-started-with-ASP.NET-Core-2).

To do this, I added a `ConfigureNLog` method to the `Program` class that's called at the start of the application. In this method we setup NLog's targets, which are like the ASP.NET Core logging providers, as we can set things like the console and files, specific log levels for each target, as well as specific log layouts for each.

`Program.cs`
```csharp
public class Program
{
    public static void Main(string[] args)
    {
        Console.WriteLine("############### Starting application ###############");
        ConfigureNLog();
        CreateWebHostBuilder(args).Build().Run();
    }
    
    //...
    
    //TODO: replace with nlog.config
    private static void ConfigureNLog()
    {
        var config = new LoggingConfiguration();

        var consoleTarget = new ColoredConsoleTarget("coloredConsole")
        {
            Layout = @"${date:format=HH\:mm\:ss} ${level} ${message} ${exception}"
        };
        config.AddTarget(consoleTarget);
        
        var fileTarget = new FileTarget("file")
        {
            FileName = "${basedir}/file.log",
            Layout = @"${date:format=HH\:mm\:ss} ${level} ${message} ${exception} ${ndlc}"
        };
        config.AddTarget(fileTarget);
        
        config.AddRule(LogLevel.Trace, LogLevel.Fatal, consoleTarget, "CodingMilitia.PlayBall.GroupManagement.Web.IoC.*");
        config.AddRule(LogLevel.Info, LogLevel.Fatal, consoleTarget);
        config.AddRule(LogLevel.Warn, LogLevel.Fatal, fileTarget);

        LogManager.Configuration = config;
    }
}
```

I'd say that just by looking at the code above you can easily understand what's going on. A couple of targets are created, console and file, each with different message layouts and log level rules, including (like we saw earlier for the builtin providers) making the log levels depend also on the namespace of the class that's logging.

In regards to the layouts, most of the options used are pretty easy to understand, apart from `ndlc`. `ndlc` (which means "Nested Diagnostics Logical Context") is the way to include the scope we talked earlier in the message.

A note on the `FileTarget` - I really didn't configure it as much as I should. For instance, defining when a new log file should be created (every new day, when it reaches a certain size, when the application restarts, etc) or when old ones are deleted is probably a good idea.

With this in place, we have everything working mostly as before, but having NLog in charge of the logs. The code that uses the logger doesn't need any change. The only noticeable differences are the amount of messages we see, as I didn't configure the log levels in exactly the same way and the layout of the message is also a bit different (and have colors because we used the `ColoredConsoleTarget`).

Console log sample
```
18:38:57 Warn ######### Helloooooo from GetById #########
```

File log sample
```
18:38:57 Warn ######### Helloooooo from GetById #########  ConnectionId:0HLIFS2291JCR RequestId:0HLIFS2291JCR:00000003 RequestPath:/groups/1 CodingMilitia.PlayBall.GroupManagement.Web.Controllers.GroupsController.Details (CodingMilitia.PlayBall.GroupManagement.Web)
```

## Outro
That's a wrap for another episode on the ASP.NET Core: From 0 to Overkill series. As usual, this is not everything there is to know about the topic, logging in this case, but it's another stepping stone in our path to over-engineering 😆

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode007).

Please send any feedback so I can improve and adjust the next episodes.

Thanks for stopping by, cyaz!