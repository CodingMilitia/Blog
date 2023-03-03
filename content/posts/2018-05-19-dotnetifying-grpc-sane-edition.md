---
author: João Antunes
date: 2018-05-19 12:00:00+01:00
layout: post
title: 'Dotnetifying gRPC: Sane Edition'
summary: 'Going about the usual way of implementing a gRPC service in .NET, creating some helpers to make it a more ASP.NET Core like experience.'
images:
- '/images/2018/05/19/csharp-grpc.jpg'
categories:
- dotnet
tags:
- dotnet
- grpc
- services
slug: dotnetifying-grpc-sane-edition
---

{{< embedded-image "/images/2018/05/19/csharp-grpc.jpg" "C# + gRPC" >}}

So last time I played around with gRPC (and wrote about it) I went a little bit insane with a different way of implementing and invoking gRPC services in .NET (you can take a look at it [here]({{< ref "2018-04-15-dotnetifying-grpc " >}})). Although it was a fun experiment and I could even see it being used, the reality is it’s better to stick closer to the official and supported way of doing things. With that in mind, this time I went on the completely opposite direction, so it’ll work more as a guide of how to integrate gRPC in a .NET application. I built a library out of this, but it’s really so simple, it’ll be mostly for me to use, as others will probably avoid adding an extra external dependency for not so much added value.

The accompanying code for this post is [here](https://github.com/CodingMilitia/GrpcExtensions/tree/may-blog-post).

## Quick gRPC service initial setup walk-through

Maybe it’s a bit of a repetition of my previous post on gRPC (and others around the internets) but to add a little more context to the rest of the post (and I don’t have to jump around posts when I can’t remember what I’ve done), I’ll start with how to go about the initial setup of a gRPC service. 

### Service definition
The first thing to do is defining the service, much like one would create a WSDL before implementing a SOAP service (I know, the usual way is to implement the service and get the generated contract afterwards, but it shouldn’t!), in gRPC we create a proto file (or more) with the service definition - its methods, input and output messages.

```protobuf
syntax = "proto3";

option csharp_namespace 
    = "CodingMilitia.GrpcExtensions.ScopedRequestHandlerSample.Generated";

service SampleService {
    rpc Send(SampleRequest) returns (SampleResponse) {}
}

message SampleRequest {
    int32 value = 1;
}

message SampleResponse {
    int32 value = 1;
}
```

So, here I’m defining a service called `SampleService` with a method `Send` that receives a `SampleRequest` parameter and returns a `SampleResponse` - such naming imagination right?
This is a unary method definition, simple request response. In gRPC we have some streaming options, but like I said, I'm keeping it simpler for now, next time I'll play around with streaming.
Finally, it’s rather obvious but, the `csharp_namespace` option indicates the namespace to which the generated classes will belong to.

### Code generation
With the service definition done, we need to generate the code to use in our applications. The tooling for the code generation is installed along with the `Grpc.Tools` NuGet package. With the tooling in place, for this simple case I’m describing, it’s just a command away from getting the required client and base server code,

```bash
./protoc service.proto --csharp_out ./Generated/. --grpc_out ./Generated/. --plugin=protoc-gen-grpc=grpc_csharp_plugin
```

With our code generated, we can move on to using it.

## Client
On the client side of things I really didn’t add anything to the library as the generated one really doesn’t require a lot of configuration.

One of the generated classes for our sample is the `SampleServiceClient`. To use it in an application, for instance a web application, we just need to configure it in the `Startup` class as following.

```csharp
services.AddSingleton(_ =>
{
    var channel = new Channel(
        "127.0.0.1:5050", 
        ChannelCredentials.Insecure
    );
    return new SampleServiceClient(channel);
});
```

Of course the hardcoded url and credentials configuration part is good only for sample purposes.

## Server
For the server side of things is where I really felt the need to create something to help with hosting the service. Before looking into hosting though, let’s first take a look at some required considerations on implementing the service logic.

### Implementing the service

The service implementation is the same for the most part, as we must inherit from a generated base service class, in this case `SampleServiceBase`. Now the problem with implementing our logic directly in the class that inherits from the generated one is that it will live as a singleton class, so if we want to have scoped dependencies we’ll need to bring the dependency injection stuff into the class, making it more of a service locator thing, which isn’t as nice.

So to reduce the need fiddle around with DI when we should be focusing on the service’s logic, the best way is probably to implement the service logic in another class, which in the simplest case (like this unary method case we’re using as sample) doesn’t even need to know anything about gRPC.

```csharp
public class RandomSampleServiceLogic : ISampleServiceLogic
{
    private static readonly Random RandomSource = new Random();
    private readonly ILogger<RandomSampleServiceLogic> _logger;

    public RandomSampleServiceLogic(
        ILogger<RandomSampleServiceLogic> logger
    )
    {
        _logger = logger;
    }
    public async Task<SampleResponse> SendAsync(
        SampleRequest request, 
        CancellationToken ct
    )
    {
        _logger.LogInformation(
            "Received request with value {requestValue}",
            request.Value
        );
        
        _logger.LogInformation(
            "Simulating slow operation with a delay for request value {requestValue}", 
            request.Value
        );

        await Task.Delay(1000, ct);

        var response = new SampleResponse
        {
            Value = request.Value + RandomSource.Next()
        };

        _logger.LogInformation(
            "Random response to request with value {requestValue} will be {responseValue}",
            request.Value,
            response.Value
        );

        return response;
    }
}
```

Notice the `CancellationToken` is passed as an argument to `SendAsync` method, as in this simple case it is completely oblivious of being part of a gRPC service, and if needed could be used as is as the logic provider of a REST API or any other type of API.

Then the service implementation to be hosted can be something like the following.

```csharp
public class AnotherSampleServiceImplementation : SampleServiceBase
{
    private readonly IServiceScopeFactory _scopeFactory;

    public AnotherSampleServiceImplementation(
        IServiceScopeFactory scopeFactory
    )
    {
        _scopeFactory = scopeFactory;
    }

    public override async Task<Generated.SampleResponse> Send(
        Generated.SampleRequest request, 
        ServerCallContext context
    )
    {
        using (var scope = _scopeFactory.CreateScope())
        {
            var service = scope.ServiceProvider
                .GetRequiredService<ISampleServiceLogic>();

            var response = await service.SendAsync(
                request.ToInternalRequest(), 
                context.CancellationToken
            );

            return response.ToExternalResponse();
        }
    }
}
```

Alternatively I created a `IScopedExecutor` that can be used to abstract away this, but in reality there’s not that much need for it.

```csharp
public class SampleServiceImplementation : SampleServiceBase
{
    private readonly IScopedExecutor _scopedExecutor;

    public SampleServiceImplementation(
        IScopedExecutor scopedExecutor
    )
    {
        _scopedExecutor = scopedExecutor;
    }

    public override async Task<Generated.SampleResponse> Send(
        Generated.SampleRequest request, 
        ServerCallContext context
    )
    {
        return await _scopedExecutor
            .ExecuteAsync<ISampleServiceLogic, Generated.SampleResponse>(
                async (service) =>
                {
                    var response = await service.SendAsync(
                        request.ToInternalRequest(), 
                        context.CancellationToken);
                        
                    return response.ToExternalResponse();
                });
    }
}
```

### Hosting

Now that we have the service implemented, we can host it. As I mentioned, this is where I really felt I could put something up to simplify things.

To start with, how to host the service? The simpler way is probably just to instantiate the `Server` class, configure it and call the `Start` method on it. The problem with this approach is if we want to use DI, we need to set it all up by hand. A good alternative to this, taking advantage of what’s already done for us, is implementing a background service, using the `IHostedService` interface provided with `Microsoft.Extensions.Hosting` and then registering it with a `HostBuilder` (much like the usual `WebHostBuilder` we use in ASP.NET Core applications).

So with this in mind I started by implementing the `GrpcBackgroundService` class. This is a pretty simple class, receiving the `Server` instances to host in the constructor, so only one `GrpcBackgroundService` instance is needed even if we want to host multiple services in multiple `Server`s (we can also host multiple services in a single `Server`).

```csharp
internal class GrpcBackgroundService : IHostedService
{
    private readonly IEnumerable<Server> _servers;
    private readonly ILogger<GrpcBackgroundService> _logger;

    public GrpcBackgroundService(
        IEnumerable<Server> servers, 
        ILogger<GrpcBackgroundService> logger
    )
    {
        _servers = servers;
        _logger = logger;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogDebug("Starting gRPC background service");

        foreach(var server in _servers)
        {
            StartServer(server);
        }

        _logger.LogDebug("gRPC background service started");

        return Task.CompletedTask;
    }



    public async Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogDebug("Stopping gRPC background service");

        var shutdownTasks = _servers
            .Select(server => server.ShutdownAsync()).ToList();
        
        await Task.WhenAll(shutdownTasks).ConfigureAwait(false);

        _logger.LogDebug("gRPC background service stopped");
    }

    private void StartServer(Server server)
    {
        _logger.LogDebug(
            "Starting gRPC server listening on: {hostingEndpoints}",
            string.Join(
                "; ", 
                server.Ports.Select(p => $"{p.Host}:{p.BoundPort}")
            )
        );

        server.Start();

        _logger.LogDebug(
            "Started gRPC server listening on: {hostingEndpoints}",
            string.Join(
                "; ", 
                server.Ports.Select(p => $"{p.Host}:{p.BoundPort}")
            )
        );
    }
}
```

Then to assist in registering the services I created some extensions methods for the `IServiceCollection` interface. Below I included only the ones I find most interesting for this scenario, as the others do little more than registering the provided `Server` to DI along with the `GrpcBackgroundService`.

```csharp
//...

public static IServiceCollection AddGrpcServer<TService>(
    this IServiceCollection serviceCollection,
    IEnumerable<ServerPort> ports,
    IEnumerable<ChannelOption> channelOptions = null
)
    where TService : class
{
    return serviceCollection.AddGrpcServer(
            serverConfigurator: 
                configurator => configurator.AddService<TService>(),
            ports: ports,
            channelOptions: channelOptions
        );
}

public static IServiceCollection AddGrpcServer(
    this IServiceCollection serviceCollection,
    Action<IGrpcServerBuilder> serverConfigurator,
    IEnumerable<ServerPort> ports,
    IEnumerable<ChannelOption> channelOptions = null
)
{
    if (serviceCollection == null)
    {
        throw new ArgumentNullException(nameof(serviceCollection));
    }

    if (serverConfigurator == null)
    {
        throw new ArgumentNullException(nameof(serverConfigurator));
    }

    if (ports == null)
    {
        throw new ArgumentNullException(nameof(ports));
    }

    if (!ports.Any())
    {
        throw new ArgumentException(
            message: "At least one port must be specified", 
            paramName: nameof(ports)
        );
    }

    var builder = new GrpcServerBuilder(
        serviceCollection, 
        ports, 
        channelOptions
    );
    serverConfigurator(builder);
    builder.AddServerToServiceCollection();
    serviceCollection
        .AddGrpcBackgroundServiceIfNotAlreadyRegistered();

    return serviceCollection;
}

//...
```

The bulk of the implementation is in the second version of the `AddGrpcServer` method I show, so the first one is only a simplified version that uses the second as base. 

Regarding the methods signature, the first one is a version that registers a single service (being it `TService`, passed as a generic argument), using the provided ports and channel options to configure the `Server`. The second version allows for the registration of multiple services in the same `Server`, getting a `Action<IGrpcServerBuilder> serverConfigurator` as argument with that goal. The `IGrpcServerBuilder` interface exposes an `AddService` method that allows for the multiple desired services to be registered.
With all the required dependencies, an instance of the `IGrpcServerBuilder` implementation (`GrpcServerBuilder`) is created and takes care of building a `Server` to be hosted by the application.

```csharp
internal class GrpcServerBuilder : IGrpcServerBuilder
{
    private readonly IServiceCollection _serviceCollection;
    private readonly IEnumerable<ServerPort> _ports;
    private readonly IEnumerable<ChannelOption> _channelOptions;
    private readonly List<ServiceRegistrationInfo> _registrationInfo;
    private bool _built;

    public GrpcServerBuilder(
        IServiceCollection serviceCollection, 
        IEnumerable<ServerPort> ports,
        IEnumerable<ChannelOption> channelOptions
    )
    {
        if (ports == null)
        {
            throw new ArgumentNullException(nameof(ports));
        }

        if (!ports.Any())
        {
            throw new ArgumentException(
                message: "At least one port must be specified", 
                paramName: nameof(ports)
            );
        }

        _serviceCollection = serviceCollection;
        _ports = ports;
        _channelOptions = 
            channelOptions ?? Array.Empty<ChannelOption>();
        _registrationInfo = new List<ServiceRegistrationInfo>();
        _built = false;

    }

    public IGrpcServerBuilder AddService<TService>() 
        where TService : class
    {
        ThrowIfServerAlreadyBuilt();

        var serviceType = typeof(TService);
        if (
            _serviceCollection.Any(
                s => s.ServiceType.Equals(serviceType)
            ) 
            || 
            _registrationInfo.Any(
                s => s.ServiceType.Equals(serviceType)
            )
        )
        {
            throw new InvalidOperationException(
                $"{typeof(TService).Name} is already registered in the container."
            );
        }

        _serviceCollection.AddSingleton<TService>();
        var serviceBinder = ServerBuildHelpers.GetServiceBinder<TService>();

        //Storing a lambda to use later, because this avoids 
        //reflection tricks later when we don't have access 
        //to the TService type so easily to invoke the binder.

        //Also, not invoking it immediately to keep it lazy.
        
        _registrationInfo.Add(
            new ServiceRegistrationInfo(
                serviceType, 
                appServices 
                    => serviceBinder(appServices.GetRequiredService<TService>())
            )
        );

        return this;
    }

    public void AddServerToServiceCollection()
    {
        ThrowIfServerAlreadyBuilt();

        if (_registrationInfo.Count == 0)
        {
            throw new InvalidOperationException(
                "Must add at least one service for a server to be created."
            );
        }

        _serviceCollection.AddSingleton(appServices =>
        {
            var server = _channelOptions.Count() > 0 
                ? new Server(_channelOptions) 
                : new Server();

            server.AddPorts(_ports);

            foreach (var serviceDefinition in _registrationInfo)
            {
                server.AddServices(
                    serviceDefinition
                        .ServiceDefinitionProvider(appServices)
                );
            }
            return server;
        });

        _built = true;
    }

    private void ThrowIfServerAlreadyBuilt()
    {
        if (_built)
        {
            throw new InvalidOperationException(
                "Server already build."
            );
        }
    }

    private class ServiceRegistrationInfo
    {
        public Type ServiceType { get; }
        public Func<IServiceProvider, ServerServiceDefinition> 
            ServiceDefinitionProvider { get; }

        public ServiceRegistrationInfo(
            Type serviceType, 
            Func<IServiceProvider, ServerServiceDefinition> 
                serviceDefinitionProvider
        )
        {
            ServiceType = serviceType;
            ServiceDefinitionProvider = serviceDefinitionProvider;
        }
    }
}
```

The `AddService` method makes some initial validity checks, registers the services in the DI container and stores some information to use when finally building the service. The most important part of this information to use when building the `Server` is what I called the `serviceBinder`, which is used to bind the service implementation to the `ServerServiceDefinition` that is registered to the `Server`. We can see this method in the generated `SampleService` class with the name `BindService`. Initially I was passing this method as an argument to the `AddGrpcServer` methods, but then I went ahead and created an helper to fetch this for me given the service implementation class.

```csharp
//Using reflection tricks and assumptions on the way 
//the core gRPC lib works so, if Google changes this, 
//it'll break and only be noticed at runtime :)
public static Func<TService, ServerServiceDefinition> GetServiceBinder<TService>()
{
    var serviceType = typeof(TService);
    var baseServiceType = GetBaseType(serviceType);
    var serviceDefinitionType = typeof(ServerServiceDefinition);

    var serviceContainerType = baseServiceType.DeclaringType;
    var methods = serviceContainerType
        .GetMethods(BindingFlags.Public | BindingFlags.Static);

    var binder =
        (from m in methods
            let parameters = m.GetParameters()
            where m.Name.Equals("BindService")
                && parameters.Length == 1
                && parameters.First()
                    .ParameterType.Equals(baseServiceType)
                && m.ReturnType.Equals(serviceDefinitionType)
            select m)
    .SingleOrDefault();

    if (binder == null)
    {
        throw new InvalidOperationException(
            $"Could not find service binder for provided service {serviceType.Name}"
        );
    }

    var serviceParameter = Expression.Parameter(serviceType);

    var invocation = Expression.Call(
        null, 
        binder, 
        new[] { serviceParameter }
    );

    var func = Expression
        .Lambda<Func<TService, ServerServiceDefinition>>(
            invocation, 
            false, 
            new[] { serviceParameter }
        ).Compile();

    return func;
}
```

This uses reflection to go through the class hierarchy of the service class to find the binder method. It’s probably a bit risky to count that this won’t change in future releases of the gRPC libraries, but for now it works well enough and simplifies the server configuration.

Getting back to the `GrpcServerBuilder`, when all the services are registered the `Server` is built and added to DI service collection with a call to `AddServerToServiceCollection`.

Wrapping up the `AddGrpcServer` method, the `GrpcBackgroundService` is added to the service collection (only once) so it can get the registered `Server`s and host them.

Everything that’s being added to DI is using the singleton scope, as we’ll only have one background service running with one (or more) `Server` listening for requests at all the time. As the `Server` requires the services instances to be registered at startup time, the services are also registered as singleton. As I mentioned earlier regarding implementing the service, it is the service that’s responsible for taking care of the lifecycle of its dependencies, hence the recommendations I made.

## Configuring the application

Now to configure the application, it’s just a matter of creating an `HostBuilder` and using the previously described extensions to configure the services.

```csharp
class Program
{
    static async Task Main(string[] args)
    {
        var serverHostBuilder = new HostBuilder()
        .ConfigureAppConfiguration((hostingContext, config) =>
        {
            //...
        })
        .ConfigureLogging((context, logging) =>
        {
            //...
        })
        .ConfigureServices((hostContext, services) =>
        {
            services
            .AddScoped<
                ISampleServiceLogic, 
                RandomSampleServiceLogic
            >()
            .AddScopedExecutor()
            //the most "magic" solution
            .AddGrpcServer<SampleServiceImplementation>(
                new[] { 
                    new ServerPort(
                        "127.0.0.1", 
                        5050, 
                        ServerCredentials.Insecur
                    e) 
                }
            )
            //a more manual solution if the flexibility is required
            //also not using the IScopedExecutor (although it could) 
            //for a more traditional example
            .AddGrpcServer(appServices =>
            {
                var scopeFactory 
                    = appServices.GetRequiredService<
                        IServiceScopeFactory
                    >();
                var server = new Grpc.Core.Server
                {
                    Services = { 
                        SampleService.BindService(
                            new AnotherSampleServiceImplementation(
                                scopeFactory
                            )
                        ) 
                    },
                    Ports = { 
                        new ServerPort(
                            "127.0.0.1", 
                            5051, 
                            ServerCredentials.Insecure
                        ) 
                    }
                };
                return server;
            });
        });

        await serverHostBuilder.RunConsoleAsync();
    }
}
```

To begin registering the services, the `ISampleServiceLogic` is added as scoped, followed by the registration of the `IScopedExecutor` I mentioned earlier. 

Then, just for the sample, I’m using 2 different extensions to register the gRPC service implementations, so we can see the available possibilities: 
In the first case, the easiest to use solution, providing the class implementing the service and its configurations, allowing for the extension method to handle the rest of the work.  
In the second case, I’m passing in a factory that returns a `Server`. In this case, the extension method only responsibility is passing the factory in to DI and registering the `GrpcBackgroundService`.

With everything configured we’re left with starting hosting the services by calling `RunConsoleAsync` on the `HostBuilder`.

## Wrapping up

Like I mentioned in the beginning, the helper library I created isn’t really very complex and some stuff is maybe unnecessary, so the main take away out of this are the things to be aware when we want to host a gRPC service in as similar fashion as possible to a ASP.NET Core application:

- Implementing IHostedService to host the `Server`
- Beware that dependency injection is not baked into gRPC core libraries, so some extra hoops are in our way to use it properly

Regarding this library, it’s on GitHub and I’ll probably create a NuGet package out of it one of these days, even if only for my own usage.

There have been hints in the past to support gRPC out of the box in ASP.NET Core (like this one [here](https://twitter.com/davidfowl/status/892553297550163968), or [this](https://github.com/grpc/grpc/issues/15139)), which would simplify a lot, but we're not there yet.

Any suggestions on improving the code (or the article) please do share.

Thanks for reading through, cyaz!
