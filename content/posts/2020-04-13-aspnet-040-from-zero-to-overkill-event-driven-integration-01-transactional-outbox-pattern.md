---
author: João Antunes
date: 2020-04-13 17:30:00+01:00
layout: post
title: "Event-driven integration #1 - Intro to the transactional outbox pattern [ASPF02O|E040]"
summary: "As we start implementing event-driven integration, the first thing we need to do is publish the events. Although it might seem straightforward, there are some important things to consider in order to make it work reliably. In this episode, we discuss the challenges and introduce the transactional outbox pattern to help us facing these challenges."
images:
- '/images/2020/04/13/aspnet-core-from-zero-to-overkill-e040.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-040-from-zero-to-overkill-event-driven-integration-transactional-outbox-pattern
---

As we start implementing event-driven integration, the first thing we need to do is publish the events. Although it might seem straightforward, there are some important things to consider in order to make it work reliably. In this episode, we discuss the challenges and introduce the transactional outbox pattern to help us facing these challenges.

**Note:** depending on your preference, you can check out the following video, otherwise, skip to the written version below.

{{< youtube suKSJ5DvynA >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

In this first episode discussing the implementation of event-driven integration in our system, we won't see much code, but rather look at the challenges that the first step we need to take, publishing the events, presents to us.

After understanding these challenges, we'll introduce the transactional outbox pattern to help us build reliable event publishing.

## The most obvious event publishing approach

Let's begin with something that, I guess, seems super obvious at first glance, but really isn't: how and when to publish the events.

Let's think about the user registered event. Our first thought is to go to `Register.cshtml.cs`, and the line after we check if registration was successful, we invoke something like `eventBus.PublishAsync(new UserRegisteredEvent(/*...*/))`. Seems simple enough, right? But is it good enough? What if something happens in that time frame? Or if the event bus is down at that time? Let's see a simple diagram, to try to visualize this a bit more clearly.

{{< embedded-image "/images/2020/04/13/e040-fail-to-publish-event.png" "fail to publish event" >}}

As we were discussing, and now looking at the diagram, what happens if the server goes done in `(1)`, or maybe if there is a connectivity issue when publishing the event, in `(2)`? In such cases, the registration succeeds, as the information is persisted to the database, but the event is never sent, so the other services will never know a new user was registered.

What we need is for the database operation and event publishing to be done in a single transaction, but that's not really possible (unless we're using SQL Server and MSMQ, which supported distributed transactions), so we need to find an alternative approach to ensure at-least-once delivery.

## The transactional outbox pattern

To get the transactional guarantees we require, we can implement the [transactional outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html).

In a nutshell, this pattern consists in storing the events in the database, in the same transaction as the other changes we're doing, making sure that both things either get persisted or are rolled back.

If we're using a relational database (as is the case in the PlayBall project), this is usually implemented using an auxiliary table, the outbox, where we store the events we want to send.

In a NoSQL database, like a document database, where usually the transactional guarantees are limited to the document being changed, it's normally implemented by storing the events inside the document.

In either case (SQL or NoSQL) , we'll have another intervenient in the process, which either polls or listens to new events in the outbox, then publishes them to the message broker. After being published, the events can be deleted from the outbox.

An illustration of this whole process would be something like:

{{< embedded-image "/images/2020/04/13/e040-outbox-pattern.png" "transactional outbox pattern" >}}

Taking this approach, gives us the at-least-once delivery guarantee we require. It's not perfect though, not only because it's more work to do, but also because, as mentioned, it gives us at-least-once delivery guarantee, not exactly once. Not having exactly once delivery comes from the fact that the outbox publisher might fail to delete the events from the database after publishing to the bus, causing it to try again at a later time.

Due to this possibility of receiving the same event multiple times, the subscribers need to be prepared to avoid processing repeated events, which could cause data correctness issues. One technique that can be used is to deduplicate the events based on their id, but that's something we'll explore in an upcoming post 🙂.

## Outro

That does it for this really quick intro to the transactional outbox pattern, the problems we had in the first place and what it can do to help us with those problems.

The first steps in our event-driven integration journey will be focused on implementing this pattern, which is a subject that will likely take the next three episodes:

- How/when to create the events based on what goes on in the auth service
- Store the events in the outbox
- Publish the events stored in the outbox

I'm trying to split things up more in the series, as I tend to mix too many topics in each episode, making it more difficult to  properly highlight relevant bits. It also results in quicker posts to read/videos to watch. Hope this approach makes sense 🙂.

Links in the post:

- [Pattern: Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!