---
author: johnny
date: 2018-04-15 20:40:00+01:00
layout: post
title: 'Dotnetifying gRPC'
summary: 'Very likely over-engineered PoC on making development of gRPC service development more dotnetish.'
image: '/assets/2018/04/15/csharp-grpc.jpg'
categories:
- dotnet
tags:
- dotnet
- grpc
- services
---

[![C# + gRPC](/assets/2018/04/15/csharp-grpc.jpg)](/assets/2018/04/15/csharp-grpc.jpg)

I started this little proof of concept with the simple goal of creating some helper functions to aid in hosting gRPC services in .NET in a way similar to ASP.NET applications. However, while scouring the web, reading some articles to try and find the best way to do that, I came across this [article](http://bartoszsypytkowski.com/c-using-grpc-with-custom-serializers/) and realized I could get a little more creative - maybe I ended up being too much, but at least I had fun along the way :).

I’ll try to cover the main bits of what I’ve done here, but if something you find important to get the whole picture that isn't here, please tell me and I’ll try to improve the explanation.

You can follow along the code that’s on [GitHub](https://github.com/CodingMilitia/Grpc/tree/april-blog-post) (I created a tag to make it easy to relate this article to the code).

Grab some popcorn, this is a looooong post!

## What’s this “dotnetifying”?
So, like I said the initial goal was just to make the service hosting somewhat similar to an ASP.NET application. Besides hosting, if the service client registration could also be streamlined, that would also be great (and probably the easiest part before I started to get too creative).

These were the initial goals. What I started to wonder after reading the aforementioned article, was that I could probably make the service implementation experience different as well, taking inspiration from Entity Framework Code First, that is, instead of the usual code generation that’s required when using gRPC, one could just create the service interface, the messages to use as requests and responses, create the service implementation and using it all without even touching a .proto file and the other usual tools - as long as you don’t need to interop with other languages, in that case you better write that .proto file so the others can generate their stuff.

## What I’m not trying to do
With this PoC I’m not trying to abstract away the core gRPC libraries, I’m using them and exposing them as needed. So even if using this code to simplify the development of these services, knowledge of the underlying core libraries is needed.

## The usual way
Before talking about what I wanted to achieve, it might be better to start with a quick intro on creating a very simple service with gRPC the normal way.

We start by creating a .proto file containing the service description.

{% highlight protobuf linenos %}
syntax = "proto3";

option csharp_namespace = "CodingMilitia.Grpc.GeneratedServerInterop.Generated";

service SampleService {
    rpc Send(SampleRequest) returns (SampleResponse) {}
}

message SampleRequest {
    int32 value = 1;
}

message SampleResponse {
    int32 value = 1;
}
{% endhighlight %}

Then we use the tools that are installed with the Grpc.Tools NuGet package to generate the C# code.

{% highlight bash linenos %}
./protoc service.proto --csharp_out ./Generated/. --grpc_out ./Generated/. --plugin=protoc-gen-grpc=grpc_csharp_plugin
{% endhighlight %}

With the generated code we can use the generated client (class `SampleServiceClient`) to invoke an already running service or inherit from the generated service base (class `SampleServiceBase`) to implement the server side.

{% highlight csharp linenos %}
//server
 var server = new Server
{
    Services = { Generated.SampleService.BindService(new SampleServiceImplementation()) },
    Ports = { new ServerPort("127.0.0.1", 5000, ServerCredentials.Insecure) }
};
server.Start();
{% endhighlight %}

{% highlight csharp linenos %}
//client
var channel = new Channel("127.0.0.1:5000", ChannelCredentials.Insecure);
var client = new Generated.SampleService.SampleServiceClient(channel);
var request = new Generated.SampleRequest { Value = 1 };
var response = await client.SendAsync(request);
{% endhighlight %}

Like I said at the beginning, my initial goal was just to wrap this with some helper methods for DI and simplify hosting, but then, I went rogue...

## Desired outcome
Just to provide a glimpse of the desired outcome in terms of hosting and consuming the services, here’s a bit of wishful thinking code (that actually works).

### Shared between client and server

{% highlight csharp linenos %}
[GrpcService("SampleService")]
public interface ISampleService : IGrpcService
{
    [GrpcMethod("Send")]
    Task<SampleResponse> SendAsync(SampleRequest request, CancellationToken ct);
}

[ProtoBuf.ProtoContract]
public class SampleRequest
{
    [ProtoBuf.ProtoMember(1)]
    public int Value { get; set; }
}

[ProtoBuf.ProtoContract]
public class SampleResponse
{
    [ProtoBuf.ProtoMember(1)]
    public int Value { get; set; }
}
{% endhighlight %}

The attributes above the messages are dependant on the serializer used. I’m using [protobuf-net](https://github.com/mgravell/protobuf-net) to implement the serializer, hence the attributes used.

### Server

{% highlight csharp linenos %}
var serverHostBuilder = new HostBuilder()
    .ConfigureServices((hostContext, services) =>
    {
        services.AddGrpcServer<ISampleService, SampleService>(new GrpcServerOptions { Url = "127.0.0.1", Port = 5000 });
    });

await serverHostBuilder.RunConsoleAsync(cts.Token);
{% endhighlight %}

### Client

{% highlight csharp linenos %}
//in a normal scenario we wouldn't instantiate the ServiceCollection, it's just for demo purposes
var clientServices = new ServiceCollection()
    .AddGrpcClient<ISampleService>(new GrpcClientOptions { Url = "127.0.0.1", Port = 5000 })
    .BuildServiceProvider();
var client = clientServices.GetRequiredService<ISampleService>();
var request = new SampleRequest { Value = 1 };
var response = await client.SendAsync(request, CancellationToken.None);
{% endhighlight %}

## Implementing it

So, like I said previously, the inspiration to what I ended up with came from the [article by Horusiath](http://bartoszsypytkowski.com/c-using-grpc-with-custom-serializers/). In a nutshell, he takes advantage of the core gRPC libraries classes that are used by the generated code to create the service and client classes without any code generation - plus, he uses a custom Protocol Buffers serializer, as opposed to the also usually generated ones.

### The base for it all
So, to start with the things that are common to both client and server.

A gRPC service is composed of methods. There are 4 types of methods: unary, server streaming, client streaming and duplex streaming - for this PoC I implemented just the most basic type, unary, a simple request response. 

To create the method definitions, I built an helper class `MethodDefinitionGenerator` (my naming skills are a bit weak, sorry about that) that exposes a method `CreateMethodDefinition`, that returns a `Method<TRequest, TResponse>` - this class is part of the core gRPC library. For each service method it’s required a method type (that right now will always be `Unary`), the service and method names, as well as a serializer for the messages. The serializer is configurable because I’m using a third party library to implement it, so I thought it was best to keep it easy to change.

{% highlight csharp linenos %}
using System;
using CodingMilitia.Grpc.Serializers;
using Grpc.Core;

namespace CodingMilitia.Grpc.Shared.Internal
{
    //TODO: review visibility
    public static class MethodDefinitionGenerator
    {
        public static Method<TRequest, TResponse> CreateMethodDefinition<TRequest, TResponse>(
            MethodType methodType,
            string serviceName,
            string methodName,
            ISerializer serializer
        )
            where TRequest : class
            where TResponse : class
        {
            return new Method<TRequest, TResponse>(
                type: methodType,
                serviceName: serviceName,
                name: methodName,
                requestMarshaller: Marshallers.Create(
                    serializer: serializer.ToBytes<TRequest>,
                    deserializer: serializer.FromBytes<TRequest>
                ),
                responseMarshaller: Marshallers.Create(
                    serializer: serializer.ToBytes<TResponse>,
                    deserializer: serializer.FromBytes<TResponse>
                )
            );
        }
    }
}
{% endhighlight %}

The `ISerializer` interface is the simplest thing ever (but after reading [this article](https://dev.to/scotthannen/depending-on-functions-instead-of-interfaces---why-and-how-50o6) I might change the approach). It just exposes a couple of methods to serialize and deserialize a generic type `T` to and from a `byte[]`.

{% highlight csharp linenos %}
namespace CodingMilitia.Grpc.Serializers
{
    public interface ISerializer
    {
         byte[] ToBytes<T>(T input);
         T FromBytes<T>(byte[] input);
    }
}
{% endhighlight %}

Besides these, there’s also a couple of attributes to apply to the service interface and methods, right now only to configure the names that’ll be used in the method definitions.

{% highlight csharp linenos %}
using System;

namespace CodingMilitia.Grpc.Shared.Attributes
{
    [AttributeUsage(AttributeTargets.Interface)] 
    public class GrpcServiceAttribute : Attribute
    {
        public string Name { get; set; }

        public GrpcServiceAttribute(string name)
        {
            Name = name;
        }
    }
}
{% endhighlight %}

{% highlight csharp linenos %}
using System;

namespace CodingMilitia.Grpc.Shared.Attributes
{
    [AttributeUsage(AttributeTargets.Method)] 
    public class GrpcMethodAttribute : Attribute
    {
        public string Name { get; set; }

        public GrpcMethodAttribute(string name)
        {
            Name = name;
        }
    }
}
{% endhighlight %}

### Server
For the server side, I initially thought I had to implement some hosting helpers to have the service hosted in a console application in a similar way to ASP.NET Core. Then I realised this was already done in .NET Core 2.1 (that is in preview 1 at this point), which would avoid a good amount of work. So, leveraging the new hosting options, the very simple class `GrpcBackgroundService` implements `IHostedService`, and simply starts and stops a `GrpcHost` instance. Right now `GrpcHost`, is a thin abstraction on the core gRPC libraries’ `Server` class that handles the hosting of the services.

{% highlight csharp linenos %}
using CodingMilitia.Grpc.Shared;
using Microsoft.Extensions.Hosting;
using System.Threading;
using System.Threading.Tasks;

namespace CodingMilitia.Grpc.Server.Internal
{
    internal class GrpcBackgroundService<TService> : IHostedService where TService : class, IGrpcService
    {
        private readonly GrpcHost<TService> _host;
        
        public GrpcBackgroundService(GrpcHost<TService> host)
        {
            _host = host;
        }

        public async Task StartAsync(CancellationToken cancellationToken)
        {
            await _host.StartAsync().ConfigureAwait(false);
        }

        public async Task StopAsync(CancellationToken cancellationToken)
        {
            await _host.StopAsync().ConfigureAwait(false);
        }
    }
}
{% endhighlight %}

{% highlight csharp linenos %}
using CodingMilitia.Grpc.Shared;
using System.Threading.Tasks;
using GrpcCore = Grpc.Core;

namespace CodingMilitia.Grpc.Server.Internal
{
    internal class GrpcHost<TService> where TService : class, IGrpcService
    {
        private readonly GrpcCore.Server _server;

        public GrpcHost(GrpcCore.Server server)
        {
            _server = server;
        }

        public Task StartAsync()
        {
            _server.Start();
            return Task.CompletedTask;
        }

        public async Task StopAsync()
        {
            await _server.ShutdownAsync().ConfigureAwait(false);
        }
    }
}
{% endhighlight %}

To get the simple service implementation registration with DI we saw before on the “desired outcome” part, there’s an extension method on `IServiceCollection` that registers `GrpcBackgroundService` and the `GrpcHost` as singletons, as well as the provided service implementation as scoped, to achieve an ASP.NET MVC controller like behavior - one instance per request.

{% highlight csharp linenos %}
using CodingMilitia.Grpc.Serializers;
using CodingMilitia.Grpc.Server.Internal;
using CodingMilitia.Grpc.Shared;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace CodingMilitia.Grpc.Server
{
    public static class ServiceCollectionExtensions
    {
        public static IServiceCollection AddGrpcServer<TServiceInterface, TServiceImplementation>(
            this IServiceCollection serviceCollection,
            GrpcServerOptions options,
            ISerializer serializer
        )
            where TServiceInterface : class, IGrpcService
            where TServiceImplementation : class, IGrpcService, TServiceInterface
        {
            serviceCollection.AddScoped<TServiceInterface, TServiceImplementation>();
            serviceCollection.AddSingleton<GrpcHost<TServiceInterface>>(
                appServices => GrpcHostFactory.Create<TServiceInterface>(appServices, options, serializer)
            );

            serviceCollection.AddSingleton<IHostedService, GrpcBackgroundService<TServiceInterface>>();
            return serviceCollection;
        }
    }
}
{% endhighlight %}

As can be seen above, besides what was already mentioned, there is also a call to a `GrpcHostFactory.Create` that’s responsible for creating the required `GrpcHost`, and here is where most of the code first magic for the server lives.

{% highlight csharp linenos %}
using CodingMilitia.Grpc.Serializers;
using CodingMilitia.Grpc.Shared;
using CodingMilitia.Grpc.Shared.Attributes;
using System;
using System.Reflection;

namespace CodingMilitia.Grpc.Server.Internal
{
    internal static class GrpcHostFactory
    {
        public static GrpcHost<TService> Create<TService>(
            IServiceProvider appServices,
            GrpcServerOptions options,
            ISerializer serializer
        )
            where TService : class, IGrpcService
        {
            var builder = new GrpcHostBuilder<TService>(appServices);
            builder.SetOptions(options);
            builder.SetSerializer(serializer);
            builder.AddUnaryMethods();
            return builder.Build();
        }

        private static GrpcHostBuilder<TService> AddUnaryMethods<TService>(
            this GrpcHostBuilder<TService> builder
        )
            where TService : class, IGrpcService
        {
            //TODO: right now it goes through every method, these must be validated and filtered
            var serviceType = typeof(TService);
            var serviceName = ((GrpcServiceAttribute)serviceType.GetCustomAttribute(typeof(GrpcServiceAttribute))).Name ?? serviceType.Name;

            foreach (var method in serviceType.GetMethods())
            {
                var requestType = method.GetParameters()[0].ParameterType;
                var responseType = method.ReturnType.GenericTypeArguments[0];

                var handlerGenerator = typeof(MethodHandlerGenerator).GetMethod(nameof(MethodHandlerGenerator.GenerateUnaryMethodHandler));
                handlerGenerator = handlerGenerator.MakeGenericMethod(serviceType, requestType, responseType);
                var handler = handlerGenerator.Invoke(null, new[] { method });

                var addUnaryMethod = typeof(GrpcHostBuilder<TService>).GetMethod(nameof(GrpcHostBuilder<TService>.AddUnaryMethod), BindingFlags.Public | BindingFlags.Instance);
                addUnaryMethod = addUnaryMethod.MakeGenericMethod(requestType, responseType);

                var methodName = ((GrpcMethodAttribute)method.GetCustomAttribute(typeof(GrpcMethodAttribute))).Name ?? method.Name;

                addUnaryMethod.Invoke(builder, new[] { handler, serviceName, methodName });
            }

            return builder;
        }
    }
}
{% endhighlight %}

As you can see, this is where the shenanigans start - not that it’s super hard, but a bit trickier than usual, mainly because we get into reflection land. The beginning is straightforward, setting the options and the serializer on a `GrpcHostBuilder` instance. After that it starts iterating on all of the service interface methods, assuming they’re all unary methods and performing no validation whatsoever (yes, this really needs to be improved).

For each method, it creates an handler using the `MethodHandlerGenerator` class. The handler is basically a `Func` that wraps a call to the service implementation method, using an instance provided to it. The obtained handler is then passed along as an argument to the `AddUnaryMethod` method of the builder.

The `GrpcHostBuilder` works with the core gRPC libraries’ `ServerServiceDefinition.Builder` to create the `Server`.

{% highlight csharp linenos %}
using CodingMilitia.Grpc.Serializers;
using CodingMilitia.Grpc.Shared;
using CodingMilitia.Grpc.Shared.Internal;
using Grpc.Core;
using Microsoft.Extensions.DependencyInjection;
using System;
using System.Threading;
using System.Threading.Tasks;

namespace CodingMilitia.Grpc.Server.Internal
{
    internal class GrpcHostBuilder<TService> where TService : class, IGrpcService
    {
        private readonly IServiceProvider _appServices;
        private readonly ServerServiceDefinition.Builder _builder;
        private GrpcServerOptions _options;
        private ISerializer _serializer;

        public GrpcHostBuilder(IServiceProvider appServices)
        {
            _appServices = appServices;
            _builder = ServerServiceDefinition.CreateBuilder();
        }

        public GrpcHostBuilder<TService> SetOptions(GrpcServerOptions options)
        {
            _options = options;
            return this;
        }

        public GrpcHostBuilder<TService> SetSerializer(ISerializer serializer)
        {
            _serializer = serializer;
            return this;
        }

        public GrpcHost<TService> Build()
        {
            var server = new global::Grpc.Core.Server
            {
                Ports = { { _options.Url, _options.Port, ServerCredentials.Insecure } },
                Services =
                {
                    _builder.Build()
                }
            };
            return new GrpcHost<TService>(server);
        }

        public GrpcHostBuilder<TService> AddUnaryMethod<TRequest, TResponse>(
            Func<TService, TRequest, CancellationToken, Task<TResponse>> handler,
            string serviceName,
            string methodName
        )
            where TRequest : class
            where TResponse : class
        {
            _builder.AddMethod(
                MethodDefinitionGenerator.CreateMethodDefinition<TRequest, TResponse>(MethodType.Unary, serviceName, methodName, _serializer),
                async (request, context) =>
                {
                    using (var scope = _appServices.CreateScope())
                    {
                        var service = scope.ServiceProvider.GetRequiredService<TService>();
                        var baseService = service as GrpcServiceBase;
                        if (baseService != null)
                        {
                            baseService.Context = context;
                        }
                        return await handler(service, request, context.CancellationToken).ConfigureAwait(false);
                    }
                }
            );
            return this;
        }
    }
}
{% endhighlight %}

In the builder constructor we get an `IServiceProvider` so we can fetch a service implementation per request. The SetOptions and SetSerializer methods are pretty self-explanatory, as is the Build method, that basically just creates the final `Server` instance using the provided arguments and the `ServerServiceDefinition.Builder` end product.

The `AddUnaryMethod` method calls the inner builder AddMethod, providing the method definition (created with the previously discussed `MethodDefinitionGenerator`) and a lambda to handle the requests. The lambda creates a new scope with the `IServiceProvider`, fetches a new service instance and uses the handler to invoke the desired method. If the service extends `GrpcServiceBase`, the `ServiceCallContext` is set, much like the `HttpContext` on an MVC `Controller` class.

Still with me this far? Nice! Onwards to the client side of things and some added craziness.

### Client

On the client side of things, we basically need to generate a proxy class, somewhat like the generated SOAP service we have in WCF. The development time generation is basically what gRPC’s tooling does, but as I’m being a smartass, I’m doing it at runtime (because, you know, reasons).

This is definitely the most overengineered part of this PoC, but at least I got to play around with something I hadn’t before - IL emitting!

Like for the server stuff, let me start the story from the outside and gradually dig into the more obscure stuff.

Just like the server side of things, I created an extension method over `IServiceCollection` to register the client as a singleton. The client is obtained with a call to `GrpcClientFactory.Create`.

{% highlight csharp linenos %}
using CodingMilitia.Grpc.Serializers;
using CodingMilitia.Grpc.Shared;
using Microsoft.Extensions.DependencyInjection;

namespace CodingMilitia.Grpc.Client
{
    public static class ServiceCollectionExtensions
    {
        public static IServiceCollection AddGrpcClient<TServiceInterface>(
            this IServiceCollection serviceCollection,
            GrpcClientOptions options,
            ISerializer serializer
        )
            where TServiceInterface : class, IGrpcService
        {
            serviceCollection.AddSingleton<TServiceInterface>(_ => GrpcClientFactory.Create<TServiceInterface>(options, serializer));
            return serviceCollection;
        }
    }
}
{% endhighlight %}

The factory just uses the `GrpcClientTypeBuilder` class to create the client proxy type and then returns a new instance of it.

{% highlight csharp linenos %}
using System;
using System.Linq;
using System.Reflection;
using System.Reflection.Emit;
using CodingMilitia.Grpc.Serializers;
using CodingMilitia.Grpc.Shared;
using CodingMilitia.Grpc.Shared.Attributes;

namespace CodingMilitia.Grpc.Client.Internal
{
    internal class GrpcClientTypeBuilder
    {
        public TypeInfo Create<TService>() where TService : class, IGrpcService
        {

            var assemblyName = Guid.NewGuid().ToString();
            var assemblyBuilder = AssemblyBuilder.DefineDynamicAssembly(new AssemblyName(assemblyName), AssemblyBuilderAccess.Run);
            var moduleBuilder = assemblyBuilder.DefineDynamicModule(assemblyName);

            var serviceType = typeof(TService);
            var typeBuilder = moduleBuilder.DefineType(serviceType.Name + "Client", TypeAttributes.Public, typeof(GrpcClientBase));

            typeBuilder.AddInterfaceImplementation(serviceType);
            AddConstructor(typeBuilder, serviceType);
            AddMethods(typeBuilder, serviceType);

            return typeBuilder.CreateTypeInfo();
        }

        //...
    }
}
{% endhighlight %}

The `Create` method starts by creating an assembly builder and a module builder, with a GUID as the name right now, maybe in the future I come up with some useful name, but right now it isn’t really relevant. Then a `TypeBuilder` is created, and the service client type starts to be defined. The client type implements the provided service interface and inherits from `GrpcClientBase`, an abstract class with some auxiliary methods to simplify the rest of the type definition.
After defining the base class and interface implementation, the `AddConstructor` and `AddMethods` are called, wrapping up with returning a completely runtime generated type that represents the service client. 

Before getting into `AddConstructor` and `AddMethod`, let’s take a quick look into `GrpcClientBase`.

{% highlight csharp linenos %}
using CodingMilitia.Grpc.Serializers;
using CodingMilitia.Grpc.Shared.Internal;
using System.Threading;
using System.Threading.Tasks;
using G = Grpc.Core;

namespace CodingMilitia.Grpc.Client.Internal
{
    public abstract class GrpcClientBase
    {
        private readonly G.Channel _channel;
        private readonly G.DefaultCallInvoker _invoker;
        private readonly ISerializer _serializer;

        protected GrpcClientBase(GrpcClientOptions options, ISerializer serializer)
        {
            _channel = new G.Channel(options.Url, options.Port, G.ChannelCredentials.Insecure);
            _invoker = new G.DefaultCallInvoker(_channel);
            _serializer = serializer;
        }

        protected async Task<TResponse> CallUnaryMethodAsync<TRequest, TResponse>(TRequest request, string serviceName, string methodName, CancellationToken ct)
            where TRequest : class
            where TResponse : class
        {
            var callOptions = new G.CallOptions(cancellationToken: ct);
            using (var call = _invoker.AsyncUnaryCall(GetMethodDefinition<TRequest, TResponse>(G.MethodType.Unary, serviceName, methodName), null, callOptions, request))
            {
                return await call.ResponseAsync.ConfigureAwait(false);
            }
        }

        private G.Method<TRequest, TResponse> GetMethodDefinition<TRequest, TResponse>(G.MethodType methodType, string serviceName, string methodName)
            where TRequest : class
            where TResponse : class
        {
            return MethodDefinitionGenerator.CreateMethodDefinition<TRequest, TResponse>(methodType, serviceName, methodName, _serializer);
        }
    }
}
{% endhighlight %}

This class is rather simple (at least compared to what’s coming next) and simply makes use of gRPC’s core libraries to make service calls.
The constructor receives some arguments to create the communication channel and a serializer like we saw earlier. The `CallUnaryMethodAsync` does exactly what the name says, invokes an unary method exposed by the service, making use of `MethodDefinitionGenerator.CreateMethodDefinition` to get the method definition like we also saw earlier.

Now for the first IL emitting part, invoking the base class constructor.

{% highlight csharp linenos %}
private void AddConstructor(TypeBuilder typeBuilder, Type serviceType)
{
    var ctorBuilder = typeBuilder.DefineConstructor(
        MethodAttributes.Public,
        CallingConventions.Standard,
        new[] { typeof(GrpcClientOptions), typeof(ISerializer) }
    );

    var il = ctorBuilder.GetILGenerator();
    il.Emit(OpCodes.Ldarg_0); //load this
    il.Emit(OpCodes.Ldarg_1); //load options
    il.Emit(OpCodes.Ldarg_2); //load serializer
    var clientBaseType = typeof(GrpcClientBase);
    var ctorToCall = clientBaseType.GetConstructor(BindingFlags.NonPublic | BindingFlags.Instance, null, new[] { typeof(GrpcClientOptions), typeof(ISerializer) }, null);
    il.Emit(OpCodes.Call, ctorToCall);//call base class constructor
    il.Emit(OpCodes.Ret);
}
{% endhighlight %}

So, just to get this out of the way, I can’t write IL by heart, so I wrote the code I expected to obtain, went for the resulting IL and used it here (used [.NET Fiddle](https://dotnetfiddle.net/) to get the IL).

In summary, it’s defining a public constructor that receives `GrpcClientOptions` and `ISerializer` instances, then loading the arguments onto the stack to then call the base class constructor. Now you see why the base class? So I didn’t need to generate the whole IL, just the minimum necessary the call the base class and code the rest like usual.

Now for the `AddMethods` + `AddMethod`

{% highlight csharp linenos %}
private void AddMethods(TypeBuilder typeBuilder, Type serviceType)
{
    foreach (var method in serviceType.GetMethods())
    {
        AddMethod(typeBuilder, method);
    }
}

private void AddMethod(TypeBuilder typeBuilder, MethodInfo method)
{
    var serviceName = ((GrpcServiceAttribute)method.DeclaringType.GetCustomAttribute(typeof(GrpcServiceAttribute))).Name ?? method.DeclaringType.Name;
    var methodName = ((GrpcMethodAttribute)method.GetCustomAttribute(typeof(GrpcMethodAttribute))).Name ?? method.Name;

    var args = method.GetParameters();
    var methodBuilder = typeBuilder.DefineMethod(
        method.Name,
        MethodAttributes.Public | MethodAttributes.Virtual,
        method.ReturnType,
        (from arg in args select arg.ParameterType).ToArray()
    );
    var il = methodBuilder.GetILGenerator();
    il.Emit(OpCodes.Ldarg_0); //load this
    il.Emit(OpCodes.Ldarg_1); //load request
    il.Emit(OpCodes.Ldstr, serviceName); //load constant service name as argument
    il.Emit(OpCodes.Ldstr, methodName); //load constant method name as argument
    il.Emit(OpCodes.Ldarg_2); //load cancellation token
    var clientBaseType = typeof(GrpcClientBase);
    var methodToCall = clientBaseType.GetMethod("CallUnaryMethodAsync", BindingFlags.Instance | BindingFlags.NonPublic);



    il.Emit(
        OpCodes.Call,
        methodToCall.MakeGenericMethod(new[]{ //TODO: must check arguments and stuff
            method.GetParameters()[0].ParameterType,
            method.ReturnType.GetGenericArguments()[0]
        })
    ); //call base class method

    il.Emit(OpCodes.Ret); //return (the return value is already on the stack )

    typeBuilder.DefineMethodOverride(methodBuilder, method);
}
{% endhighlight %}

The `AddMethods` just loops through the service interface’s methods, but the `AddMethod` is where things go crazy again.

It starts with a simple fetching of the service and method names from the custom attributes discussed earlier. Then the code is similar to the discussed regarding the constructor - define the method, getting its argument types from the `MethodInfo` we’re using at the moment, then start emitting IL to load the arguments onto the stack to then call the base class `CallUnaryMethodAsync` method. Because `CallUnaryMethodAsync` is a generic method, there are some extra reflection hoops to jump through, but the method invocation is done like in the constructor code. To wrap up the return statement is emitted and the new method is defined as an override in the type builder.

## Summary
Now this has gotten to be a longer post than what I expected, but I hope it’s useful for something. 

Now to wrap it up, what does this do and what it doesn’t? Like I said along the way, I tried to cover only the most basic of scenarios to get a feel of what could be done, and by looking at the outcome, I think that with a little more work it is possible to support all the usual options that the core gRPC libraries provide, while making it more similar to other .NET technologies to start with. 

If this is used in a purely .NET environment, we don't even have to worry about proto files and stuff like that, but if we want interoperability with other techs, it's better to create the service definition files, so the others can work as usual, although in this case this whole thing gets less useful.

So, just to make it clear, is it ready to production?... NO! (as if it wasn’t obvious by this time) But if it’s useful, I think it can get there, adding the missing parts, improving the API, making sure performance is on par with what we would get if using the core libraries as usual.

Even if this PoC doesn’t go any further, at least I got a excuse to play with some different stuff, including IL emitting :)

If you read this whole thing, thanks a lot for that! I hope it was useful. Cyaz!
