---
author: João Antunes
date: 2023-05-16 08:20:00+01:00
layout: post
title: 'From domain events to infrastructure - thinking out loud about possible approaches I don’t hate'
summary: "On the topics of domain-driven design and event-driven architecture, something that’s been on my mind for quite some time, is the process of handling domain events, as well as potentially translating them into integration events."
images:
- '/images/2023/05/16/from-domain-events-to-infrastructure.png'
categories:
- csharp
- dotnet
tags:
- 'event-driven architecture'
- ddd
---

## Intro

On the topics of domain-driven design and event-driven architecture, something that’s been on my mind for quite some time, is the process of handling domain events, as well as potentially translating them into integration events.

Most of the content I look at on this subject, in general falls on one of two sides:

- Completely ignores the associated challenges (I’ve done that myself)
- I just don’t agree with it 🙃

In this post, I’m going to try to distill the ideas I have on the subject, trying to come up with a possible approach that makes sense in general, for the kinds of projects I’ve been involved in. There’s no one size fits all, so it certainly won’t be a solution for every case, but I would really like to have, for myself, an approach that makes sense to me, and that at least I can use as a starting point, adapting as needed.

This post is assuming you are aware of common DDD and other usually related terms, like entities, aggregates, repositories, command and event handlers, and so on.

Before getting into the approaches I’ve been considering, let’s start by looking at some things I’ve commonly seen and am not a fan, to understand where I’m coming from with the approaches that make more sense (to me at least 😅).

## Dislike #1 - dispatching and handling domain events through an in-memory bus

I’ll start with one that is really common in C# code bases, in particular ones using some kind of Clean Architecture template (there’s like a gazillion of them 🤣): dispatching and then handling domain events through an in-memory bus. This is very commonly implemented by storing the domain events in a collection within an entity class, then in an overridden EF Core’s `SaveChanges` method (or some other abstraction), get those events and publish them into an in-memory bus, like MediatR, before or after persisting the changes (i.e. committing the transaction).

I’m not a fan of this approach for a number of reasons:

- Publishing and handling domain events after committing might result in inconsistent state
- Handling domain events within the triggering command’s transaction promotes a false sense of decoupling
- Handling domain events within the triggering command’s transaction assumes that’s always a possibility
- Handling domain events within the triggering command’s transaction ignores DDD guidance of one transaction per aggregate

Let me try to go into each one very briefly.

### Publishing and handling domain events after committing changes might result in inconsistent state

This one’s the quickest to explain: if the events are published and handled after the triggering command handler changes are persisted, if something goes wrong when handling the events, be it an unexpected exception or a service crash, the event is just gone, and we might get our data into an inconsistent state.

In earlier posts, I’ve talked about the [transactional outbox pattern]({{< relref path="2020-04-13-aspnet-040-from-zero-to-overkill-event-driven-integration-01-transactional-outbox-pattern.md" >}}), which can be used to address this issue, by persisting events in the same transaction as the aggregate changes are persisted. The outbox pattern seems to be typically associated with integration events, but I don’t feel like the type of event is relevant, as what matters is that if the event is to be executed in a different transaction, we need to ensure that it’ll be executed, regardless of potential application or infrastructure hiccups.

I’ve read articles stating that this is rarely a problem, but I’ve got two issues with this take. For starters, while it certainly can depend on the scenario, in general, “rarely” isn’t good enough. I won’t go to the business and say “you know, sometimes, the data just gets messed up”, and shrug my way out of there. Secondly, I’ve worked in projects where multiple non-transactional actions were being done in one go, and I can tell you that it wasn’t so “rare”, and we weren’t even in a cloud or microservices context. In this cloud context we work in these days, with serverless, Kubernetes, auto-scaling and all of those buzzwords, assuming these issues don’t happen or are rare, is just plain irresponsible.

### Handling domain events within the triggering command’s transaction promotes a false sense of decoupling

Now unto the next approach, publishing and handling domain events within the triggering command handler transaction. I’ve got a couple of issues with this, and the first one is that it promotes a false sense of decoupling.

The whole point (at least in my view) of splitting command and event handling, is to decouple things, so the command is processed just considering what it needs to do in the context of an aggregate, and an event is raised so that other aggregates can potentially react, also only considering what they need to consider in their context. The problem is, by doing things in the same transaction, there’s not really much decoupling.

If the actions triggered by a domain event are really simple, with a very low probability of failure, then this whole thing becomes less of a problem. But what if the triggered actions are more complex and can fail more often, be it due to domain logic or infrastructure issues? Then it starts to become a problem, and we need to be implementing the logic triggered by the event in a way that considers what came before it, which doesn’t really look that decoupled, does it?

If a failure of the actions that are done in response to the triggered event should indeed make the original command being handled fail, then I’d prefer to make it explicit and orchestrate things in the command handler, rather then raising events that need to be aware of what preceded them.

Another example of things becoming messy, is if the actions triggered by the domain event trigger further actions, for example: a domain event triggers a command on another aggregate, which publishes another domain event, which triggers another command on yet another aggregate, and so on. Now what? We handle a bunch of commands in the same transaction? Feels like something will go wrong, sooner, rather than later.

### Handling domain events within the triggering command’s transaction assumes that’s always a possibility

Another issue I have with doing the event handling within the original command handler’s transaction, though it annoys me less than the previous one, is the assumption that this is even possible. This assumption stems from the fact that relational databases are very often used, and in these cases, it’s highly likely that we can just wrap everything in a transaction.

My biggest issue isn’t assuming a type of database, but the fact that folks create a bunch of abstractions, convinced it will allow them to replace the database with very low effort, when in fact it won’t, as the abstractions are very leaky in that regard. Not only that, but most of the times, these abstractions aren’t only leaking the type of database, but also the libraries used to implement them. A couple of examples: exposing `IQueryable`?  the fact that there is an implementation of it just leaked; manipulating entities then calling some parameterless `SaveChanges`like method? the existence of a change tracker was just leaked. If anyone thinks they’re gonna move from EF Core to Dapper easily, if any of these assumptions were made, then good luck implementing an `IQueryable` provider and a change tracker 😅.

Now, while I’m not saying to implement things in a such a way that we can jump from a relational database to a document one without touching anything other than a couple of repositories (that would be nice, but not always possible, or, at least, worth the effort and compromises), I’d like to have an approach where the core behavior is similar enough to feel familiar when developing, as teams go from service to service.

### Handling domain events within the triggering command’s transaction ignores DDD guidance of one transaction per aggregate

This one should be sufficiently self-explanatory. In DDD literature, the recommendation is to make an aggregate not only a consistency boundary, but also the concurrency and transactional boundary. Anything that aggregates need to do in response to another’s events, should ideally be eventually consistent. By handling domain events in the same transaction, we’re immediately discarding this guidance.

Now, you know I’m not a purist (most of the time at least 😅), and that’s clear in the previous reasons, where I said I’d be open to touching multiple aggregates in one transaction, but preferred to do it explicitly. And that’s the thing, by doing it explicitly, we’re explicitly saying that in that particular case, we’re ignoring this guidance for a specific reason, instead of just ignoring the guidance implicitly.

## Dislike #2 - disconnected domain and integration events

Another one of my dislikes, but this one should be quicker to explain, is a common disconnect between domain and integration events.

If we’re following DDD ideas, I’d expect that the domain is the thing that makes everything tick. But what I see at times, is that domain and integration events are implemented in a disconnected way, where domain events are raised by the domain types (if we’re lucky and it isn’t an anemic domain model), while the integration events are raised by a command handler, on it’s own, without any consideration for the domain and its events.

Maybe I’m slipping a bit into purism, but what I would expect, is for a domain event to be raised, which is then transformed into an integration event (or multiple), that can later be dispatched. This way, the domain drives all the things, while the code around it instead of taking more responsibilities than it should, just creates a bridge between the bounded context and the rest of the world.

## Considering options that solve these dislikes

Ok, so after this not so brief look at some of my dislikes related to approaches I see in the wild (and I probably forgotten some), let me start thinking out loud about potential alternatives.

To summarize some of my requirements, considering my dislikes above:

- The transactional outbox pattern should be used, to ensure all events are published
- Avoid doing multiple disconnected actions in the same transaction (at least implicitly)
- Similar approach can be used with different common database types for these kinds of applications (e.g. relational or document databases)

### #1 Push all domain events into outbox, map then publish

I’ll start with one which I don’t actually think is a good approach, but given I said I was gonna think out loud, I’ll share this one, that also came to mind.

The simplest approach I thought of, from a domain and command handler point of view, was to simply push all domain events into the transactional outbox, then have the outbox publisher transform and publish them in whatever ways needed: publish the domain events themselves, for example to an internal topic*, so that other aggregates in the same bounded context could react to them, and/or transform the domain events into integration events and publish them to external topics. Besides its simplicity on the domain/command handler side, it also has low impact on the command handling processing time, as it just pushes things into the outbox, doesn’t do any additional processing.

The reason I don’t believe this would be a good approach, is that when publishing integration events, there’s some probability that we want to include more information in the event. This means that the outbox publisher, would be required to do more than a simple mapping and publishing, it would need to also do queries to include more detail. This could potentially cause a couple of problems:

- As the outbox publisher should normally sequentially publish messages, adding to it these added responsibilities, can have a severe impact on performance, causing the message publishing to lag, if the system is generating more events than the publisher is able to handle
    - We can probably do something to minimize this, but it’ll make things even more complex
- From the time the message was added to the outbox, to the time the publisher gets to work on it, the related aggregate might have already changed, which could cause issues when doing the transformation from domain to integration event

This should go without saying, but for clarities sake, for the same performance reasons as just described, the domain event handling shouldn’t be triggered directly by the outbox. Additionally, if we used the outbox as a sort of queue infrastructure, we’d need to manually implement many things existing messaging infrastructure already does for us.

\* quick note: when I say internal topic, it could be a topic using the same messaging infrastructure used for external events, just destined to be consumed by the same service, or it could be something else, for example Hangfire. I’d probably go with the same messaging infrastructure, since it’s already there and setup for our needs, but you might prefer something else.

### #2 Push all domain events into outbox, publish, consume and then map

This next potential approach, was one I was considering to be a good possibility, even if potentially not my preferred one, but found some common flaw when comparing with the previous one.

For this approach, the idea would be to avoid the potential performance issues with the outbox publisher described before, by making it always publish the domain events to the internal topic. Then, the service would subscribe to those events, not only to handle any needs from other aggregates in the bounded context, but also to transform from domain to integration event when needed.

The interesting thing about this approach, similar to the previous one, is that it’s simple and fast from a domain and command handler point of view, as it’s only needed to push the domain events into the outbox. It improves on the previous one, in terms of simplifying the outbox publisher as well, removing potential performance issues. Even with these improvements, the reason it wasn’t my preferred option, it that it seems a bit inefficient, as it always pushes the events into the messaging infrastructure, even if just to then map them to other events. As usual, trade-offs.

Now, as briefly mentioned, it shares a flaw with the previous approach, which you might have noticed as you read this section: if the aggregate that triggered the domain event has changed between then and when the transformation is performed, some issues might arise.

So, it’s still not an approach I believe is good enough, but it’s nice to consider these various scenarios 🙂.

### #3 Push already mapped events into outbox, then publish

We get to this last one, which I hope is a valid one (though maybe you’ll notice some issue I didn’t take into consideration? let me know if so).

Let’s try to address the main issues with the previous couple of ideas:

- We need to keep the outbox publisher simple and performant
- We need to avoid problems with changes to an aggregate between domain event publishing and integration event creation
- It’d be nice if we can avoid too much back and forth with infrastructure (e.g. publish event to then consume, transform and publish again)

To address these issues, the idea that comes to mind is: transform the raised domain events before pushing them into the outbox. Then, the outbox publisher can simply read the events and publish them accordingly, either to an internal topic if it is a domain event, or an external topic if it is an integration one.

{{< embedded-image "/images/2023/05/16/from-domain-events-to-infrastructure-diagram.png" "From domain events to infrastructure" >}}

> Side note: this is a high-level diagram, just to try and make things a bit clearer. There are more details involved, like how are the events mapped and persisted.

This does seem to solve all the issues presented above:

- Outbox just gets the events and publishes them to the right topic, no queries or other more time consuming tasks
- Domain and integration events are created in the same context and transaction, so they’re consistent with each other
- More efficient use of infrastructure

This approach is not without its trade-offs, of course. By having the transformation happen in the original command handler’s context, before pushing the events into the outbox, we’re adding more work to be done, which can make this handling slightly slower (e.g. if we need to query something to include in the integration event). This means that if the command was triggered by a user action, the user will need to wait a bit more. Realistically, this is probably a non-issue, as this event transformation will probably just add some milliseconds to the overall command handling time. With this, we spread the latency associated with transforming events, in a way that’s negligible, unlike if we did it in the outbox publisher, or even by putting more unnecessary strain in the infrastructure, as described in the previous potential approaches.

## Quick note about the need for the transactional outbox

From this whole post, it might seem that I’m saying that we can’t have an event-driven system without an outbox. That’s not the case, so I just wanted to add note about it as I’m wrapping things up.

One scenario in which an outbox isn’t needed, is if we’re using event sourcing. With event sourcing, by design, we store the events that cause the evolution of our aggregates, so we just need to subscribe to changes to the database, then map the events as needed before publishing them into messaging infrastructure.

Another scenario in which you might not need an outbox, is if you’re able to guarantee that the command will be retried until everything succeeds, including the publishing of the events. In such a situation, I’d probably use an outbox anyway, as it would be probably easier to implement, but it wouldn’t be strictly needed.

One final scenario that comes to mind, is if the only side-effect caused by handling the command, is to publish events (i.e. no changes to a database or calls to other APIs). This might happen if you’re using something like Kafka, and simply subscribing to a topic to transform events into other kinds of events (and maybe filtering, distributing to different topics, and so on). In such a case, one could even use something like Kafka transactions, so that an event is marked as consumed transactionally with the publishing of another, but even that might not be needed if the downstream services implement idempotency adequately.

I’m sure there are other cases where the transactional outbox pattern is not a requirement, but these are just some examples of why it’s not always needed.

## Outro

That does it for this kind of different post. Normally I get straight into coding, but this one was just about thinking 😅.

As mentioned throughout the post, this was mostly an exercise in finding an approach to go from domain events to infrastructure. Most of the approaches I’ve seen explained weren’t really doing it for me, so I thought about taking bits and pieces from several ideas, and trying to come up something I like.

The usual disclaimer, that the suggested approaches likely aren’t ideal for every scenario. For the projects I’ve been involved in, I feel like I found an interesting starting point, that can been adjusted on a case by case basis.

Anyways, hope this little exercise was an interesting read, and don’t forget to ping me if you have further ideas on the subject 🙂.

Relevant links:

- [Intro to the transactional outbox pattern]({{< relref path="2020-04-13-aspnet-040-from-zero-to-overkill-event-driven-integration-01-transactional-outbox-pattern.md" >}})
- [Domain Driven Design articles by Martin Fowler](https://martinfowler.com/tags/domain%20driven%20design.html)
- [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Transactions in Apache Kafka](https://www.confluent.io/blog/transactions-apache-kafka/)

Thanks for stopping by, cyaz! 👋