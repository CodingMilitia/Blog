---
author: João Antunes
date: 2021-04-07 18:45:00+01:00
layout: post
title: "Virtual as a code smell*"
summary: "Lately I noticed I'm essentially treating virtual methods as a code smell, so thought it could result in an interesting blog post."
images:
- '/images/2021/04/07/virtual-as-a-code-smell.png'
categories:
- csharp
- dotnet
tags:
- oop
- patterns
slug: virtual-as-a-code-smell
---

## Intro

Right off the bat, I know the title is a bit clickbaity, but treating a virtual method as a code smell is actually something I have been doing for a while without even noticing, until just recently when I had a discussion and a light bulb lit over my head, so I thought it could result in an interesting blog post.

Note that the asterisk in the title is not a typo, but rather an acknowledgment that the train of thought I'll share in this post is heavily influenced by the types of projects I've been working on, so it doesn't apply in all contexts.

For my context, which is more often line of business applications, not as much library or framework code, I've been noticing this trend.

And just before getting into it, a friendly reminder that a code smell isn't necessarily something that's wrong, it's just something that should be given some extra attention to understand if it makes sense or not.

## Where is this coming from

So, where does this "virtual as a code smell" come from?

Recently I noticed after reviewing a couple of pull requests, as soon as I notice the `virtual` keyword being used, I give it a second look (and maybe also a third or fourth), eventually resulting in asking, "is this really needed?".

For starters, using inheritance is already something that should be well thought out (you know, [composition over inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance) and that sort of stuff), so that by itself makes using virtual a very localized occurrence. On this note, I appreciate that [Kotlin's classes are final by default](https://kotlinlang.org/docs/inheritance.html), cause I always forget to seal them in C# ([some related thoughts by Mark Seemann](https://blog.ploeh.dk/2021/03/08/pendulum-swing-sealed-by-default/)).

In addition to not inheriting that often, when I do, it's rare that I make a method virtual. From my recent memory, when implementing class hierarchies, it mostly fell on something like:

- There is an evident ["is-a" relationship](https://en.wikipedia.org/wiki/Is-a) that warrants the hierarchy (e.g. the error type I [discussed in a previous post](https://blog.codingmilitia.com/2020/03/02/aspnet-038-from-zero-to-overkill-more-explicit-error-handling-with-either/))
- Implementing the [template method pattern](https://en.wikipedia.org/wiki/Template_method_pattern), resorting to abstract methods

Off the top of my head, I can't really remember the last time I marked a method as virtual. I remember overriding virtual methods, all of them from libraries or frameworks (e.g. EF Core `SaveChanges` methods), but marking them as virtual? Nope, can't remember.

And to better grasp why I don't think it's needed most of the time, introducing a common counterargument might be of assistance.

## Extensibility and the open/closed principle

The first reaction to my question "why is this virtual needed?" was "in case the inheriting classes want to override it". That "in case" triggered me (probably more than it should), not only because in that context it didn't really make sense (which was probably the biggest cause for my overreaction), but also because we're getting into the futurology realm, where we don't have an actual use case, but doing things "just in case".

When I pushed on the subject, the follow-up reasoning was mentioning the open/closed principle. This is sadly a recurrent misconception about the principle in particular (it doesn't help that this principle, like a couple of others in SOLID, is a bit too vague) and object oriented design in general (side note, [Dan North has some interesting thoughts on SOLID in general](https://dannorth.net/2021/03/16/cupid-the-back-story/)).

This sort of thinking, is assuming that extensibility points should be provided by way of inheritance, which I'd argue isn't the way to go. There are multiple ways to provide extensibility points, it can be something as simple as passing a function as an argument (ASP.NET Core's request pipeline is a good example of this).

Getting back to the futurology subject and the open/closed principle, my take on it, particularly in modern times, where the pace of change is so fast, is that we should consider it in terms of the current requirements, not trying to guess the future.

As an example, which I used in my "[OOPs, I did it again](https://youtu.be/IM5sTt39A8g?t=1345)" presentation, if there's a requirement to conditionally apply multiple rules to a given use case, it might make sense to create a way to represent a rule and make it easy to implement multiple times (e.g. create an interface and implement it for each specific rule). Again, in this case we're not too heavy on the guess work, we have an actual present requirement and maybe are aware of further future requirements in terms of implementing more rules, we're not overcomplicating code "just in case".

## The asterisk

Before wrapping up, I think it's worth adding another note about the asterisk in the title.

As I mentioned, these thoughts are coming from the context of developing line of business applications. Framework and library code has different characteristics from application code, so your mileage may vary, but even then, it doesn't jump off to me as wise to just make everything unsealed and virtual as a strategy for extensibility.

Extensibility should be thought out, not just opening up everything, as that might result in unforeseen consequences.

As [Mark Seemann mentions in this post on sealed by default,](https://blog.ploeh.dk/2021/03/08/pendulum-swing-sealed-by-default/) sealing an unsealed class is a breaking change, unsealing a sealed class isn't. Same applies to virtual methods. Start out with the thing as constrained as possible and open up extensibility points as needed, explicitly and with concrete use cases in mind, not "just in case". If the extensibility point presents itself as making a method virtual, then, by all means.

## Outro

That's it for this one. I'm fully aware this isn't a topic where we're all going to agree, particularly considering the discussion on [Mark Seemann's tweet](https://twitter.com/ploeh/status/586560002313302016), and even more on [Kotlin's final by default decision](https://discuss.kotlinlang.org/t/classes-final-by-default/166) (now that's a loooong discussion 😛), but I felt like sharing by two cents, particularly after the recent aha moment 🙂.

Links in the post:

- [Composition over inheritance - Wikipedia article](https://en.wikipedia.org/wiki/Composition_over_inheritance)
- [Is-a - Wikipedia article](https://en.wikipedia.org/wiki/Is-a)
- [Template method pattern - Wikipedia article](https://en.wikipedia.org/wiki/Template_method_pattern)
- [CUPID – the back story](https://dannorth.net/2021/03/16/cupid-the-back-story/)
- [Inheritance - Kotlin docs](https://kotlinlang.org/docs/inheritance.html)
- [Pendulum swing: sealed by default - Mark Seemann](https://blog.ploeh.dk/2021/03/08/pendulum-swing-sealed-by-default/)
- [Mark Seemann's tweet on sealed by default](https://twitter.com/ploeh/status/586560002313302016)
- [Kotlin classes final by default forum discussion](https://discuss.kotlinlang.org/t/classes-final-by-default/166)
- [OOPs, I did it again](https://youtu.be/IM5sTt39A8g)
- [More explicit domain error handling and fewer exceptions with Either and Error types [ASPF02O|E038]](https://blog.codingmilitia.com/2020/03/02/aspnet-038-from-zero-to-overkill-more-explicit-error-handling-with-either/)

Thanks for stopping by, cyaz!