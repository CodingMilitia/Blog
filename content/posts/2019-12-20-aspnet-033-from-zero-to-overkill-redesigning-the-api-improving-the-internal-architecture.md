---
author: João Antunes
date: 2019-12-20 18:25:00+00:00
layout: post
title: "E033 - Redesigning the API: Improving the internal architecture - ASPF02O"
summary: "In this episode, we'll take a look at redesigning an API that has been developed in a traditional layered way, moving to a more interesting onion/hexagonal/ports and adapters architecture (which is apparently very fashionable right now in .NET land)."
images:
- '/images/2019/12/20/aspnet-core-from-zero-to-overkill-e033.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- onion architecture
- hexagonal architecture
- ports and adapters
- clean architecture
slug: aspnet-033-from-zero-to-overkill-redesigning-the-api-improving-the-internal-architecture
---

In this episode, we'll take a look at redesigning an API that has been developed in a traditional layered way, moving to a more interesting onion/hexagonal/ports and adapters architecture (which is apparently very fashionable right now in .NET land).

**Note:** depending on your preference, you can check out the following video or skip it to the written version below.

{{< yt 3Nuc-rXxaEA >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

Our group management API - the only we have so far, but that will eventually change - is pretty simple at the moment, but we want to start adding more features.

To keep things simple and familiar, until now we followed a pretty traditional layered architecture, with a presentation layer (in this case the web API project), a business layer and a data access layer. Besides this usual layering approach, the implementation tends to follow a common pattern: the controllers call a service (many times with basically the same name), the service has all the logic

These days, I'm not a big fan of this layered approach, particularly on the "services have all the logic" front, so we'll go with a more interesting (in my opinion, as everything I write on the blog), [onion/hexagonal/ports and adapters architecture](https://blog.ploeh.dk/2013/12/03/layers-onions-ports-adapters-its-all-the-same/), with some [DDD](https://en.wikipedia.org/wiki/Domain-driven_design) ideas added to the mix.

In this episode, I'll just introduce this reorganization of the internal API architecture. In the next episodes we'll go into more implementation details, as we add more features and learn new bits to use while developing the application.

Another note I'd like to leave here is that I don't believe in the "one true way™", so in other components of the application we develop later, we'll look at other approaches. I think we should take these kinds of articles/courses/videos not as the canonical source on how to do things, but as a source of ideas, that we can inspire ourselves in, adopting what makes sense in our situation, dropping what doesn't, adapting to each project and team specificities.

## Onion/hexagonal/ports and adapters architecture

Just to take this out of the way, I'm aware that by putting all these names together, some people might not be completely happy, arguing they're not the same, but at the end of the day, the main principles advocated by all these buzz words are the same, so I'm sticking with it, following [Mark Seemann's lead](https://blog.ploeh.dk/2013/12/03/layers-onions-ports-adapters-its-all-the-same/).

There are a ton of articles on the subject, like the original [The Onion Architecture : part 1](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/) or the always present [Wikipedia](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)), so I want go into a ton of detail, but present the gist of it, along with how I'm mapping it to this specific implementation.

In this episode, we'll just go for a quick look, particularly at what I feel are the most important difference when comparing to the layered architecture.

## The layered approach

The most common way I've seen implementing the layered approach, being reflected in the group management API until now (even though there wasn't a lot of logic), conforms to the following flow:

- Presentation layer - controller receives request, passes it on for a service to process
- Business layer - service does basically everything, then pushes data to persistence
- Data layer - repository (EF Core in this case) persists things in the database

In this approach, data layer is only known by the business layer, which in turn is only known by the presentation layer. In passing data between layers, a lot of mapping goes on in there, to enforce these abstractions.

## Comparing with onion/hexagonal

In terms of flow, the onion/hexagonal architecture is similar to the layered one.

For me, the most important principle of the onion/hexagonal architecture is to make the domain/business the focus of the project, everything else is "secondary" - not really secondary as it needs to be there for everything to work, but in terms of not being part of the core business domain, being an implementation detail.

The domain/application core contains everything needed to implement the business logic, like classes that represent the domain concepts and services that implement some logic. Additionally, as to implement the logic we depend on things that are not part of the core - like accessing a database or a web service - it includes interfaces (or ports, from ports and adapters) that components external to the core implement.

In the onion architecture (applied to a web application), these dependency interfaces are usually implemented in an infrastructure related component, while for instance receiving HTTP requests is done by a web component that then forwards it to the domain code that can handle it.

## Rationale for the redesign

This redesign is mostly focused on some things I'm not a fan with the traditional approach, even if not inherent of it, happen commonly.

- My greatest gripe is that most of the times, the services, in terms of interface, mirror the operations exposed by the controllers, then, in terms of implementation, have **all the logic**. This results in, what I called in [a recent presentation](https://github.com/joaofbantunes/BackToBasicsTheMessWereMakingOutOfOOP), a super service. We should try to break things apart, into smaller, more understandable pieces.
- It's common to wrongly focus on designing the data layer first, just then writing the services to manipulate the data.
- To pass information between layers, we create classes (DTOs) in each layer level, just to make sure, for instance, that the presentation layer doesn't ever see a class from the data layer. On some situations this may make sense (e.g. DTOs used on serialization/deserialization of requests/responses), but doing it **everywhere**, we end up with an immense amount of mapping code.

## About this implementation

Talking about using onion/hexagonal architecture is a bit abstract, so let's go for a quick overview of how we're reorganizing things in the group management API.

In terms of project structure, we'll go with 3 main ones:

- Domain - where we'll implement all the core application logic
- Infrastructure - where we'll implement the infrastructure dependencies required by the domain
- Web - where we expose the HTTP API, then forward the requests to the domain

Note that I'm going with this separation of things into projects, just because I like it. I feel like it makes it a bit harder to mistakenly reference things (e.g. referencing infrastructure things directly from the domain). Other people dislike this artificial separation, considering that project splitting should be used for separate deployments. Honestly, I'm in the do what you prefer camp on this one, so 🤷.

Inside the most important project, the domain one, I'm splitting things up in some main top level folders:

- Entities - in here, we'll have the classes that represent domain concepts (e.g. group, user, player, ...).
- Use cases - this folder will contain the application use cases, i.e. the operations that are implemented by it. Some examples are create a group, delete a group, get the user's groups. We can think of these use case classes as single operation domain service. Instead of a giant service implementing a bunch of use cases, we split them up, better respecting a bunch of SOLID principles (namely single responsibility, open-closed and interface segregation).
- Services - for now we don't have any service in there, but this will be where we put some specific domain logic, that can be reused across use cases and doesn't make sense to be part of any particular entity.
- Data - in the data folder, we'll add interfaces related with data access that we need the infrastructure component to implement.

In the other projects, there isn't much to talk about:

- The infrastructure project has implementations of the interfaces defined in the domain project.
- The web project remains pretty similar to how it was before, but now instead of calling services (the `GroupsService` more specifically), it's using [MediatR](https://github.com/jbogard/MediatR) to forward the requests to be handled by the domain implementation. Also moved from having `Controllers` and `Models` folders to a `Features` one, where things will be split by family of features they implement (e.g. group related features) instead of kind of class.

As mentioned earlier, we'll go into more details on the actual implementation in the coming episodes, but if you want to take a look, just head to the [GitHub repo](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode033). Keep in mind though, that some things will change.

## Over-engineer much?

If you're not thinking it already, you will once you look at the code in the coming episodes.

So, is this over-engineered? I think, to some degree, yes, particularly in a small API like this one. I like the rules and structure that this approach kind of enforces, but also realize it may be a bit overwhelming and, depending on the development process, we might not make use of the full potential it could bring.

That's why I believe it's good to see different approaches, and we'll do just that when implementing different parts of the PlayBall application, so we look at the pros and cons, being able to decide for a specific project, what we feel is more appropriate.

The approach I'm taking for the group management API is partially based on the recommendations of [Steve Smith](https://twitter.com/ardalis), as you can see from his [clean architecture repo](https://github.com/ardalis/CleanArchitecture) and Microsoft's reference application [eShopOnWeb](https://github.com/dotnet-architecture/eShopOnWeb), to which Steve contributed.

Another different approach, but also very interesting is the one from [Jimmy Bogard](https://twitter.com/jbogard), that he calls [vertical slice architecture](https://jimmybogard.com/vertical-slice-architecture/), and you can see an example of it [here](https://github.com/jbogard/ContosoUniversityDotNetCore).

I try to gather the best of both, but as these differ in some key points, I ended up pending a bit more towards the first one, with sprinkles of my own (and others) ideas thrown in.

## Outro

That's about it for this episode. Hopefully this episode shed some light on the project direction, and the architectural decisions made, in this case regarding the group management API.

I'm not a fan of heated arguments that often occur on the interwebs, with people saying one way is completely wrong, their ideas are the ones that are right and so on. I prefer to take a more nuanced stand, learning from the approaches people share. It seems more helpful to understand the ideas and where they make sense, using what we feel appropriate in each situation, instead of choosing a side and just always sticking to it. That doesn't mean there aren't some things really wrong and others really right, but there's also space in between 🙂.

Links in the post:

- [The Onion Architecture : part 1](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/)
- [Hexagonal architecture (software)](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software))
- [Layers, Onions, Ports, Adapters: it's all the same](https://blog.ploeh.dk/2013/12/03/layers-onions-ports-adapters-its-all-the-same/)
- [Domain-driven design](https://en.wikipedia.org/wiki/Domain-driven_design)
- [MediatR](https://github.com/jbogard/MediatR)
- [Steve Smith's Clean Architecture template](https://github.com/ardalis/CleanArchitecture)
- [Microsoft eShopOnWeb ASP.NET Core Reference Application](https://github.com/dotnet-architecture/eShopOnWeb)
- [Vertical Slice Architecture](https://jimmybogard.com/vertical-slice-architecture/)

The source code for this post is spread across the repositories in the ["Coding Militia: ASP.NET Core - From 0 to overkill" organization](https://github.com/AspNetCoreFromZeroToOverkill), tagged as `episode033`.

- [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode033)
- [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode033)
- [WebFrontend](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode033)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!
