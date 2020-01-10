---
author: João Antunes
date: 2020-01-10 18:25:00+00:00
layout: post
title: "E034 - Segregating use cases with MediatR - ASPF02O"
summary: "In this episode, we'll take a look at segregating our application use cases in dedicated classes instead of shoving everything into the same service class."
image: '/assets/2020/01/10/aspnet-core-from-zero-to-overkill-e034.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- mediatr
---
In this episode, we'll take a look at segregating our application use cases in dedicated classes instead of shoving everything into the same service class.

**Note:** depending on your preference, you can check out the following video or skip it to the written version below.

{% youtube "https://youtu.be/yJ5aHTTVm34" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

In the [previous episode](https://blog.codingmilitia.com/2019/12/20/aspnet-033-from-zero-to-overkill-redesigning-the-api-improving-the-internal-architecture), we took a look at the internal architecture redesign of the group management API, to get away from the traditional three layer architecture, in favor of a more interesting onion/hexagonal/ports and adapters/clean architecture.

Even though not inherent of any of these project organization approaches, one of my biggest gripes with traditional logic organization is the reliance on service classes to put every bit of business logic in. In more complex applications. these service classes end up being bloated messes, resulting in the [big ball of mud](https://en.wikipedia.org/wiki/Big_ball_of_mud) we (rightfully) fear.

## Problem statement

As introduced, one of my biggest annoyances with traditional project logic organization is the reliance on services where we shove all the logic. Let's take the group management example, namely the interactions spawned from the `GroupsController`, to see how it would end up following this traditional approach.

- `GroupsController` - Very simple logic here, focused on the interaction with the HTTP requests/responses. One action method for each operation (use case) we expose.
- `GroupsService` - Everything related to groups domain logic would end up here, traditionally with one method for each operation we expose (mapping to the action methods on the controller).
- Entities (for data access) - DTO like bags of data, with properties exposing public getters and setters.

For these three points, the first one, I'm ok with. When going with a more complex project organization like this, we'd like to keep the controllers devoid of domain logic, focusing on the HTTP interactions.

The later two points however, are more of a problem. In this episode we'll focus on the service part, leaving the entities for a posterior episode.

This abuse of services that basically map 1-to-1 with the controllers, not only seems like an unnecessary abstraction when used like this (if we're going to put everything in the same place, why not just put it in the controller?), but worst than that, completely destroys the [SOLID principles](https://en.wikipedia.org/wiki/SOLID) everyone pretends to love but don't actually implement.

Let's take a look at the three principles I feel are more affected by this:

- (S)ingle responsibility principle - This principle normally translates to a component should have a single reason to change. If we have a service with all the logic that's related to groups and probably other related concepts like its players, users, etc, will it really have a single reason to change? Probably not.
- (O)pen-closed principle - Regarding being open for extension, closed for modification, considering a service, as described previously, exposes a bunch of operations, I'd say we'll probably be modifying it pretty often as we do adjustments/add features/ fix bugs.
- (I)nterface segregation principle - Following the typical .NET developer approach, we extract an interface out of our service. We look at it, and what do we see? Maybe some 10 or 20 methods? Will the calling code always need all of that? Going about it this way, when looking at code that depends on such service, it's hard do understand on what parts of the service the code actually depends on. We .NET developers love dependency injection, and say that we like doing constructor injection, because it makes the dependencies clear, but if a dependency has such a broad API surface area, it doesn't feel as clear as we're selling.

Before going further let me be clear, we shouldn't follow these principles as super hard rules, it wouldn't make much sense - maybe a silly example but, a `List<T>` has `Add` and `Remove` methods, it wouldn't make sense to split them as they're part of the core list logic - but we should try to understand their ideas and apply where it makes sense. In this "super service" case, I think we can do better to apply them.

Pretty regularly these kinds of services, particularly taking into consideration that when using them we don't extract logic to other classes (i.e. entities, value objects, auxiliary services...), grow to the large hundreds, eventually thousands of lines. I don't know about you, but I have a terrible time when trying to navigate such files.

## Use case segregation

So, I've ranted enough about what I don't like about the common approach of "super service + DTO all the things", so let's look at a possible alternative.

Instead of having a service with all the operations, we can create a class to handle each, which ends up playing nicely with a [CQRS](https://www.martinfowler.com/bliki/CQRS.html) approach.

Also in line with the CQRS approach, even though not mandatory to split things up, we can create models specific to each use case, avoiding the reuse of classes with data that is not need all the time. A good example we had already in the group management API is the reuse of the `GroupModel` for every operation. When creating a group, we disregard the id and the row version, as in our case they're handled by the database upon insertion, but looking at the API contract seriously, the client may put stuff in there. Creating specific models avoids this kinds of situation, allowing us to tailor things for each case.

Going with such an approach, we're on track to follow those principles we talked about:

- (S)ingle responsibility principle - As a use case typically maps to a feature we provide (create a group, update a group, list all groups, ...) the reason to change a specific class boils down to make adjustments to a specific feature. When we want to add a new feature, normally we'll create a new use case, the existing ones should be unchanged.
- (O)pen-closed principle - This principle will always be tough, as requirements change and we eventually need to adjust things, but with the reduced need to change a specific use case handler due to logic segregation, it's much more closed for modification than before. Also, regarding extension, if we implement the use case handling in a somewhat standardized way (as we are by using MediatR, even though it's not the only way to do it), it's pretty easy to sit extra features, particularly cross-cutting concerns, on top of our use case implementations without directly touching the domain related code.
- (I)nterface segregation principle - As we move from a big interface with a bunch of methods, to an interface designed to handle a single use case, the API surface area reduces drastically and it's much easier to understand the actual dependencies of our code.

Other loose reasons for me being a fan of this approach:

- As we'll see in a bit, the use case implementation will be a class with a single public method. As it is a good practice, we can create other private methods to make the code of our main public method more readable. When we do this in a bigger service approach, we end up putting all the private methods towards the end of the file, making it a bit harder to grasp by which exposed operation it is used, and even if it's used by one or more. With this segregation, it's pretty clear how the private methods are used. *Side note*: in cases where it's harder to separate things, a nice C# feature we can use are the [local functions](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/local-functions),
- Smaller class, smaller file, easier to read, harder to cause merge conflicts 🙂.
- When a service does too much, it often ends up with more dependencies, making it harder to understand which parts of the service actually depend on what. This segregation makes dependencies clearer per use case.

## Implementation with MediatR

### Quick intro to MediatR

Now that I yapped long enough about my dislike for the more traditional approach, and how I think use case segregation can help, let's look at a possible implementation, using [MediatR](https://github.com/jbogard/MediatR), an open source project created by [Jimmy Bogard](https://twitter.com/jbogard).

MediatR provides us with a generic implementation of the [mediator pattern](https://en.wikipedia.org/wiki/Mediator_pattern), allowing us to invoke a specific use case handler without being aware of it's actual implementation, by way of in-memory message passing. We could implement something similar, or even simpler targeting our needs, without too much difficulty, but that's the point of using libraries like these, make us write less code, and in this case we take advantage of something very well tested in production already.

MediatR supports two kinds of message dispatching:

- Request/response messages, dispatched to a single handler
- Notification messages, dispatched to multiple handlers

For now we'll only use request/response, as we're replacing the `GroupsService` with MediatR, mapping each `GroupsController` action method to a MediatR request message.

Another interesting concept in MediatR (that we won't be using now, but will surely in the future) are the [behaviors](https://github.com/jbogard/MediatR/wiki/Behaviors), which allow us to build a request handling pipeline withing MediatR. As an example, imagine we want to measure the time all the requests take, or maybe we want to cache our query results. Instead of mixing that code with the domain logic inside the request handler itself, we add behaviors to the pipeline, keeping the domain logic focused. It's a similar concept to [middlewares in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-3.1), but at an application logic level, instead of mixed in with the HTTP request pipeline.

### Implementing the use cases

Let's finally look at the implementation. If you recall from the [last episode](https://blog.codingmilitia.com/2019/12/20/aspnet-033-from-zero-to-overkill-redesigning-the-api-improving-the-internal-architecture), we have a reorganized project structure, and these use cases will go into the domain project (`CodingMilitia.PlayBall.GroupManagement.Domain`). In there, the current organization is the following:

```
.
├── Data
|   └── ...
├── Entities
|   └── ...
├── Mappings
|   └── ...
└── UseCases
    ├── CreateGroup
    |   ├── CreateGroupCommand.cs
    |   ├── CreateGroupCommandHandler.cs
    |   └── CreateGroupCommandResult.cs
    ├── DeleteGroup
    |   ├── DeleteGroupCommand.cs
    |   └── DeleteGroupCommandHandler.cs
    ├── GetUserGroupDetail
    |   ├── GetUserGroupQuery.cs (just noticing I messed up naming 😅)
    |   ├── GetUserGroupQueryHandler.cs
    |   └── GetUserGroupQueryResult.cs
    ├── GetUserGroups
    |   └── ...
    └── UpdateGroupDetails
        └── ...
```

As you can see, for each use case we have at least a class representing the request (e.g. `CreateGroupCommand` or `GetUserGroupQuery`) and another one to handle it (e.g. `CreateGroupCommandHandler` or `GetUserGroupQueryHandler`). Then some might also have a result (e.g. `CreateGroupCommandResult` or `GetUserGroupQueryResult`), while others might not have it (e.g. the delete group use case doesn't really need a result right now).

Regarding the project structure, it could be simplified, for instance, no need to split in as many files as I have. For a simplified example take a look at Jimmy Bogard's [ContosoUniversity project](https://github.com/jbogard/ContosoUniversityDotNetCore/tree/master/ContosoUniversity/Features).

Let's look at the create group use case for some sample code.

`CreateGroupCommand.cs`
```csharp
public sealed class CreateGroupCommand : IRequest<CreateGroupCommandResult>
{
    public CreateGroupCommand(
        string userId,
        string name)
    {
        UserId = userId;
        Name = name;
    }

    public string UserId { get; }
    public string Name { get; }
}
```

`CreateGroupCommandResult.cs`
```csharp
public sealed class CreateGroupCommandResult
{
    public CreateGroupCommandResult(
        long id,
        string name,
        string rowVersion,
        User creator)
    {
        Id = id;
        Name = name;
        RowVersion = rowVersion;
        Creator = creator;
    }
    
    public long Id { get; }
    public string Name { get; }
    public string RowVersion { get; }
    public User Creator { get; }

    public class User
    {
        public User(
            string id,
            string name)
        {
            Id = id;
            Name = name;
        }

        public string Id { get; }
        public string Name { get; }
    }
}
```

`CreateGroupCommandHandler.cs`
```csharp
public sealed class CreateGroupCommandHandler 
: IRequestHandler<CreateGroupCommand, CreateGroupCommandResult>
{
    private readonly IRepository<Group> _groupsRepository;
    private readonly IQueryHandler<UserByIdQuery, User> _userByIdQueryHandler;

    public CreateGroupCommandHandler(
        IRepository<Group> groupsRepository,
        IQueryHandler<UserByIdQuery, User> userByIdQueryHandler)
    {
        _groupsRepository =
            groupsRepository
            ?? throw new ArgumentNullException(nameof(groupsRepository));

        _userByIdQueryHandler =
            userByIdQueryHandler
            ?? throw new ArgumentNullException(nameof(userByIdQueryHandler));
    }

    public async Task<CreateGroupCommandResult> Handle(
        CreateGroupCommand request,
        CancellationToken cancellationToken)
    {
        var currentUser = await _userByIdQueryHandler.HandleAsync(
            new UserByIdQuery(request.UserId),
            cancellationToken);

        var group = new Group
        {
            Name = request.Name,
            Creator = currentUser
        };

        group.GroupUsers.Add(
            new GroupUser {
                User = currentUser,
                Role = GroupUserRole.Admin
            });

        var addedGroup = await _groupsRepository.AddAsync(
            group,
            cancellationToken);

        return new CreateGroupCommandResult(
            addedGroup.Id,
            addedGroup.Name,
            addedGroup.RowVersion.ToString(),
            new CreateGroupCommandResult.User(
                currentUser.Id,
                currentUser.Name));
    }
}
```

As we can see above, `CreateGroupCommand` and `CreateGroupCommandResult` are just DTOs, in the case of the former, containing only the required information to create a group (no more shared `GroupModel`), and in the case of the latter, containing the information resulting from the group creation. `CreateGroupCommand` implements `IRequest<CreateGroupCommandResult>`, which is basically a marker interface, just to simplify glueing together the three use case related classes.

The handler (`CreateGroupCommandHandler`) implements `IRequestHandler<CreateGroupCommand, CreateGroupCommandResult>`, indicating the type of parameter it'll get, as well as the output type.

In the constructor we get the dependencies that the handler requires. This might not look like much right now, but this is great for more complex scenarios, as we discussed previously, as we can clearly see the dependencies of a use case, instead of a bunch of dependencies in a service that would handle multiple use cases. The dependencies are a group repository, so we can add the newly created group, as well as something I called query handler, to fetch information about the creator of the group. Think about this query handler as a specialized repository, with a single method. We'll look at why I went with this (probably overkill approach) in the next episode.

Finally, the `Handle` method, which is defined by the `IRequestHandler<TRequest, TResponse>` interface, is where we do the actual implementation. In this example, as the use case is pretty simple, the implementation is also simple: setup the group using the inbound information, add it to the database and return the relevant information using the result DTO.

### Tying it all together

Now that we've seen an example implementation of a use case using MediatR, we need to make it work with our ASP.NET Core Web API. The domain project will have a dependency on the MediatR package. The Web project will require an additional dependency, to make MediatR work seamlessly with ASP.NET Core dependency injection, `MediatR.Extensions.Microsoft.DependencyInjection`.

Now we head to where we're configuring our dependency injection container, in this case `IoC\ServiceCollectionExtensions.cs` and add MediatR in there. Thankfully it's really easy, because the package reference we added comes with helper methods to aid us.

`IoC\ServiceCollectionExtensions.cs`
```csharp
// ...
public static IServiceCollection AddBusiness(this IServiceCollection services)
{
    services.AdMediatR(typeof(GetUserGroupsQuery));

    return services;
}
// ...
```

Using the `AddMediatR` extension method, we can simply provide the assembly that contains our MediatR requests and handlers, and it'll scan it and add everything.

Now in the `GroupsController`, we just need to send the requests to MediatR, instead of calling the service as before.

`Features\Groups\GroupsController.cs`
```csharp
[Route("groups")]
public class GroupsController : ControllerBase
{
    // ...

    public GroupsController(
        IMediator mediator,
        ICurrentUserAccessor currentUserAccessor)
    {
        _mediator = mediator;
        _currentUserAccessor = currentUserAccessor;
    }

    // ...

    [HttpPut]
    [HttpPost]
    [Route("")]
    public async Task<ActionResult<CreateGroupCommandResult>> AddAsync(
        CreateGroupCommandModel model,
        CancellationToken ct)
    {
        var result = await _mediator.Send(
            new CreateGroupCommand(
                _currentUserAccessor.Id,
                model.Name));

        return CreatedAtAction(
            GetByIdActionName,
            new {id = result.Id},
            result);
    }

    // ...
}
```

Simple enough. We depend on `IMediator`, which we'll use to forward our requests to the handlers, and on `ICurrentUserAccessor`, just to grab the current user id (which is present in JWT we get in each API request) and add it as part of the create group command DTO. The action method is pretty basic, mapping all the required data to the `CreateGroupCommand` DTO, passing it in to MediatR and then grabbing the result to create an HTTP response out of it.

## Outro

That does it for the segregation of the `GroupService` into individual use case handlers. As I mentioned, the thing I like the most about this approach is getting rid of "super classes" that do too much, get messy and terrible to maintain. If you're just doing CRUD, this is probably too much, but if you have a fair amount of domain logic, I think it's really worth the separation.

We used MediatR for this implementation, as it already does for us a bunch of things we would end up doing manually, but if you don't want to add it as a dependency, you can start by simply creating something like an `IRequestHandler<TInput, TOutput>`, create handlers implementing it and register them in the DI container. I took a similar approach in a [recent presentation](https://github.com/joaofbantunes/BackToBasicsTheMessWereMakingOutOfOOP) I did (keep in mind it's even more demo code than what I do in this series, so it's not completely implemented).

Links in the post:

- [Big ball of mud](https://en.wikipedia.org/wiki/Big_ball_of_mud)
- [SOLID](https://en.wikipedia.org/wiki/SOLID)
- [CQRS](https://www.martinfowler.com/bliki/CQRS.html)
- [Local functions](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/local-functions)
- [MediatR](https://github.com/jbogard/MediatR)
- [Mediator pattern](https://en.wikipedia.org/wiki/Mediator_pattern)
- [ASP.NET Core Middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-3.1)
- [ContosoUniversity on ASP.NET Core with .NET Core](https://github.com/jbogard/ContosoUniversityDotNetCore/)
- [E033 - Redesigning the API: Improving the internal architecture - ASPF02O](https://blog.codingmilitia.com/2019/12/20/aspnet-033-from-zero-to-overkill-redesigning-the-api-improving-the-internal-architecture)

The source code for this post is in the [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode033) repository, tagged as `episode033` (didn't really change anything for this post).

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!