---
author: João Antunes
date: 2024-12-16 19:20:00+00:00
layout: post
title: Care about the domain, not about implementing all the patterns and principles under the sun
summary: Time again for a rant adjacent post, on this occasion, about building applications following some domain-driven design ideas.
images:
  - /images/2024/12/16/care-about-the-domain-not-about-implementing-all-the-patterns-and-principles-under-the-sun.png
categories:
  - smalltalk
tags:
  - ddd
slug: care-about-the-domain-not-about-implementing-all-the-patterns-and-principles-under-the-sun
---

## Intro

Time for a short, wouldn't say rant, but rant adjacent post about building applications following some domain-driven design ideas.

This post is about what I believe to be the main value we can get from trying to apply DDD ideas, as well as what I see as more accessory. Of course, as always, I'm basing myself on my learnings, my experience and context of the projects I've been involved in. In any case, I've been building stuff for long enough to be somewhat confident I'm not just spewing complete nonsense 😅.

## Focusing on the wrong things

I'll go straight to the point: from what I see, developers tend to focus on auxiliary things, some that are actually related to DDD (like tactical patterns), others not even that (e.g. project structures, layers, "clean architecture" and the like).

I may be totally wrong here, but for me, the main point of domain-driven design should be, well, how do I put it... focusing on the domain? 😅

## An (exaggerated) example

Let's imagine we're building a service to report and track incidents as they happen and get resolved. A couple of example features: report a new incident; assign a team to handle it. This service is a part of a bigger system, and we want to use an event-driven strategy for integration between the services.

For this example, let's consider two approaches to building an API that implements the aforementioned features (without going into a lot of detail).

As the first approach, we can implement the first endpoint, by receiving the required information, then invoking a function named `ReportIncident`. This function inserts the data into a relational database with a simple `insert` statement, plus publishes an `IncidentReported` event.

The second feature is implemented in a similar way, by having an endpoint invoke a function named `AssignTeamToIncident`, which in turn inserts the association between the team and the incident to the database (likely with another simple `insert` statement), then publishing a `TeamAssignedToIncident` event.

As an hypothetical second approach, we have a first endpoint implementation which sends a `CreateIncidentCommand` instance to an in-memory bus. This command handler uses an `IncidentFactory` to create an `IncidentEntity`, which is then persisted using an `IncidentRepository` (which inherits from a `GenericRepository`). By the end of the process, an `IncidentCreatedEvent` is published.

Again, the second feature is implemented in a similar fashion. An endpoint is exposed, which sends an instance of `UpdateIncidentCommand` through an in-memory bus, where its handler updates the `IncidentEntity` with the team information, saves it to the database and publishes an `IncidentUpdatedEvent`.

Now, at first glance, which of the two approaches would you say better aligns with DDD ideas? For me, without a doubt, the first one. While the second may use all the patterns and tools some have come to associate with what DDD is, the first one actually focuses on the domain, using the same language as the end-users of the application.

## Taking a step back

Granted, in the previous section, I exaggerated things a bit (or have I?) to make a point, but I really believe folks are focusing on the wrong things, ending up over-engineering solutions, with little to no benefit, because, again, the focus was on the wrong thing.

With this, I'm not saying all of those auxiliary patterns and principles are useless, just that we should start with a focus on understanding what problem our application is trying to solve for people, tailor it for the problem, use the same language as the users, so that everyone is on the same page, ensuring what we're building is what is needed.

Focus on the domain, and on structuring the code in the way that better allows you and your colleagues to be efficient working with it. If applying any of the previously mentioned patterns and techniques benefits this end-goal, then use it, otherwise, if it doesn't objectively help your team ship faster, higher quality software, then it's just useless added complexity.

Instead of defaulting to adding everything and the kitchen sink from the get go, default to the simplest possible thing, adding more complex patterns and abstractions only when they're actually needed. You'd be pleasantly surprised by how much more productive you can be with simpler code and less abstractions 😉.

## Outro

That does it for this very quick post, hope it did make some sense.

Keep in mind this is coming from someone who also built over-engineered software, trying to apply all possible patterns that seemed to make some sense, to then realize I wasn't getting any benefit from it. At some point I'll write a more general post about learning and changing opinions over time, but for now, I just wanted to emphasize a simple message: focus on the domain.

Thanks for stopping by, cyaz! 👋