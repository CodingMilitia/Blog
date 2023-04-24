---
author: João Antunes
date: 2023-04-24 08:45:00+01:00
layout: post
title: 'A reminder to consider that your context, isn’t the only context (.NET 8 TimeProvider edition)'
summary: "This will be a very brief post, just as a reminder that we need to give things a good thought, trying to limit our biases (I’ll focus on software development, but it’s applicable to everything really)."
images:
- '/images/2023/04/24/a-reminder-that-your-context-isnt-the-only-context-dotnet8-timeprovider-edition.png'
categories:
- csharp
- dotnet
tags:
- smalltalk
- testing
---

## Intro

This will be a very brief post, just as a reminder that we need to give things a good thought, trying to limit our biases (I’ll focus on software development, but it’s applicable to everything really).

The idea for this post came from a discussion going on in the .NET Runtime repository, about the introduction of a new `TimeProvider` type ([discussion here](https://github.com/dotnet/runtime/issues/36617)). It was a good reminder about how our takes on various subjects heavily depend on our context, and that we need to think things through and have an open mind, so we can try as best as possible to consider the overall picture.

## What’s the discussion around TimeProvider

So, what’s this `TimeProvider` I’m talking about? It’s a new time abstraction that’s very likely coming with .NET 8.

This API is exposed primarily through an abstract `TimeProvider` class, which, as expected given its name, provides a few features related to date and time, like getting the current local or UTC date and time, creating timers or calculating elapsed time. You can see it in [detail on GitHub](https://github.com/dotnet/runtime/blob/main/src/libraries/Common/src/System/TimeProvider.cs).

API design is always bound to generate discussions, as it’s rare everyone agrees, but besides the regular discussion, the `TimeProvider` generated a bit more, as at first glance, it seemed more complex than it should be. If the main goal is testability of time reliant code, why isn’t it just an `IClock` with a `Now` method? Is all of the other stuff really needed? If it is needed, why can’t we just segregate it, instead of putting everything in the same type?

## My initial take and how it changed

My initial take was exactly along the lines “why isn’t this just an `IClock` with a `Now` method?”. There were some comments in the issue that mirrored my concerns (although some go overboard with SOLID mumbo jumbo).

However, at some point, Stephen Toub [commented with an explanation](https://github.com/dotnet/runtime/issues/36617#issuecomment-1488795826) that lit the light bulb for me, and I finally understood the point of this API from a more general perspective. In particular, when he mentioned the use of this `TimeProvider` along with a `Task.Delay`, in order to control how the time goes by during a test. This was the real eye opening moment, because until this point, I was stuck in my little world, thinking of implementing domain logic in the typical line of business application many of us work on, and thinking I only need an `IClock` with a single method.

Getting back to the title of this post, I was so focused on my context, on the kind of code I do most often, that I didn’t consider there are other contexts, other kinds of code, where we need more than just something that gives us the current date and time. The `TimeProvider` goal is really to provide a core abstraction that allows us to control the flow of time in our applications.

So, how did my initial take change? Definitely! I still think that in the core of any business logic code, we should go with a simplified abstraction, be it an `IClock` with a single method, or even just a delegate (probably even better), so I’m probably not going to use the new `TimeProvider` in this context. However, when working in other layers, mainly infrastructure, I certainly see myself using the `TimeProvider`. 

## Outro

Wrapping up this super quick post: do remember that your context, isn’t the only context. Particularly when looking at these kinds of framework features, always keep in mind that some features might be targeted at your use cases, others not so much, and others, even if usable in your context, might not look exactly as you expect, because they need to cater to a variety of different scenarios.

Try to keep your biases in check (not only applicable on software 🙂).

Relevant links:

- [API to provide the current system time](https://github.com/dotnet/runtime/issues/36617)
- [TimeProvider class on GitHub](https://github.com/dotnet/runtime/blob/main/src/libraries/Common/src/System/TimeProvider.cs)

Thanks for stopping by, cyaz! 👋