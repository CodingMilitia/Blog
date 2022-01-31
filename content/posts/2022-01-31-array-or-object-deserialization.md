---
author: João Antunes
date: 2022-01-31 17:25:00+00:00
layout: post
title: "Array or object JSON deserialization (feat. .NET & System.Text.Json)"
summary: "Ah, the joys of integrating with third-party APIs... We always end up having to hammer something to get things working 🤣. This is a post about one of such situations, resorting to some JSON deserialization trickery (via JsonConverter) to be able to get things working."
images:
- '/images/2022/01/31/array-or-object-deserialization.png'
categories:
- csharp
- dotnet
tags:
- json
slug: array-or-object-deserialization
---

## Intro

Ah, the joys of integrating with third-party APIs... We always end up having to hammer something to get things working 🤣.

This is a post about one of such situations, resorting to some JSON deserialization trickery (via `JsonConverter`) to be able to get things working.

## Problem statement

So, I’m integrating with this API, which in a specific endpoint, for some reason, returns an object in which one of its properties, when empty, is an empty array, but when there’s data, it’s an object with an items property which in turn is the array.

Some examples, for better understanding:

When collection is empty:

```json
{
    "someItems": []
}
```

When collection has entries:

```json
{
    "someItems": {
        "items": [
            {
                "id": 123,
                "text": "some text"
            },
            {
                "id": 456,
                "text": "some more text"
            }
        ]
    }
}
```

Let’s just say that the deserializer wasn’t happy when this happened 🙂.

So, how do we get out of this mess? Enter `JsonConverter`s.

Side note: I reported this behavior as a bug, which will be fixed at some point, but until then, I still need to get things working.

## Implementing a JsonConverter

Let’s jump right into the code!

```csharp
public class ArrayOrObjectJsonConverter<T> : JsonConverter<IReadOnlyCollection<T>>
{
    public override IReadOnlyCollection<T>? Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options)
        => reader.TokenType switch
        {
            JsonTokenType.StartArray => JsonSerializer.Deserialize<T[]>(ref reader, options),
            JsonTokenType.StartObject => JsonSerializer.Deserialize<Wrapper>(ref reader, options)?.Items,
            _ => throw new JsonException()
        };

    public override void Write(Utf8JsonWriter writer, IReadOnlyCollection<T> value, JsonSerializerOptions options)
        => JsonSerializer.Serialize(writer, (object?) value, options);

    private record Wrapper(T[] Items);
}
```

Fortunately, as we can see, it’s not that much code, so we’ll go through it quickly.

We’re inheriting from `JsonConverter<IReadOnlyCollection<T>>`, to implement the JSON converter for that specific type (I normally use `IReadOnlyCollection` when passing collections around instead of `IEnumerable`, to be sure the collection isn’t lazy unless I really want it to be).

As for the read implementation, we check what the first token of the object’s JSON representation is:

- Array start token (`[`) - we deserialize it as an array
- Object start token (`{`) - we deserialize it as the `Wrapper` type declared below, which fits the structure of the objects we’re receiving
- Another thing - that’s unexpected, so blowing things up 💥

Note that the usage of `T[]` is important. If we used `IReadOnlyCollection<T>`, we’d get into an infinite loop, with the converter basically calling itself again and again. I used `T[]` here, but it could also be another type of collection, just can’t be `IReadOnlyCollection`, to avoid the loop.

Regarding writing, it wasn’t important in my case, as at least for now I only need to deserialize things, but anyway, created a simple implementation where it’s always serialized as a JSON array. We could easily adjust it a bit if we wanted to serialize in different ways, depending on some logic.

Again, note that the cast to `object?` is important, for the same reason we used `T[]` earlier, we end up in a loop because the value is of type `IReadOnlyCollection<T>`. Casting it like this, the serializer will care for the underlying type, being oblivious to the original type of the object reference. We could achieve the same in some other ways, like casting to `IEnumerable<T>`, or removing the cast altogether and pass the generic parameter like `JsonSerializer.Serialize<object?>(writer, value, options)`.

## Outro

That’s it for this quick post.

We saw how to make use of `JsonConverter`s to customize (de)serialization of specific types, in this case to workaround a bug in an API, but it’s also useful for a lot of other situations.

I’ve been dabbling with `JsonConverter`s a bunch lately, so I’ll probably share one or two more of these posts, to document my shenanigans.

Fortunately, it ended up being less complicated than I expected, so I was happy not to waste too much time hammering things to hide bugs.

 Links in the post:

- [Sample implementation](https://gist.github.com/joaofbantunes/37ebec5576b2b5a8624aa8a664dba113)
- [How to write custom converters for JSON serialization (marshalling) in .NET](https://docs.microsoft.com/en-us/dotnet/standard/serialization/system-text-json-converters-how-to?pivots=dotnet-6-0)

Thanks for stopping by, cyaz! 👋