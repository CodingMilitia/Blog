---
author: João Antunes
date: 2022-01-03 18:00:00+00:00
layout: post
title: "Getting a complex type as a simple type from the query string in a ASP.NET Core API controller"
summary: "This is a tale of a good amount of hours of wasted time, so I’m going to document it so I remember it in the future. The idea is simple: when building an API, how can we treat a complex type as a simple type, to avoid things like primitive obsession, implement strongly typed ids and other related patterns? Let's find out! 🙂"
images:
- '/images/2022/01/03/complex-type-as-simple-type-aspnetcore.png'
categories:
- csharp
- dotnet
tags:
- dotnet
- aspnetcore
slug: getting-complex-type-as-simple-type-query-string-aspnet-core-api-controller
---

## Intro

This is a tale of a good amount of hours of wasted time, so I’m going to document it so I remember it in the future 🙂.

One of these days, while implementing a web API and respective OpenAPI/Swagger documentation, I wanted to treat a complex type as a simple one, passing it in the query string. I had done this in the past, but getting things from the route and with no Swagger involved, so I assumed it would work the same. Apparently not, and hence the story begins.

Side note: I’m using ASP.NET Core controllers in this specific case, but I’ll leave some notes regarding doing the same with minimal APIs towards the end.

## Prologue: why treat a complex type as a simple type

Before getting into how I got things working, it’s probably worth to mention why is it even relevant to go through these troubles. I won’t take too long explaining the reasons though, as you probably already heard this a bunch of times: **avoiding primitive obsession**.

[Andrew Lock wrote a series](https://andrewlock.net/using-strongly-typed-entity-ids-to-avoid-primitive-obsession-part-1/) on avoiding primitive obsession, with a focus on ids, but others, like [Jimmy Bogard](https://lostechies.com/jimmybogard/2007/12/03/dealing-with-primitive-obsession/) and [Vladimir Khorikov](https://enterprisecraftsmanship.com/posts/functional-c-primitive-obsession/) also wrote about it.

Very briefly, the idea is to avoid using primitive types for everything, so we have types representing specific concepts, centralize logic specific to them, as well make things more type safe in general.

With this in mind, I have a type like the following:

```csharp
public record struct SomeWrapperType(int Value)
{
    public int Value { get; } = Value;

    public static bool TryParse(string? value, out SomeWrapperType result)
    {
        if (int.TryParse(value, out var parsed))
        {
            result = new SomeWrapperType(parsed);
            return true;
        }

        result = default;
        return false;
    }
}
```

As you can see, the type itself is just a wrapper around an `int` (hence the name 🙂). In this case, I don’t have specific logic, just wanted to encapsulate the `int` value as a detail, so I can pass `SomeWrapperType` around the application.

We also have a `TryParse` method to help with creating an instance of the type from a `string` (similar idea to `int.TryParse` or `Guid.TryParse`).

## First try: create and configure model binder

As I briefly mentioned, in the past I did something similar, but getting the parameter from the route, instead of the query string, plus, and more relevant, no Swagger was involved. I assumed it would work the same, so tried the exact same strategy: created and configured a model binder.

As we’ll soon find out, not only didn’t it work as well as I hoped, but it’s even overkill, but we’ll get there. Before that, let’s quickly see the model binding bits.

Starting with the model binder, we have:

```csharp
public class SomeWrapperTypeModelBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        if (bindingContext == null)
        {
            throw new ArgumentNullException(nameof(bindingContext));
        }

        var modelName = bindingContext.ModelName;

        var valueProviderResult = bindingContext.ValueProvider.GetValue(modelName);

        if (valueProviderResult == ValueProviderResult.None)
        {
            return Task.CompletedTask;
        }

        var value = valueProviderResult.FirstValue;
        
        if (bindingContext.ModelType == typeof(SomeWrapperType)
            && SomeWrapperType.TryParse(value, out var result))
        {
            bindingContext.Result = ModelBindingResult.Success(result);
        }
        else
        {
            bindingContext.Result = ModelBindingResult.Failed();
        }

        return Task.CompletedTask;
    }
}
```

A bunch of boiler plate, but you can find the relevant part towards the end, in the last `if`, where it’s using the `SomeWrapperType.TryParse` to try to get an instance of the type from the provided `string`.

Additionally we need a model binder provider, to hook things up with ASP.NET Core:

```csharp
public class SomeWrapperTypeModelBinderProvider : IModelBinderProvider
{
    public IModelBinder? GetBinder(ModelBinderProviderContext context)
    {
        if (context == null)
        {
            throw new ArgumentNullException(nameof(context));
        }
            
        if (typeof(SomeWrapperType).IsAssignableFrom(context.Metadata.ModelType))
        {
            return new BinderTypeModelBinder(typeof(SomeWrapperTypeModelBinder));
        }

        return null;
    }
}
```

And finally, configure it:

```csharp
builder.Services.AddControllers(options =>
{
    options.ModelBinderProviders.Add(new SomeWrapperTypeModelBinderProvider());
});
```

The controller action, looks like this:

```csharp
[HttpGet("basic")]
public IActionResult GetWithNoCustomization(SomeWrapperType wrapper)
    => Ok(wrapper.Value);
```

If we run this now, what do we get?

{{< embedded-image "/images/2022/01/03/01-only-mode-binder-configured.png" "Swagger view with only model binder configured" >}}

So... something’s not great... not only is it not being treated as a simple type, but it’s showing up as part of the body, which in a GET request, is unexpected at best.

Part of it makes sense, as I added a model binder, but as we’re finding out, that doesn’t really contribute to the metadata required to generate the OpenAPI/Swagger bits, so it still shows up like that.

If we look at the available schemas, we see `SomeWrapperType` in there, further showing that it’s being treated as a complex type.

{{< embedded-image "/images/2022/01/03/02-type-shown-as-complex-in-swagger-schema.png" "Type shown as complex in Swagger schema" >}}

## Add type specific Swagger configuration

So, if the problem seems to be lack of metadata for docs generation, let’s try to add some.

When configuring Swashbuckle (which is the library I’m using here for Swagger stuff), we can add info about specific types. In this case, we can do the following:

```csharp
builder.Services.AddSwaggerGen(options =>
{
	// ...
    options.MapType<SomeWrapperType>(() => new OpenApiSchema { Type = "string" });
});
```

With this in place, `SomeWrapperType` should be treated as a simple type, in this case, like a regular `string`. Let’s see what happened in the Swagger UI:

{{< embedded-image "/images/2022/01/03/03-type-shown-as-simple-in-swagger.png" "Type shown as simple in Swagger" >}}

So, we have improvements, as now the type is being shown as `string` (and no longer shows up in the schemas section), but it’s still showing up in the request body.

## Add FromQuery attribute

Things showing up in the request body that shouldn’t, should be as simple as adding the `[FromQuery]` to the parameter, right? Let’s try it!

```csharp
[HttpGet("from-query")]
public IActionResult GetWithFromQuery([FromQuery]SomeWrapperType wrapper)
    => Ok(wrapper.Value);
```

{{< embedded-image "/images/2022/01/03/04-type-show-as-in-swagger.png" "Type shown as simple in Swagger" >}}

Almost, but we’re still not there!

As we can see, we now don’t have a body, the parameter moved to the query string, but as we further inspect, the parameter should be called `wrapper` and be a `string` (as we configured with Swashbuckle). Unfortunately, we see the name `Value` and type `int`, both referring to `SomeWrapperType`'s internal property.

## Enter type converters

At this point, I was getting annoyed and going further down rabbit holes and model binding shenanigans, but remembered to check Andrew Lock’s series of posts on strongly typed ids, to check if he touched on this, which fortunately for my sanity, he did (minus Swagger specifics).

And the answer is: [type converters](https://docs.microsoft.com/en-us/dotnet/api/system.componentmodel.typeconverter?view=net-6.0), which “Provides a unified way of converting types of values to other types, as well as for accessing standard values and subproperties.".

Going back to `SomeWrapperType`, did the following:

```csharp
[TypeConverter(typeof(SomeWrapperTypeTypeConverter))]
public record struct SomeWrapperType(int Value)
{
    public int Value { get; } = Value;

    public static bool TryParse(string? value, out SomeWrapperType result)
    {
        if (int.TryParse(value, out var parsed))
        {
            result = new SomeWrapperType(parsed);
            return true;
        }

        result = default;
        return false;
    }

    private class SomeWrapperTypeTypeConverter : TypeConverter
    {
        public override bool CanConvertFrom(ITypeDescriptorContext? context, Type sourceType)
            => sourceType == typeof(string) || base.CanConvertFrom(context, sourceType);

        public override object? ConvertFrom(ITypeDescriptorContext? context, CultureInfo? culture, object value)
        {
            var stringValue = value as string;
            if (TryParse(stringValue, out var result))
            {
                return result;
            }

            return base.ConvertFrom(context, culture, value);
        }
    }
}
```

As we can see above, created a class inheriting from `TypeConverter`, implemented the conversion from `string` to `SomeWrapperType`, and finally used it with the `TypeConverterAttribute`.

We can delete the model binder we created earlier, as it won’t be relevant when we’re using the type converter (and by now you see why I said the model binder was overkill, this is simpler).

Going back to Swagger UI...

{{< embedded-image "/images/2022/01/03/05-type-shown-as-string-in-query-string.png" "Type shown as simple string in query string in Swagger" >}}

... and it finally works as desired (and the `FromQuery` attribute isn’t needed).

A quick note that, even if we could get rid of the model binding bits (other than the type converter), the Swashbuckle configuration is still required, otherwise it’ll still show up as a complex type in the docs, even though it works as expected on the implementation side.

## Quick aside: it “just works” with minimal APIs

As a final piece of info, as I’ve also played with the new ASP.NET Core minimal APIs (which seem to be controversial for reasons that I don’t understand), everything would “just work”, without implementing most of the things described here.

Using this new approach, the existence of a `TryParse` method with a signature like I showed earlier, as well as a `BindAsync` method, are detected and used for parameter binding ([more information on the official docs](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis?view=aspnetcore-6.0#custom-binding)).

```csharp
app.MapGet(
    "/sample/minimal",
    (SomeWrapperType wrapper) => Results.Ok(wrapper.Value));
```

{{< embedded-image "/images/2022/01/03/06-all-good-with-minimal-apis.png" "All good with minimal APIs" >}}

## Outro

That does it for this mythical journey, full of twists and turns, where at the end we find out, a straight path was available right in our grasp 🤣.

Hope this can help someone avoid my mistakes.

In summary, for a complex type to be treat as a simple one when getting its value from the query string:

- For controllers, derive from `TypeConverter`, implement conversion logic and decorate the type with the `TypeConverterAttribute`.
- For minimal APIs, `TryParse` and `BindAsync` will do the trick.
- To make things look nice in the Swagger UI, use `MapType` to configure the schema when using Swashbuckle.

 Links in the post:

- [Sample code repository](https://github.com/joaofbantunes/GetComplexTypeFromQueryAsSimpleType)
- [An introduction to strongly-typed entity IDs](https://andrewlock.net/using-strongly-typed-entity-ids-to-avoid-primitive-obsession-part-1/)
- [Adding JSON converters to strongly typed IDs](https://andrewlock.net/using-strongly-typed-entity-ids-to-avoid-primitive-obsession-part-2/)
- [Dealing with primitive obsession](https://lostechies.com/jimmybogard/2007/12/03/dealing-with-primitive-obsession/)
- [Functional C#: Primitive obsession](https://enterprisecraftsmanship.com/posts/functional-c-primitive-obsession/)
- [TypeConverter class](https://docs.microsoft.com/en-us/dotnet/api/system.componentmodel.typeconverter?view=net-6.0)
- [Minimal APIs overview](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis?view=aspnetcore-6.0)

Thanks for stopping by, cyaz!