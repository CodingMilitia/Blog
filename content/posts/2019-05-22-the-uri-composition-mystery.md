---
author: João Antunes
date: 2019-05-22 21:30:00+01:00
layout: post
title: "The Uri composition mystery"
summary: "A quick tale of an idiot wasting hours by not checking the docs earlier."
images:
- '/images/2019/05/22/the-uri-composition-mystery.png'
categories:
- dotnet
tags:
- dotnet
- csharp
slug: the-uri-composition-mystery
---

## Intro
This will be one of those posts of shame, that I'll use to make sure the next time I get into the same problem, I don't waste hours trying to figure it out 😛.

## So, what's the problem?
So I was using an `HttpClient`, passing it an `Uri` for the request to make. To compose the `Uri`, the constructor that gets two `Uri`s was being used, the first `Uri` represents the base address, so an absolute `Uri`, while the second should be a relative address.

With this in mind, looking at the following code, what would you expect to be written to the console?

```csharp
var baseUri = new Uri("https://api.dev/v3");
var routeUri = new Uri("stuff", UriKind.Relative);
var fullUri = new Uri(baseUri, routeUri);

Console.WriteLine(fullUri);
```

My expectation would be `https://api.dev/v3/stuff`, but alas, that's not what we get! The output is `https://api.dev/stuff`, because I didn't add a `/` to the end of the base address. If the base address ends with `/`, then the composition would work as expected.

But wait! There's more...

Even with the trailing slash in the base address, if the relative address starts with a slash, it will replace the relative part of the base address as well.

So, the following code:

```csharp
var baseUri = new Uri("https://api.dev/v3/");
var routeUri = new Uri("/stuff", UriKind.Relative);
var fullUri = new Uri(baseUri, routeUri);

Console.WriteLine(fullUri);
```

Will also output `https://api.dev/stuff`.

## "Adding insult to injury"
What's worse than the time I wasted on this, is that this behavior is described in the [docs](https://docs.microsoft.com/en-us/dotnet/api/system.uri.-ctor?view=netcore-2.2#System_Uri__ctor_System_Uri_System_Uri_).

> Remarks
>
> This constructor creates a new Uri instance by combining an absolute Uri instance, baseUri, with a relative Uri instance, relativeUri. If relativeUri is an absolute Uri instance (containing a scheme, host name, and optionally a port number), the Uri instance is created using only relativeUri.
> 
> If the baseUri has relative parts (like /api), then the relative part must be terminated with a slash, (like /api/), if the relative part of baseUri is to be preserved in the constructed Uri.
> 
> Additionally, if the relativeUri begins with a slash, then it will replace any relative part of the baseUri
> 
> This constructor does not ensure that the Uri refers to an accessible resource.

The behavior just seemed so strange to me (although there's probably a good reason for it), I didn't think about looking at the docs, and kept scouring the code for some other reason to what was happening.

## Outro
Wrapping up, if we want to be sure the `Uri` composition works well, we should end the base address with a `/` and **not** start the relative part with one.

```csharp
var baseUri = new Uri("https://api.dev/v3/");
var routeUri = new Uri("stuff", UriKind.Relative);
var fullUri = new Uri(baseUri, routeUri);

Console.WriteLine(fullUri);
// outputs -> https://api.dev/v3/stuff
```

Hopefully I'll remember this the next time! 🙃