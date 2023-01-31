---
author: João Antunes
date: 2023-01-31 17:30:00+00:00
layout: post
title: 'Mapping ASP.NET Core minimal API endpoints with C# source generators'
summary: "I’m pretty late to the C# source generator party, but hey, better late than never 😅. In this post, let's take a look at how we can automagically map minimal API endpoints using C# source generators"
images:
- '/images/2023/01/31/mapping-aspnet-core-minimal-api-endpoints-with-csharp-source-generators.png'
categories:
- csharp
- dotnet
tags:
- aspnetcore
- 'minimal apis'
slug: mapping-aspnet-core-minimal-api-endpoints-with-csharp-source-generators
---

## Intro

This will be a very simple post about how we can use C# source generators to map minimal API endpoints automagically.

The obvious way to do this automatic endpoint mapping, would be to do reflection based assembly scanning, which is a very common approach to do these kinds of things, like registering services, which, for example, we can do pretty easily with libraries like [Scrutor](https://github.com/khellang/Scrutor).

Although reflection based assembly scanning works, there’s just a little something we can do better: performance. Reflection isn’t the fastest thing, so if we can avoid it and just have things put in place at compile time, there’s one less thing slowing down our application’s startup. Additionally, [.NET Native AOT](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/) and reflection might not go too well together (still early days though), so it’s good to already start looking at possibilities.

I’m pretty late to the C# source generator party, but hey, better late than never 😅. Not only am I pretty late, but I’m pretty sure what I’m talking about in this post, has already been talked loads of times, but I wanted to try things out for myself, so, like many other of my posts, if for no one else, it’ll be relevant for future me 🙃.

One final note on the post, is that the code you’ll was just to try things out, so things should be improved for actual production use (e.g. better type safety and avoiding finding things with hardcoded strings). But that’s hopefully something you already assume when looking at code in blogs 🙂.

## The API and hooking points

Let’s start with the API (which is as basic as it can be), as well as, more importantly, the integration points put in place for the source generator to hook into.

For starters, we have an `IEndpoint` interface, exposing a single `static abstract` method to implement ([new feature in C# 11](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11#generic-math-support)). This will allow the source generator to look for all implementations of the interface.

```csharp
namespace Api;

public interface IEndpoint
{
    static abstract IEndpointRouteBuilder Map(IEndpointRouteBuilder endpoints);
}
```

Then, we have the most basic hello world implementation:

```csharp
namespace Api;

public class HelloEndpoints : IEndpoint
{
    public static IEndpointRouteBuilder Map(IEndpointRouteBuilder endpoints)
    {
        endpoints.MapGet("/", () => "Hello World!");
        return endpoints;
    }
}
```

The whole point of this post, is that we don’t want to call `HelloEndpoints.Map` manually, so instead, in the `Program.cs`, we call a method `RegisterEndpoints`, which will be implemented by the source generator, to map all endpoints:

```csharp
using Api;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.RegisterEndpoints();

app.Run();
```

This `RegisterEndpoints` method is defined as a partial extension method in a partial `EndpointRegistrationExtensions` class.

```csharp
namespace Api;

public static partial class EndpointRegistrationExtensions
{
    public static partial IEndpointRouteBuilder RegisterEndpoints(this IEndpointRouteBuilder endpoints);
}
```

The importance of this class and method being `partial`, is that then the source generator can generate code for the same class, actually implementing the method. Source generators cannot modify existing code, only add more code, so this is a way to leave our type open for extensibility, allowing our source generator to contribute to it.

## Bootstrap the source generator

I’m just going to quickly skim past this one, as this is one of the first things in the [docs](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview#get-started-with-source-generators): we need to create a .NET Standard 2.0 class library, with references to the `Microsoft.CodeAnalysis.CSharp` and `Microsoft.CodeAnalysis.Analyzers` packages.

My `Generators.csproj` looks like the following:

```xml
<Project Sdk="Microsoft.NET.Sdk">

    <PropertyGroup>
        <TargetFramework>netstandard2.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <LangVersion>preview</LangVersion>
        <Nullable>enable</Nullable>
        <WarningsAsErrors>nullable</WarningsAsErrors>
    </PropertyGroup>

    <ItemGroup>
        <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.3.1" PrivateAssets="all" />
        <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.3">
            <PrivateAssets>all</PrivateAssets>
            <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
        </PackageReference>
    </ItemGroup>

</Project>
```

There’s some extra stuff I didn’t mention (like nullable and language version bits), but that’s just about some of my preferences, not really source generator related.

With the project in place, we can reference it from the API project, like so:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

    <PropertyGroup>
        <TargetFramework>net7.0</TargetFramework>
        <Nullable>enable</Nullable>
        <WarningsAsErrors>nullable</WarningsAsErrors>
        <ImplicitUsings>enable</ImplicitUsings>
    </PropertyGroup>

    <ItemGroup>
        <ProjectReference Include="..\Generators\Generators.csproj" OutputItemType="Analyzer" ReferenceOutputAssembly="false"/>
    </ItemGroup>

</Project>
```

Then we can create our source generator class. This class should be decorated with the `Generator` attribute, and inherit from `ISourceGenerator`, which will provide us a couple of methods to implement.

```csharp
using Microsoft.CodeAnalysis;

namespace Generators;

[Generator]
public class EndpointRegisterExtensionsGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
        throw new NotImplementedException();
    }

    public void Execute(GeneratorExecutionContext context)
    {
        throw new NotImplementedException();
    }
}
```

## Collecting required information

Before generating the endpoint mapping code, the source generator needs to do two things: find the `EndpointRegistrationExtensions` class we want to extend, as well as find all implementations of `IEndpoint`, so we can map them.

There are a couple of ways to do this: in the execute method, using the `context` parameter go through the all nodes and find what we want; implement an `ISyntaxReceiver` that we configure to be invoked by the runtime for each node it finds. I’m not sure I have a preference between them at this point, so for no particular reason, I went with the latter. If you want to learn more, [Khalid wrote a post on the subject](https://khalidabuhakmeh.com/dotnet-source-generators-finding-class-declarations).

The `ISyntaxReceiver` implementation, which I named `Collector` and is an internal class of the source generator, looks like this:

```csharp
private class Collector : ISyntaxContextReceiver
{
    private readonly List<ClassDeclarationSyntax> _endpoints = new();
    private ClassDeclarationSyntax? _partial;

    public IReadOnlyCollection<ClassDeclarationSyntax> Endpoints => _endpoints;

    public ClassDeclarationSyntax Partial
        => _partial ?? throw new InvalidOperationException("Could not collect partial class to implement");

    public void OnVisitSyntaxNode(GeneratorSyntaxContext context)
    {
        if (context.Node is not ClassDeclarationSyntax @class) return;

        if (@class.Identifier.ValueText == "EndpointRegistrationExtensions")
        {
            _partial = @class;
        }
        else
        {
            var classSymbol = context.SemanticModel.GetDeclaredSymbol(@class);
            if (classSymbol?.AllInterfaces.Any(i => i.ToDisplayString().EndsWith("IEndpoint")) ?? false)
            {
                _endpoints.Add(@class);
            }
        }
    }
}
```

As you can see, it’s not something particularly esoteric. We check if the node in the context is a class, then check if it’s one of the two things we’re looking for: the partial class we’ll extend, or an implementation of `IEndpoint` (yes, doing hardcoded string comparison isn’t a good idea, I warned you at the beginning of the post 😜). We then expose this information in properties for the source generator to use.

To configure the `Collector` to be used, we implement the source generator `Initialize` method like so:

```csharp
[Generator]
public class EndpointRegisterExtensionsGenerator : ISourceGenerator
{
    private readonly Collector _collector = new();

    public void Initialize(GeneratorInitializationContext context)
        => context.RegisterForSyntaxNotifications(() => _collector);

		// ...
}
```

## Generating the code

With the bulk of the work done, all that’s left is to generate the code. We do this in the source generator `Execute` method, which looks like the following:

```csharp
[Generator]
public class EndpointRegisterExtensionsGenerator : ISourceGenerator
{
    // ...

    public void Execute(GeneratorExecutionContext context)
    {
        // Retrieve the populated receiver
        var collector = (Collector)context.SyntaxContextReceiver!;

        var @namespace = GetNamespace(collector.Partial);

        var endpointRegistrations = new StringBuilder();

        foreach (var endpointClass in collector.Endpoints)
        {
            endpointRegistrations.AppendLine($"{endpointClass.Identifier.ValueText}.Map(endpoints);");
        }

        var source = // lang=C#
            $$""""
// <auto-generated/>

namespace {{@namespace}}; 

public static partial class EndpointRegistrationExtensions
{
    public static partial IEndpointRouteBuilder RegisterEndpoints(this IEndpointRouteBuilder endpoints)
    { 
				{{endpointRegistrations}}
        return endpoints;
    }
}
"""";

				context.AddSource(
            $"{nameof(EndpointRegisterExtensionsGenerator)}.generated.cs", 
            SourceText.From(source, Encoding.UTF8));
    }

    private static string GetNamespace(SyntaxNode? node)
        => node switch
        {
            NamespaceDeclarationSyntax namespaceNode => namespaceNode.Name.ToString(),
            FileScopedNamespaceDeclarationSyntax fileScopedNamespaceNode => fileScopedNamespaceNode.Name.ToString(),
            { } => GetNamespace(node.Parent),
            _ => throw new InvalidOperationException("Could not find namespace")
        };

    // ...
}
```

Let’s go step by step, they should be mostly self explanatory from the code.

We start by casting the context’s `SyntaxtContextReceiver` property to our `Collector`, so we can access our collected data.

Then, we grab the namespace in which the `EndpointRegistrationExtensions` lives, so we add the partial to the same one.

We then compose the lines of code invoking the `Map` method on all `IEndpoint` implementations we found.

With these things ready, we compose the final code, which is just a plain string. The code is again using some recent C# features, namely [raw string literals](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11#raw-string-literals), to make it much more readable (plus that `// lang=C#` comment, is a JetBrains Rider feature that enables syntax highlighting within the string 🤯). This syntax highlighting is a bit messed up in the blog, as my blog engine doesn’t understand C# 11 🙃.

Finally, we invoke `context.AddSource` to add our code to the solution.

With all of this in place (assuming I didn't forget any step while writing this post), we can now run our API and everything will work as we hoped 🙂.

## Outro

That’s it for this quick look at how we can implement automatic discovery and mapping of minimal API endpoints with C# source generators.

As I mentioned, this was a quick and dirty test of how things could be done, so the code would need some extra effort to be a bit more production ready.

In any case, it was a good exercise to have a better understanding of how source generators work and how we can use them to solve some common problems.

Relevant links:

- [Source code repository](https://github.com/joaofbantunes/RegisteringMinimalApisWithSourceGeneratorsSample)
- [Source Generators - Microsoft Docs](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview)
- [.NET Source Generators: Finding Class Declarations](https://khalidabuhakmeh.com/dotnet-source-generators-finding-class-declarations)
- [What's new in C# 11](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11)

Thanks for stopping by, cyaz! 👋
