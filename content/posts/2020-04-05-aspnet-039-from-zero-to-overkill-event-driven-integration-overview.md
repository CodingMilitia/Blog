---
author: João Antunes
date: 2020-04-05 11:30:00+01:00
layout: post
title: "Event-driven integration - Overview [ASPF02O|E039]"
summary: "In this episode, we take a look at the integration problem we have between services right now, go through some options to avoid or fix it, then have a quick overview of the solution we'll be implementing in the coming episodes."
images:
- '/images/2020/04/05/aspnet-core-from-zero-to-overkill-e039.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-039-from-zero-to-overkill-event-driven-integration-overview
---

In this episode, we take a look at the integration problem we have between services right now, go through some options to avoid or fix it, then have a quick overview of the solution we'll be implementing in the coming episodes.

**Note:** depending on your preference, you can check out the following video, otherwise, skip to the written version below.

{{< yt rRAO_LoEe4w >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

If we try to grab all the source code for the PlayBall project and run it, we'll notice that after registering as a new user, if we try to create a new group, the group management API will fail. That's because the group management API keeps a view of the users' information, but when we register a new user (or update or delete an existing one) in the auth service, we're not communicating that to group management service.

In this episode we'll see some possibilities to tackle this problem, finishing up with the steps we'll take in the coming episodes to fix this issue, while we also lay the groundwork for more integrations following the same patterns.

## The problem and approaches to avoid or fix it

As introduced, the problem we have right now is that the group management service keeps a view of the users' information, with some fields that come from the auth service. However, there's no integration going on between the services, so when a new user is registered, the group management service knows nothing about it, and when the user does some action on its API, it'll fail.

{{< embedded-image "/images/2020/04/05/e039-user-sync-with-original-issue.png" "no integration" >}}

We have some options to avoid or fix this problem, so let's quickly list them and then detail a bit:

- Use the information from the JWT/user info endpoint
- Keep only the user identifier, obtained from the JWT/user info endpoint
- Use an event-driven approach to communicate to other services things happening in one

### Use the information from the JWT/user info endpoint

Instead of relying on some other kind of integration, we could obtain the information we need either from the JWT the API gets on every request or the user info endpoint exposed by the auth service.

This approach would solve our user registered but not known by the group management service issue, but not other related issues. Assuming we store the user's name, what happens when it is updated in the auth service? The group management service would still be oblivious to this change, as it stored the required information when it detected a new user, but after that, it assumes it's all good. We could additionally check for changes on every request, but it seems a bit overkill and wouldn't avoid yet another issue: what if the user deletes the account? In such a case, no more requests will happen, leaving the group management service in the dark.

### Keep only the user identifier from the JWT

Keeping only the user identifier that we can get from the JWT is a simplified version of the previously discussed approach. It would avoid the user registration and modification issues, but not the case where the user deletes the account. Not only that, but it would make the group management service more coupled to the auth service than it needs to be, as if we require additional information to fulfill a request, we'd need to make additional calls. Sometimes we can't avoid it or it's just too complex, otherwise, minimizing coupling between services is ideal.

As for the final approach we'll go trough, being the one we'll implement, it warrants it's own top-level section in the post 🙂.

## Event-driven integration

As it's clear from the post title, the approach we'll be taking is integration through events. Recovering the initial diagram, we'll add the event bus as a new component to our overall architecture:

{{< embedded-image "/images/2020/04/05/e039-user-sync-event-driven.png" "event-driven integration" >}}

In a nutshell, we'll use the event bus to publish significant events that happen inside a service and might be of interest to other services. In this case we'll start with user account events - user registered, user updated and user deleted - but in the future, we can continue building on this, publishing more and more events from various services, informing others of relevant happenings.

A couple of examples of additional events we might add in the future might be:

- Group management service publishes a "group deleted" event, the statistics service removes all the related information
- Live matches service publishes a "match completed" event, the matches service can consolidate the match information, while the statistics service processes the stats for that match

Going with an event-driven approach means we're opting to go with an asynchronous communication strategy, in detriment of a more typical synchronous one. As usual, it's a tradeoff, and even if it has pros, it also has cons.

Some typical tradeoffs are:

- Asynchronous communication may allow for better decoupling between services. This decoupling is helpful to:
  - Simplify some development aspects, by avoiding the need to know how to invoke all other services.
  - Increase services resilience, by not needing all the services to be up to fulfill a request, things eventually synchronize (you might have already heard the term eventual consistency a lot).
- Typically, an event-driven system tends to duplicate more data across services, to ensure that a service can fulfill its purpose independently. This independence comes at the cost of the complexity to keep things in sync (again, eventual consistency and its challenges come into the discussion).
- A service being independent may allow it to be faster at its task. Take an API as an example, instead of making requests to multiple services to fulfill a single inbound request, it can grab all the required information locally.

For some more reading on sync vs async communication and event-driven approaches, check the end of the post for a couple of links from Microsoft docs and Martin Fowler's blog.

## Next steps

Even if going with an event-driven approach has advantages, it is complex to implement, so we'll take our time to go through it, and I'll try to split things into a bunch of episodes to keep it small and focused. That's why this post exists, even if with little content, to be an introduction to what's coming next.

For the coming episodes, my idea is to go through the following topics:

- Detect user account events when saving changes through EF Core
- Implement the outbox pattern to ensure at least once delivery
- Briefly introduce Apache Kafka, the platform we'll use to implement our event bus
- Implement an event publisher on top of Kafka
- Implement an event consumer on top of Kafka
- Ensure idempotency of event processing - in simpler wording, ensure an event isn't processed multiple times

These are the next steps I have in mind right now, but it's not written in stone, so changes in planning are not impossible.

## Outro

That's a wrap for this quick overview of the current issues we have regarding integration between services in the application, as well as looking at some options to tackle the project.

Coming in the next episodes, we'll start to work on fixing the issues, taking an event-driven approach to communicating relevant things happening in these services.

**PSA:** Friendly reminder that what we just discussed, as well as the related incoming developments are complex and wouldn't be needed if this simple application we have was a monolith as it probably should. Always remember the "overkill" in the title of the series and think if the complexity of some of the things we go through here makes sense for your use case.

Related links:

- [What do you mean by “Event-Driven”?](https://martinfowler.com/articles/201701-event-driven.html)
- [Event-driven architecture style](https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven)
- [Communication in a microservice architecture](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/communication-in-microservice-architecture)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!