---
author: João Antunes
date: 2025-06-11 08:30:00+01:00
layout: post
title: How I've been building APIs and microservices lately (feat. C# & .NET)
summary: This post is all about how I've been building APIs and microservices lately (using .NET and C#), what's been making me productive, and how my approach has evolved over time.
images:
  - /images/2025/06/11/how-ive-been-building-apis-and-microservices-lately-feat-csharp-dotnet.png
categories:
  - csharp
  - dotnet
tags:
  - ddd
  - microservices
  - api
slug: how-ive-been-building-apis-and-microservices-lately-feat-csharp-dotnet
---

## Intro

This post is all about how I've been building APIs and microservices lately (using .NET and C#), mostly in work context, but with a couple of personal preference related tweaks here and there.

It is not my goal to tell you that this is the way you should do things. I'm just sharing the approaches I've been feeling productive with, particularly when compared to previous ways of doing things.

Do note that most of what I'll be talking about is inspired by what I've learned from others over the years. What I think is the relevant part here (relevant for those interested, of course), is that as time goes one, we encounter different and even contradicting opinions, so how I write code, and how I'll describe it in this post is basically cherry picking, summarizing and embracing the opinions I've come to agree more with experience (and I'm pretty sure these opinions will evolve in the next few months/years).

I prepared an [accompanying repository](https://github.com/joaofbantunes/HowIveBeenBuildingAPIsAndMicroservices) where you can check everything out. I'll embed a couple of snippets in the post, but it's a good amount of code, so to see the whole thing, better to check out the repo.

## Some initial context

It's probably worth it to start with a bit of context about the kind of software we build at my current company, as that affects a bunch of the decisions regarding the approaches taken.

To support a business in the mobility area (toll collection, infrastructure management, field force management, etc), we develop a cloud hosted, microservices based, event-driven platform (so many buzzwords 😂).

As much as possible, we avoid synchronous communication between services, relying heavily on events instead (though I won't be talking much about events in specific in this post).

The majority of our microservices do expose HTTP APIs, which are used by our frontend applications, typically via tailored backends for frontends (aka BFFs). Many of these APIs are also exposed to external clients, so well designed APIs are even more important than if they were simply meant for internal use.

Taking advantage of one of the most interesting benefits of using microservices, we're polyglot, meaning we build services using different tech stacks, depending on context and needs - in this post I'll be focusing on .NET.

## Vertical slice-ish

Let's start by one of the topics folks tend to waste a lot of time debating, even if it's not as relevant as the constant yapping would lead one to believe: structuring a project. I mean, structuring a project is important, as it should be done in a way that allows one to work efficiently, it's just not a relevant topic to warrant the constant discussion (and here I am, contributing to that discussion, like an idiot 😅).

### Single vs multiple projects

Even though, like many, I initially thought splitting a solution into multiple projects (one executable and a bunch of extra libraries) made sense, I've come to agree with those that advocate for having a single project in the cases where it's all going to generate a single binary in the end, never deployed independently.

The whole "but this avoids folks putting things in the wrong layer" seems to make sense in theory. In practice though, if folks don't understand the ideas, they'll put things in the wrong place anyway. We should invest in teaching people how separate concepts correctly, instead of creating artificial barriers that are easily broken, while at the same time overcomplicate the solution unnecessarily.

Do keep in mind the context I presented earlier though. Would splitting the solution into multiple projects make sense in the case of a large monolith? Perhaps, but I’m not the best person to tell you. I’ve been building microservices and other smaller, independently deployed applications and services for a long time ([SOA](https://en.wikipedia.org/wiki/Service-oriented_architecture) was all the rage when I was starting my career), so a really large monolith isn’t something I have relevant experience with.

### Slicing things vertically (mostly)

Regardless of the amount of projects used, another overly-debated topic remains: should we have a bunch of explicitly named layers (e.g. infrastructure, domain, application), or should we just skip them?

My opinion: skip them. That's not to say to have everything mixed together in a single file. For example, having SQL code together with domain logic in C# is probably not ideal, as it's mixing different levels of abstraction (unless you like to write domain logic in SQL, in which case, you do you 😉), but you don't need to spread things all over your project/solution just to clean things up.

I try to follow the ideas around vertical slices (made popular by [Jimmy Bogard](https://www.jimmybogard.com/vertical-slice-architecture/), or at least made popular with this name, as far as I'm aware), splitting into folders based on features and capabilities, instead of layers or other less obvious alternatives.

As an example, inside the API project, I'd structure things something like this:

```sh
├── Features
│   ├── Orders
│   │   ├── CancelOrder.cs
│   │   ├── GetOrderDetails.cs
│   │   ├── RegisterOrder.cs
│   │   ├── ...
│   ├── Shared
│   │   ├── Data
│	│   │   ├── EntityTypeConfigurations
│	│   │   ├── Migrations
│	│   │   ├── AppDbContext.cs
│	│   │   ├── ...
│   │   ├── Domain
│	│   │   ├── Dish.cs
│	│   │   ├── Miscelleaneous.cs
│	│   │   ├── Order.cs
│	│   │   ├── OrderDish.cs
│   │   ├── Http
│   │   ├── Observability
│   │   ├── ...
│   ├── ...
└── ...
```

Inside a `Features` folder, I'd put a folder for each high level feature. In the sample code I prepared, only added one, the `Orders` feature. For code that is shared among different high level features, I put it in, well, a `Shared` folder.

Inside a specific feature folder, we put the slices that implement the feature. In this example, being a demo API, we have a slice per endpoint exposed, with a single file for each. The same would apply if we had other kinds of triggers, like events or scheduled jobs.

Because the sample slices are simple enough, I'm putting their whole implementation in a single file (minus domain, to discuss in a bit). If they were more complex, then I'd create a folder and put the relevant code in it. If there is some code that is shared among slices within a feature, but not shared with other features, then I'd just create a `Shared` folder inside the feature folder.

In the `Features/Shared` folder, the majority of things end up being more infrastructure related, as that's typically what's more generally applicable throughout the application. The big exception to this "rule" is the domain.

Ideally, we'd put the domain code inside a feature folder as well. However, there are a couple of issues that limit a bit the ability to do so.

The first potential issue, is that there are situations in which different features make use of the same domain logic.

Another one, is that due to the convenience of using the domain model as the data model, along with something like EF Core, with its navigation properties and the like, our entities end up a bit intertwined with each other. There are ways to tackle this, but we either lose some of the convenience, or things start to get needlessly complicated, so we need to consider the trade-offs.

Besides the domain bits, the rest of the code in the `Features/Shared` is more obvious as to why it's there.

The EF Core `DbContext`, along with other related bits (migrations, entity type configurations, etc) can be found in this folder. Some observability related helpers are also in there, along with other such auxiliary things.

In this sample though, there's also a bit of extra boilerplate code found in this shared folder. At work, most of this is pushed to internal NuGet packages, as it's easily reusable across services. Some of it is just glue code, other is code that helps in following some of our internal guidelines (e.g. around the usage of [problem details](https://www.rfc-editor.org/rfc/rfc9457.html)).

### On "clean architecture" and the like

Just to address the mammoth in the room (for C# folks at least, doesn't seem like the majority working on other stacks cares about this): "clean architecture" and other related prescriptive project structuring approaches. If it wasn't clear until now, over the years I've come to not care much about them.

That's not to say some of the underlying ideas aren't useful. They are, and are shared with other approaches, so I do make use of some of them. Restating the example I mentioned earlier: having domain code and infrastructure code cobbled together isn't the best idea, so it's good to split it, but it doesn't mean we need to spread it 100 kilometers apart.

One amusing tidbit I personally enjoy, is the C# community obsession with using MediatR to implement clean architecture, when the guy behind MediatR is the same guy who popularized vertical slice architecture and advocated against spreading things as it seems to be the norm in C# solutions, including splitting things into multiple projects.

## Implementing a slice

### Overview

After all this talk, let's look at a bit of code, to see how I'd implement a slice. In this example all the slices are HTTP endpoints, but as mentioned, there could be other types of triggers, like events or scheduled jobs.

I'll just drop the code for the cancel order slice, and then we'll dissect it a bit.

```csharp
public static class CancelOrder
{
    private sealed record Request
    {
        [FromRoute] public required Guid OrderId { get; init; }
    }

    private sealed class Endpoint(AppDbContext db, TimeProvider timeProvider)
	    : EndpointBase
    {
        [HttpPost("/api/v1/orders/{orderId}/cancel")]
        public async Task<IActionResult> HandleAsync(
          Request request,
          CancellationToken ct)
        {
            if (Validate(request).With<Validator>()
              is { IsValid: false } validationResult)
            {
                return this.CustomValidationProblem(
                    validationResult,
                    ApiErrors.General.Validation,
                    "Invalid request");
            }

            var parsedOrderId = OrderId.Parse(request.OrderId);
            var order = await db.Orders.FirstOrDefaultAsync(
                o => o.ExternalId == parsedOrderId, ct);

            if (order is null)
            {
                return this.CustomProblem(
                    HttpStatusCode.NotFound,
                    ApiErrors.General.NotFound,
                    "Order not found");
            }

            var result = order.Cancel(timeProvider.GetUtcNow());

            if (result.TryGetValue(ok: out var @event, out var fail))
            {
                db.AddDomainEvent(order, @event);
                await db.SaveChangesAsync(ct);
                return NoContent();
            }

            return fail switch
            {
                CancelOrderError.NoLongerCancellable => this.CustomProblem(
                    HttpStatusCode.Conflict,
                    ApiErrors.Orders.OrderNoLongerCancellable,
                    "Order is no longer cancellable"),
                _ => throw new InvalidOperationException(
                  "Unexpected cancel order error")
            };
        }
    }

    private sealed class Validator : AbstractValidator<Request>
    {
        public Validator()
            => RuleFor(x => x.OrderId)
                .Must(id => OrderId.TryParse(id, out _))
                .WithMessage("Invalid order id format")
                .WithState(static _ => new ValidationMetadata
                {
                    Parameter = "orderId"
                });
    }
}
```

As you can see, and probably to the shock of many C# devs, **at least in these simple slices**, I put all of the slice specific code inside a single static class. Any required DTOs, input validators and the HTTP endpoint, everything is put in there and made private, as no other code should access it directly.

If there are some DTOs that should be shared, then I'd move them out, but I see it as a good practice to avoid sharing DTOs as much as possible.

The endpoint implemented here, inherits from `EndpointBase`, but this doesn't do much more than inherit from `ControllerBase`, add the `ApiController` attribute and serve as a target for a bunch of helper extension methods. I'm still thinking if I should add some more features to `EndpointBase`, to further reduce boilerplate code.

I initially wanted to use minimal APIs, but I had a couple of issues with missing features (mainly around binding/deserialization), so I ended up falling back to controllers. Hopefully the missing features are added in the future and I can drop controllers for good.

As you can see, I'm implementing the command handler logic directly in the endpoint, there's no `OrdersService`, MediatR or anything of the sort. That's because, looking back, I didn't get much benefit from doing so, so I just started skipping it altogether. 

The command handler logic (in this case at least) is rather straightforward. The idea is really for it to be straightforward most of the times, as the more complex logic should be extracted out. What the handler should do, is orchestrate different pieces of code that contain the bulk of the logic. In this case, the command handler:

- Accepts inputs and delegates their validation
- Informs the client of any validation issue
- Fetches the aggregate that will handle the domain logic
- If no aggregate is found, inform the client
- Invokes the command on the aggregate
- Based on the command result, it either:
	- Persists the changes and informs the client about the successful command execution
	- Informs the client about the issues found when executing the command

And for the overview, that's about it. I think anyone who has some programming understanding can look at this code and understand what it's doing. It has little magic, most things are explicit. 

Some will say that there's a bit of boilerplate code that can be extracted, and I'd partially agree. However, I don't want to replace some simple obvious code with a bunch of magic just to remove 5 lines of code. If I find a way to clean things up without adding complexity, I'll do it, otherwise, I prefer explicit simple code.

### Enter the repository pattern discussion

Let's quickly touch on another holy grail of C# developer discussions: the repository pattern, and using it to abstract ORMs like EF Core.

You figured out my opinion by now, given the direct usage of the `AppDbContext` in the slice implementation we've just looked at. As always, it's context dependent, but for the type of services I'v been building, abstracting an ORM like EF Core, which by itself already implements the repository and unit of work patterns, increases complexity with little to no advantage.

If everything you need to do is doable through extensibility points provided by your ORM, just do that. If, however, you need some place to put some extra logic, then that's the time to introduce something, be it an abstraction in between, or even some simple extension methods that enrich what the ORM can do.

### Rich domain model

An approach I typically try to take, is to build a rich domain model, in which I put as much of the domain logic I can into the types representing the domain (aggregates, entities, value objects, yada yada yada, all the DDD jargon).

If some domain logic doesn't fit in a particular type, I'll just add it as a static method in some static class, and add it to the domain folder. In my opinion, the domain doesn't need dependency injection (not in the way C# devs think about it), so using static methods to encapsulate domain logic seems more than a good fit for me.

Note that the domain model ends up not being super pure, as to use is along side EF Core means some of the ORM's requirements leak into the model (also true for other data access tools). Considering the tradeoffs against making the domain model really ORM agnostic, and all the boilerplate that would bring, I think this concession is acceptable.

Let's take a look at the `Order` aggregate used in the previously analyzed slice.

```csharp
public sealed class Order
{
    private readonly List<OrderItem> _items = new();

    public long Id { get; private set; }
    public OrderId ExternalId { get; private set; }
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public OrderStatus Status { get; private set; }
    public DateTimeOffset RegisteredAt { get; private set; }

    public static (Order Order, OrderRegistered Event) Register(
        IReadOnlyCollection<(Dish Dish, OrderItemQuantity Quantity)> items,
        DateTimeOffset now)
    {
        if (items.Count == 0)
            throw new ArgumentException("Order must contain at least one item", nameof(items));

        var order = new Order
        {
            ExternalId = OrderId.New(now),
            Status = OrderStatus.Registered,
            RegisteredAt = now,
        };

        order._items.AddRange(items.Select(d => new OrderItem
        {
            Order = order,
            Dish = d.Dish,
            Quantity = d.Quantity
        }));

        var @event = new OrderRegistered
        {
            OrderId = order.ExternalId,
            Items = [..items.Select(d => new KeyValuePair<DishId, OrderItemQuantity>(d.Dish.ExternalId, d.Quantity))],
            OccurredAt = now
        };

        return (order, @event);
    }

    public DomainResult<OrderCancelled, CancelOrderError> Cancel(DateTimeOffset now)
    {
        if (Status != OrderStatus.Registered)
            return new CancelOrderError.NoLongerCancellable();

        Status = OrderStatus.Cancelled;

        return new OrderCancelled
        {
            OrderId = ExternalId,
            OccurredAt = now
        };
    }
}
```

The example domain is rather simple, so there isn't much going on, but should be enough to demonstrate my approach.

The `Order` aggregate exposes methods implementing its related logic. It's creation is done via a static `Register` method, which returns the created entity instance to persist, as well as a domain event representing what happened. If there were domain errors that could occur, then that should also be a possible thing to return, but there isn't in this case, so the tuple with entity and event is the only thing returned.

The `Cancel` method, is an instance one, as it's acting on an existing entity, unlike the `Register` that creates one. This method returns either an `OrderCancelled` event, in case of successful cancellation, or a `CancelOrderError`, in case there's a domain error associated with cancelling the order.

Note the use of explicitly defined domain error types, to represent problems I want to communicate back to the user. Exceptions are used for things that aren't expected to happen. In this case, in the `Register` method, I'm not expecting to get an order with no items, as it should be validated out earlier, so I just throw.

You might have noticed that the way I return the result of domain operations is following the [result pattern](https://andrewlock.net/working-with-the-result-pattern-part-1-replacing-exceptions-as-control-flow/). While I enjoy this pattern and use it often, I've been following one rule: no chaining! Every time I get a result, I handle it imperatively, with an `if` or `switch` and return something, as you can see in the command handler. [Railway oriented programming](https://fsharpforfunandprofit.com/posts/recipe-part2/) is great, if the language has the right idioms to make it work cleanly. C# doesn't, so I don't do it - I did try it, and it kind of works until a certain point, but then things start to get messy, fast 😂.

Some folks aren't super fans of this way of handling domain errors, but I massively prefer super explicit code with ifs, over linear looking code, which is throwing exceptions behind the scenes, then catching them half-way across the codebase. By the way, this way of handling errors is somewhat inspired by the way it's done in [Go](https://go.dev/blog/error-handling-and-go).

### Interesting alternatives to a rich domain model

While I've been using a rich domain model, with an object oriented mindset, it's certainly not the only option, so I want to take the opportunity to reference a couple of interesting alternative ideas.

The first one is the [decider pattern](https://thinkbeforecoding.com/post/2021/12/17/functional-event-sourcing-decider), which I learned about from an [Oskar Dudycz post](https://event-driven.io/en/testing_event_sourcing/). It's a more FP way of doing things.

Another interesting idea, is applying the [A-Frame Architecture](https://www.jamesshore.com/v2/projects/nullables/testing-without-mocks#a-frame-arch) to handlers, which is presented in the linked blog in the context of a large, but very interesting post about testing without mocks. Jeremy Miller [talks about implementing it with C# and his Critter Stack](https://jeremydmiller.com/2023/07/19/a-frame-architecture-with-wolverine/). I believe the way I do things isn't super far-off from this idea, but I do implement it quite differently.

Besides these two, there are other interesting approaches, particularly in the FP world, which at some point I'd like to give it a whirl, but I'm unsure they'll be a good fit for C#, at least while using the same libraries (e.g. EF Core).

In any case, the way I do stuff, and am presenting in this post, is already seen as "disruptive" by many colleagues, so I can only imagine what would happen if I pushed these alternatives 😂. Anyway, these are just a couple of examples of super interesting and valid approaches to solve the problem of achieving good testability of domain logic, without mocking half the planet.

### Validation

In the unlikely event you've read this blog before (if that's the case, thanks!), you might have read the post  I yapped about [parsing instead of validating](https://blog.codingmilitia.com/2023/08/17/parse-dont-validate-and-other-type-safety-driven-shenanigans-plus-a-csharp-wishlist/), and wonder, what happened to no validating? 😅

Well, a bit like railway oriented programming, I didn't find a super clean way to achieve it in C#, at least not one that scales to more complex, non-demo projects. If things start to get in the way, it's time to drop them (but keep the learning for the future).

So, at the end of the day, reverted back to using [FluentValidation](https://docs.fluentvalidation.net/), which is a super nice validation library many folks are already used to.

Main difference with the usage of FluentValidation, is that I keep the validator code together with the rest of the slice, and don't use DI nor async stuff, as I don't feel like validating request parameters requires any of that.

I also created some helpers to use it (the `Validate(request).With<Validator>()` seen earlier), though I'm still thinking about better approaches.

Side note: it's probably a bit overkill to create a validator just to validate an id, as is the case in the example slice, but I just wanted to exemplify the pattern I've been using.

### Problem Details

Not going to talk a lot about problem details, as I'm planning a post about our adoption of problem details at work, cause I think it would be useful to present something real world, in contrast to the more demo like content available online.

In any case, as it shows up in the context of this post, it's worth mentioning that we started using it very deliberately to thoroughly inform our API clients of what went wrong.

You can see in the example slice code, as well as the rest of the demo code in the repo, that all the errors that might be relevant for the API clients are mapped and responded to them (of course an unexpected error will result in a 500 without further details to the client). The possible error types are also included in the OpenAPI document, so clients know what to expect.

## Contract first API development

I've wrote about [contract first OpenAPI](https://blog.codingmilitia.com/2023/04/02/contract-first-openapi-development-but-still-use-swagger-ui-with-asp.net-core/), and that's something we've been adopting at work. For some reason, it seems this adoption causes much annoyance to many colleagues (namely .NET devs, Java folks were already used to it).

The gist of it is, design the API contract before implementing the API, which translates to writing the OpenAPI doc manually, no auto-generated stuff.

To be clear, I don't think this is a mandatory way of working in all contexts. For example, if you're building a full-stack web app, want to use OpenAPI to generate client code for frontend, the auto-generated stuff is more than good enough and it would be a waste of time not take advantage of it.

However, if the context is building a public API, one for broad use by different internal teams or, more importantly, by external clients, then carefully crafting a contract becomes much more important. From what I’ve experienced, folks tend to treat API design as an afterthought, so my hope is to counter that with a contract first approach - wish me luck 🤞.

There are two main reasons I believe writing the contract manually is important, as much as writing a large YAML file is annoying. For starters, it allows us to create some distance between the API design and implementation, making it easier to think from an API client perspective. Additionally, despite the fact the contract generation tools available are excellent pieces of software, they still have limitations and can add unnecessary hurdles that distract or even prevent us from providing the best contract possible.

As for the example in this post, you can see in the repo that the contract is written manually, living in `wwwroot`. We can still serve Swagger UI or some alternative (in this demo I used [Scalar](https://guides.scalar.com/scalar/scalar-api-references/net-integration) just to try it out) to make it easier to poke around the API.


The defined contract is also used in tests, as we'll see in the next section.

## Testing

Now, let's take a look at testing.

The first thing many would point out looking at the code so far, is that by having the command handler code directly in the endpoint, I can't unit test it. Well, good thing I wasn't planning to anyway.

I don't want to unit test a command handler, with all of its dependencies, wasting time mocking a bunch of stuff, while also not testing all the relevant things (e.g. database interaction).

In terms of unit tests, I focus on the domain and other independent components. Tends to be fast and easy to do.

A very simple example would look like the following:

```csharp
[Fact]
public void WhenCancellingCancellableOrderThenItIsCancelledSuccessfully()
{
    var now = new DateTimeOffset(2025, 05, 31, 13, 17, 0, TimeSpan.FromHours(1));
    var orderItems = new[]{(TestData.Dish, OrderItemQuantity.From(3))};
    var (sut, _) = Order.Register(orderItems, now);
    
    var result = sut.Cancel(now);

    var @event = result.Value.ShouldBeOfType<OrderCancelled>();
    @event.OrderId.ShouldBe(sut.ExternalId);
    @event.OccurredAt.ShouldBe(now);
}
```

Nothing fancy, just setting up an `Order` then calling `Cancel` on it, asserting some bits (there's more stuff that should be asserted, but keeping it light).

As for the endpoint/command handler, I "integration" test it (or whatever you wanna call it), meaning I mock very little when I can't avoid it (e.g. mock calls to external APIs), while using the closest thing possible to production (e.g. use a real database during testing). For the latter, I've been relying on [Testcontainers](https://testcontainers.com).

I take advantage of the OpenAPI contract to generate a client using NSwag, then use it in tests. This has two benefits: tests are simpler to write, plus we do contract testing, ensuring the contract provided to API clients is correctly implemented.

An example test, would look like this:

```csharp
[Fact]
public async Task WhenRegisteringOrderWithUnknownDishThenItReturnsError()
{
    var unknownDishId = Guid.CreateVersion7();
    var act = () => _client.RegisterOrderAsync(new RegisterOrder
    {
        Items =
        [
            new OrderItem
            {
                DishId = unknownDishId,
                Quantity = 1
            }
        ]
    });

    await act.ShouldReturnProblemDetailsAsync(
        HttpStatusCode.UnprocessableEntity,
        ApiErrors.Orders.UnknownDishes,
        "Some dishes are not known",
        "Some dishes are not known",
        new UnknownDishesErrorDetail
        {
            DishIds = [unknownDishId]
        },
        (actual, expected) => actual.DishIds.ShouldBe(ß
          expected.DishIds,
          ignoreOrder: true));
}
```

Again, nothing fancy, just using the client to make a request, then asserting the response is what's expected. You can even TDD this if you so desire (I've done it here and there).

## A quick note on event-driven

While this post sample code has been mostly focused on APIs, the same way of thinking applies to other triggers, such as events and scheduled jobs, of course with differences in implementation.

Find the place in the feature organization where a given handler fits, put it there, along with any specific code. If there's code that's reusable, push it into whatever shared folder it makes sense to be in. Keep related things together and don't go overboard with abstractions.

Even contract first still applies to events, as we can design our contracts (we use [Avro](https://avro.apache.org) at work) before implementing their publishing and handling.

A quick shout-out to the [transactional outbox pattern](https://outboxkit.yakshavefx.dev/intro/transactional-outbox-pattern.html), which while not making an explicit appearance in this post, is lurking behind the scenes. When in the endpoint implementation you see `db.AddDomainEvent(order, @event)`, what this does is map the domain event into 0 or multiple integration events, which are pushed into an outbox table for later publishing.

## Miscellaneous

There's a bunch more subjects that are relevant to building a microservice, but probably not worth a whole section, so will just drop a quick bullet list to mention them.

- [OpenTelemetry](https://opentelemetry.io) all the things, not only taking advantage of existing instrumentations, but also instrumenting our own code to make it easier to understand what's going on in a service
- [Docker Compose](https://docs.docker.com/compose/) for dev time dependencies
- [`.editorconfig`](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/code-style-rule-options) to enforce code formatting and style
- Templates (or [chassis](https://microservices.io/patterns/microservice-chassis.html)) to streamline the setup of a new microservices
- Internal NuGet packages to simplify following internal patterns and guidelines, plus get rid of some boilerplate code, though avoid going overboard and creating bloated internal frameworks
- No mention of authentication and authorization in this post, though it's of course super important, and we make use of ASP.NET Core's extensible support to implement something that works for us

## Progressively introducing complexity

What I've been trying to tell my colleagues more and more, is: does a given thing make you more productive, make it easier to deliver and maintain quality software? If so, do it/use it, otherwise drop it.

That's the motivation behind a lot of what I described throughout this post. If creating a bunch of layers, introducing repositories and other abstractions are not effectively contributing to create better software faster, then they're all just obstacles that are slowing us down, so we need to drop them and simplify things.

I think an important idea is that of progressively introducing complexity. I'm certain there are scenarios where additional abstractions are helpful, I just don't think it should be the default. Start simple, then introduce further complexity when you absolutely need to: slice too complex to fit in a single file? create a folder and split things; need an abstraction over a `DbContext` to add some cross cutting concerns? then do it - but only do this when it's needed, not by default, when most of the times a simpler approach will suffice.

## Outro

And that's a wrap for this post. I tried to distill the best I could how I've been building APIs and microservices, including the reasoning behind most decisions. I'm sure it's not a fit for everyone, but I'm also sure it's a better fit for many than what seems to be the status quo of .NET development.

Want to point out, if it isn't clear, that what I presented here is mostly the default approach I've been taking, doesn't mean everything is done exactly the same way, though the core ideas remain. As an example, we had to build a smaller but more performance sensitive component, in which case we used gRPC, Dapper and FusionCache, ignored isolating the domain (it was a readonly component though), and overall followed a slightly different design.

While I've been feeling productive with the approach described, I'm sure it can be improved, in particular trying to further reduce ceremony. I need to continue learning from folks like [Oskar Dudycz](https://event-driven.io), [Jeremy Miller](https://jeremydmiller.com) and others, which I believe are much better influences for .NET devs building these types of applications, than other more popular "influencers" in the space.

Even though I didn't spend much of the post talking about it explicitly, the main takeaway I'd love folks to take from it is that about progressively introducing complexity. Do the simplest things possible by default, add complexity when needed.

Relevant links:

- [All the code](https://github.com/joaofbantunes/HowIveBeenBuildingAPIsAndMicroservices)
- [Vertical Slice Architecture](https://www.jimmybogard.com/vertical-slice-architecture/)
- [Service-oriented architecture](https://en.wikipedia.org/wiki/Service-oriented_architecture)
- [RFC 9457 - Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)
- [Replacing Exceptions-as-flow-control with the result pattern](https://andrewlock.net/working-with-the-result-pattern-part-1-replacing-exceptions-as-control-flow/)
- [Railway oriented programming](https://fsharpforfunandprofit.com/posts/recipe-part2/)
- [Error handling and Go](https://go.dev/blog/error-handling-and-go)
- [Functional Event Sourcing Decider](https://thinkbeforecoding.com/post/2021/12/17/functional-event-sourcing-decider)
- [Testing business logic in Event Sourcing, and beyond!](https://event-driven.io/en/testing_event_sourcing/)
- [Testing Without Mocks: A Pattern Language](https://www.jamesshore.com/v2/projects/nullables/testing-without-mocks)
- [A-Frame Architecture with Wolverine](https://jeremydmiller.com/2023/07/19/a-frame-architecture-with-wolverine/)
- ["Parse, don't validate" and other type safety driven shenanigans (plus a C# wishlist)](https://blog.codingmilitia.com/2023/08/17/parse-dont-validate-and-other-type-safety-driven-shenanigans-plus-a-csharp-wishlist/)
- [FluentValidation](https://docs.fluentvalidation.net)
- [Contract first OpenAPI development (but still use Swagger UI with ASP.NET Core)](https://blog.codingmilitia.com/2023/04/02/contract-first-openapi-development-but-still-use-swagger-ui-with-asp.net-core/)
- [Scalar .NET Integration](https://guides.scalar.com/scalar/scalar-api-references/net-integration)
- [Testcontainers](https://testcontainers.com)
- [Apache Avro](https://avro.apache.org)
- [Transactional Outbox Pattern - OutboxKit](https://outboxkit.yakshavefx.dev/intro/transactional-outbox-pattern.html)
- [OpenTelemetry](https://opentelemetry.io)
- [Docker Compose docs](https://docs.docker.com/compose/)
- [Microservice chassis pattern](https://microservices.io/patterns/microservice-chassis.html)

Thanks for stopping by, cyaz! 👋