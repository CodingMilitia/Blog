---
author: João Antunes
date: 2020-04-20 18:00:00+01:00
layout: post
title: "Event-driven integration #2 - Inferring events from EF Core changes [ASPF02O|E041]"
summary: "In this first step implementing event-driven integration between services, we'll hook-up into EF Core's infrastructure, namely when saving changes, to infer if any event should be raised based on the information provided by the change tracker."
images:
- '/images/2020/04/20/aspnet-core-from-zero-to-overkill-e041.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- efcore
slug: aspnet-041-from-zero-to-overkill-event-driven-integration-inferring-events-from-efcore-changes
---

In this first step implementing event-driven integration between services, we'll hook-up into EF Core's infrastructure, namely when saving changes, to infer if any event should be raised based on the information provided by the change tracker.

**Note:** depending on your preference, you can check out the following video, otherwise, skip to the written version below.

{{< yt rJDZiFJGP90 >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

As introduced in the previous episode, we're going to implement the transactional outbox pattern, to make event delivery reliable. In this first step, we're going to look at where we can put the event creation and storage logic.

As it's made obvious from the post title, we're going to hook-up into EF Core's infrastructure to implement this, but before, we'll take a look at a couple of alternative options.

## Where to put the event creation and storage logic

Let's begin with a small task: where do we put the event creation and storage logic?

Normally I'd say together with the specific domain logic. Thinking of the register user case, this means either in the `PlayBallUser` entity itself or in `UserManager.CreateAsync`. Problem is, as this is ASP.NET Core Identity code, we don't have full control to put this extra logic in there (though we could probably do something about it, considering `UserManager` has its public members marked as `virtual`).

So, as this approach is not easily doable, how about some other options?

As we discussed in the previous episode, what we require is that things are done in a single transaction, so we could to go `Register.cshtml.cs` and do something like (pseudo-code):

`Pages\Register.cshtml.cs`

```csharp
BeginTransaction();
var result = await _userManager.CreateAsync(user, Input.Password);
if(result.Succeeded)
{
    AddEvent(new UserRegisteredEvent());
    CommitTransaction();
}
else
{
    RollbackTransaction();
}
```

This would do the trick, but it doesn't feel like the right place to put this kind of code, feels like it's mixing things up, considering we're in the page, which should be more worried with invoking domain logic and mapping that to the UI, not dealing with transactions.

The last option we'll look at, and the one we'll implement, is to take advantage of EF Core's extensibility points, namely by overriding `SaveChanges` and using the change tracker to get information about what happened.

## Overriding EF Core SaveChanges

Overriding EF Core `SaveChanges` is a common strategy, as it allows centralization of certain types of logic that would be a pain to have spread everywhere.

Some common use-cases are related to events, like is our case, but not necessarily inferring them as we'll do. As an example from [Steve Smith's Clean Architecture template](https://github.com/ardalis/CleanArchitecture/blob/master/src/CleanArchitecture.Infrastructure/Data/AppDbContext.cs), he overrides `SaveChanges` to get the domain events added to the entities, then dispatches them.

Another common use-case is to handle properties we'd like to be filled in automatically, like entity modification date and author. [The linked post implements something like this.](https://www.c-sharpcorner.com/UploadFile/d87001/overriding-savechanges-in-entity-framework/)

The gist of the approach is rather simple: in `AuthDbContext`, override `SaveChanges` and add the extra code.

`Data\AuthDbContext.cs`

```csharp
public class AuthDbContext : IdentityDbContext<PlayBallUser>
{
    // ...

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = new CancellationToken())
    {
        // TODO: add event detection logic here

        var result = await base.SaveChangesAsync(cancellationToken);

        return result;
    }
}
```

Note: in this implementation we're overriding one of `SaveChangesAsync` overloads only, and it's enough because it's the commonly used one, but to be on the safe side, we should override all overloads of `SaveChanges` and `SaveChangesAsync`.

## Inferring events from the change tracker

The `DbContext` class exposes a property `ChangeTracker`, where we can access all the entities that have been changed in some way, requiring them to be persisted. We'll take advantage of this to find all changes to entities of type `PlayBallUser`, then map them to events (in the next post).

As a quick example, if we want to get all created users - normally it will be a single one, but we can make the code generic - we can do something like the following:

```csharp
db.ChangeTracker.Entries<PlayBallUser>().Where(u => u.State == EntityState.Added);
```

To avoid putting too much responsibilities into our `AuthDbContext`, instead of having it all directly in the `SaveChangesAsync` implementation, we can extract things. With this in mind we can create an `IEventDetector` interface, which can be implemented to detect different kinds of events.

`Data\IEventDetector.cs`

```csharp
public interface IEventDetector
{
    void Detect(AuthDbContext db);
}
```

Now in the `AuthDbContext`, we get instances of `IEventDetector` provided through the constructor and invoke them in `SaveChangesAsync`.

`Data\AuthDbContext.cs`

```csharp
public class AuthDbContext : IdentityDbContext<PlayBallUser>
{
    private readonly IEnumerable<IEventDetector> _eventDetectors;

    public AuthDbContext(DbContextOptions<AuthDbContext> options, IEnumerable<IEventDetector> eventDetectors)
        : base(options)
    {
        _eventDetectors = eventDetectors;
    }

    // ...

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = new CancellationToken())
    {
        foreach (var detector in _eventDetectors)
        {
            detector.Detect(this);
        }

        // TODO: in the next post we'll go through storing the events in the outbox table

        var result = await base.SaveChangesAsync(cancellationToken);

        return result;
    }
}
```

With this we avoid bloating `AuthDbContext`, keeping all event detection segregated. As we have three kinds of events (user registered, updated and deleted), we create three implementations of `IEventDetector`.

User registered and deleted are the most straightforward, as we need only to check if the entity state is `Added` or `Deleted`.

`Infrastructure\Data\EventDetectors\UserRegisteredEventDetector.cs`

```csharp
public class UserRegisteredEventDetector : IEventDetector
{
    private readonly ILogger<UserRegisteredEventDetector> _logger;

    public UserRegisteredEventDetector(ILogger<UserRegisteredEventDetector> logger)
    {
        _logger = logger;
    }

    public void Detect(AuthDbContext db)
    {
        var userRegisteredChanges =
            db
                .ChangeTracker
                .Entries<PlayBallUser>()
                .Where(u => u.State == EntityState.Added)
                .ToList();

        foreach (var change in userRegisteredChanges)
        {
            _logger.LogInformation("UserRegisteredEvent - {username}", change.Entity.UserName);
        }
    }
}
```

`Infrastructure\Data\EventDetectors\UserDeletedEventDetector.cs`

```csharp
public class UserDeletedEventDetector : IEventDetector
{
    private readonly ILogger<UserDeletedEventDetector> _logger;

    public UserDeletedEventDetector(ILogger<UserDeletedEventDetector> logger)
    {
        _logger = logger;
    }

    public void Detect(AuthDbContext db)
    {
        var userDeletedChanges =
            db
                .ChangeTracker
                .Entries<PlayBallUser>()
                .Where(u => u.State == EntityState.Deleted)
                .ToList();

        foreach (var change in userDeletedChanges)
        {
            _logger.LogInformation("UserDeletedEvent - {username}", change.Entity.UserName);
        }
    }
}
```

Detecting user updated events is slightly more complicated, just because we don't necessarily want to create an event for every type of change. For instance, it's not really relevant for the other services to get an event when the password changes, or two-factor is enabled, as these are more internal responsibilities of the auth service, not really something other services care about (depends on your use case of course).

With this in mind, for user updated event detection, besides checking that the entity state is `Modified`, we also check if any of the properties that are relevant for other services have changed. In this case let's assume the only thing other services care about is the `UserName` property. To do this check we can use the `EntityEntry<PlayBallUser` class' `OriginalValues` and `CurrentValues` properties.

`Infrastructure\Data\EventDetectors\UserUpdatedEventDetector.cs`

```csharp
public class UserUpdatedEventDetector : IEventDetector
{
    private readonly ILogger<UserUpdatedEventDetector> _logger;

    public UserUpdatedEventDetector(ILogger<UserUpdatedEventDetector> logger)
    {
        _logger = logger;
    }

    public void Detect(AuthDbContext db)
    {
        const string UserNameProperty = nameof(PlayBallUser.UserName);

        var userUpdatedChanges =
            db
                .ChangeTracker
                .Entries<PlayBallUser>()
                .Where(u => u.State == EntityState.Modified
                            &&
                            u.OriginalValues.GetValue<string>(UserNameProperty) !=
                            u.CurrentValues.GetValue<string>(UserNameProperty))
                .ToList();

        foreach (var change in userUpdatedChanges)
        {
            _logger.LogInformation("UserUpdatedEvent - {username}", change.Entity.UserName);
        }
    }
}
```

And with this we have things in place to detect the three types of events we'll be publishing.

Just as a side note, to get the `IEventDetector`s configured in DI in order to be injected into the `AuthDbContext`, I'm using [Scrutor](https://github.com/khellang/Scrutor):

`IoC\EventExtensions.cs`

```csharp
public static class EventExtensions
{
    public static IServiceCollection AddEvents(this IServiceCollection services)
        => services.Scan(
            scan => scan
                .FromAssemblyOf<UserRegisteredEventDetector>()
                .AddClasses(classes => classes.AssignableTo(typeof(IEventDetector)))
                .AsImplementedInterfaces()
                .WithSingletonLifetime()
        );
}
```

## Outro

That does it for this episode. We took a quick look at hooking into EF Core's infrastructure, overriding `SaveChanges` and inferring events from the change tracker.

Main takeaways are:

- overriding `SaveChanges` is a common strategy for centralizing code that acts on entities just before/after persistence changes
- we have access to Entity Framework's change tracker, being able to see what changes were applied to our entities

In the next episode we'll build upon these event detectors, mapping the detected changes to actual events we'll store in the outbox table.

As a quick PSA before closing, just to remind that many of the problems we're looking into in these event-driven topic (not this episode in particular) can solved by existing libraries, so it might be interesting to look into them before doing everything manually.
We're doing things manually in the series to make the problems surface so everyone's aware o them, not just assume everything works magically.
As a couple of examples of such libraries for .NET projects, we have [MassTransit](https://masstransit-project.com/) and [NServiceBus](https://particular.net/nservicebus).

Links in the post:

- [Steve Smith's Clean Architecture template](https://github.com/ardalis/CleanArchitecture)
- [Overriding SaveChanges in Entity Framework](https://www.c-sharpcorner.com/UploadFile/d87001/overriding-savechanges-in-entity-framework/)
- [Scrutor](https://github.com/khellang/Scrutor)

The source code for this post is in the [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode041) repository, tagged as `episode041`.

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!