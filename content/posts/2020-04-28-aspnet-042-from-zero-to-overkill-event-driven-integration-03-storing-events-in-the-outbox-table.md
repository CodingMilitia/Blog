---
author: João Antunes
date: 2020-04-28 17:30:00+01:00
layout: post
title: "Event-driven integration #3 - Storing events in the outbox table [ASPF02O|E042]"
summary: "On the footsteps of the last episode, in this one we store the inferred events in the outbox table, doing so transactionally, so we have guarantees that any change will eventually result in a published event."
images:
- '/assets/2020/04/28/aspnet-core-from-zero-to-overkill-e042.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- efcore
slug: aspnet-042-from-zero-to-overkill-event-driven-integration-03-storing-events-in-the-outbox-table
---

On the footsteps of the last episode, in this one we store the inferred events in the outbox table, doing so transactionally, so we have guarantees that any change will eventually result in a published event.

**Note:** depending on your preference, you can check out the following video, otherwise, skip to the written version below.

{{< yt G187r-rzzkk >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

In the [previous episode](https://blog.codingmilitia.com/2020/04/20/aspnet-041-from-zero-to-overkill-event-driven-integration-inferring-events-from-efcore-changes/), we implemented the detection of events based on EF Core changes. In this episode, we'll take a look at the event storage part of the transactional outbox pattern introduced in [episode 40](https://blog.codingmilitia.com/2020/04/13/aspnet-040-from-zero-to-overkill-event-driven-integration-transactional-outbox-pattern/).

To situate ourselves, we can take a look at the diagram introduced in episode 40:

[![situating ourselves](/assets/2020/04/28/e042-outbox-pattern-storage.png)](/assets/2020/04/28/e042-outbox-pattern-storage.png)

## The outbox table

The first thing we need is to create the outbox table, where we can store the events to later publish. We'll keep it simple with three columns, even though one of them is "special": an identifier, a `DateTime` when the event was created and a JSON column where we store the complete information about an event.

We'll use a JSON column for the event data, because it doesn't really make much sense to model with a bunch of columns all the possible properties the events will have, as it can be rather dynamic and doesn't provide any value. PostgreSQL [has great support for JSON columns](https://www.postgresql.org/docs/12/datatype-json.html), being even used as a [document database with some abstractions on top of it](https://github.com/JasperFx/Marten).

Starting with the class that'll map to the table, we have `OutboxMessage`:

`Data\OutboxMessage.cs`

```csharp
public class OutboxMessage
{
    public OutboxMessage(DateTime createdAt, BaseAuthEvent @event)
    {
        CreatedAt = createdAt;
        Event = @event;
    }

    public long Id { get; private set; }

    public DateTime CreatedAt { get; private set; }

    public BaseAuthEvent Event { get; private set; }
}
```

The `Event` property will be the one mapped as JSON. To keep it simple, we have a class named `BaseAuthEvent` (we'll look at it in a minute) that'll be the base for all types of events published by the auth service.

As for configuring the mapping between the class and the database, we implement `IEntityTypeConfiguration<OutboxMessage>` as usual:

`Infrastructure\Data\Configurations\OutboxMessageConfiguration.cs`

```csharp
public class OutboxMessageConfiguration : IEntityTypeConfiguration<OutboxMessage>
{
    public void Configure(EntityTypeBuilder<OutboxMessage> builder)
    {
        var settings = new JsonSerializerSettings
        {
            TypeNameHandling = TypeNameHandling.Objects
        };

        builder
            .HasKey(e => e.Id);

        builder
            .Property(e => e.Id)
            .UseIdentityAlwaysColumn();

        builder
            .Property(e => e.Event)
            .HasColumnType("json")
            .HasConversion(
                e => JsonConvert.SerializeObject(e, settings),
                e => JsonConvert.DeserializeObject<BaseAuthEvent>(e, settings));
    }
}
```

The `Id` property configuration is the same we already saw in past episodes. Now the interesting part is the `Event` property.

We start by setting the column type as `json`. If we didn't have to deal with inheritance, which we will because we'll be inheriting from `BaseAuthEvent`, that would be all that we need to do, as the PostgreSQL provider for EF Core [(Npgsql)](https://github.com/npgsql/Npgsql) supports serializing things to JSON out of the box, using [System.Text.Json](https://docs.microsoft.com/en-us/dotnet/standard/serialization/system-text-json-overview).

As, unfortunately, System.Text.Json doesn't handle inheritance without custom code (at the time of writing), we can either implement that custom code or use something else. To keep it simple, we'll use [Json.NET](https://github.com/JamesNK/Newtonsoft.Json), which is capable of handling inheritance fine, provided we do the necessary configurations, which we can see in the code, by setting the `TypeNameHandling` property. What we're saying with this setting, is that when serializing objects, Json.NET should include the type name, so it's capable of figuring out what's the actual type it needs to deserialize data into. With the call to `HasConversion`, we setup the `Event` property serialization to be handled by Json.NET with the desired settings.

An example stored event:

```json
{
    "$type": "CodingMilitia.PlayBall.Auth.Web.Data.Events.UserUpdatedEvent, CodingMilitia.PlayBall.Auth.Web",
    "UserId": "4138e44c-da10-4108-b3ab-4901eb27da5f",
    "UserName": "test2@test.com",
    "Id": "43ae30cf-105b-4354-aa61-8f7f765e81fc",
    "OccurredAt": "2020-04-25T15:25:37.5250567Z"
}
```

As a final note about this configuration, I had initially used the column type as `jsonb` not `json`, but ended up changing. The main difference is that `json` stores things in text format while `jsonb` stores things in a optimized binary format. [From the docs](https://www.postgresql.org/docs/12/datatype-json.html):

> The json and jsonb data types accept almost identical sets of values as input. The major practical difference is one of efficiency. The json data type stores an exact copy of the input text, which processing functions must reparse on each execution; while jsonb data is stored in a decomposed binary format that makes it slightly slower to input due to added conversion overhead, but significantly faster to process, since no reparsing is needed. jsonb also supports indexing, which can be a significant advantage.

As mentioned, the main advantage of `jsonb` is its usage in queries, which we won't really take advantage of, but that's not the main reason I ended up changing types (even though it's a good reason). The main reason is when storing the column data, `jsonb` might reorder the object properties. Normally this is no problem, but because Json.NET needs to find the `$type` property (seen in the example above) as the first property to know the type to deserialize things into, and PostgreSQL was moving the `Id` property to the first spot, things didn't work 🙂.

The final thing we need is to add the `OutboxMessages` property (a `DbSet<OutboxMessage>`) to the `AuthDbContext` class and add the migration.

```sh
dotnet ef migrations add CreateOutboxMessagesTable -c AuthDbContext
```

## Modeling the events

Now to model the events. In previous episodes we defined that, for now, we'll have three types: user registered, user updated and user deleted. As mentioned in the previous section, we'll have a base class for the events, named `BaseAuthEvent`.

`Data\Events\BaseAuthEvent.cs`

```csharp
public abstract class BaseAuthEvent
{
    public Guid Id { get; set; }

    public DateTime OccurredAt { get; set; } = DateTime.UtcNow;
}
```

Keeping things simple, we have an identifier and the `DateTime` in which the event occurred. In the future, we might need to add something more, but for now, it'll do.

As for the actual events, not much to them as well, as we don't have a ton of data to include.

`Data\Events\UserRegisteredEvent.cs`

```csharp
public class UserRegisteredEvent : BaseAuthEvent
{
    public string UserId { get; set; }

    public string UserName { get; set; }
}
```

`Data\Events\UserUpdatedEvent.cs`

```csharp
public class UserUpdatedEvent : BaseAuthEvent
{
    public string UserId { get; set; }

    public string UserName { get; set; }
}
```

`Data\Events\UserDeletedEvent.cs`

```csharp
public class UserDeletedEvent : BaseAuthEvent
{
    public string UserId { get; set; }
}
```

Both `UserRegisteredEvent` and `UserUpdatedEvent` include the id of the user affected, as well as its username, as it's the only property that we care to inform the listening services about. `UserDeletedEvent` only needs the user id.

## Mapping changes to events

With things in place at the database level, we need to implement the logic bits, starting with the mapping of the events.

In the last episode, we created the concept of event detectors, to which we provided the `DbContext` and they would check the change tracker for event worthy changes. We'll build upon that concept to map the events that are detected.

The `IEventDetector` interface becomes `IEventMapper`, with slight tweaks to the exposed method.

`Data\IEventMapper.cs`

```csharp
public interface IEventMapper
{
    IEnumerable<OutboxMessage> Map(AuthDbContext db, DateTime occurredAt);
}
```

Instead of just detecting events, it now returns a collection of events detected, already mapped as an `OutboxMessage` instance. Just for optimization's sake, it also gets as a parameter the occurrence date/time.

In the `AuthDbContext` class, we can replace the event detection with the mapping.

`Data\AuthDbContext.cs`

```csharp
public class AuthDbContext : IdentityDbContext<PlayBallUser>
{
    private readonly IEnumerable<IEventMapper> _eventMappers;

    public AuthDbContext(DbContextOptions<AuthDbContext> options, IEnumerable<IEventMapper> eventMappers)
        : base(options)
    {
        _eventMappers = eventMappers;
    }

    // ...

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = new CancellationToken())
    {
        var eventsDetected = GetEvents();

        // ...
    }

    private IReadOnlyCollection<OutboxMessage> GetEvents()
    {
        var now = DateTime.UtcNow;

        return _eventMappers
            .SelectMany(mapper => mapper.Map(this, now))
            .ToList();
    }
}
```

As for the event mapper implementations, they're similar to the detectors we saw in the past episode, with the extra mapping logic:

`Infrastructure\Data\UserRegisteredEventMapper.cs`

```csharp
public class UserRegisteredEventMapper : IEventMapper
{
    public IEnumerable<OutboxMessage> Map(AuthDbContext db, DateTime occurredAt)
        => db
            .ChangeTracker
            .Entries<PlayBallUser>()
            .Where(entry => entry.State == EntityState.Added)
            .Select(entry =>
                new OutboxMessage(occurredAt,
                    new UserRegisteredEvent
                    {
                        Id = Guid.NewGuid(),
                        OccurredAt = occurredAt,
                        UserId = entry.Entity.Id,
                        UserName = entry.Entity.UserName
                    }));
}
```

`Infrastructure\Data\UserUpdatedEventMapper.cs`

```csharp
public class UserUpdatedEventMapper : IEventMapper
{
    public IEnumerable<OutboxMessage> Map(AuthDbContext db, DateTime occurredAt)
    {
        const string UserNameProperty = nameof(PlayBallUser.UserName);

        return db
            .ChangeTracker
            .Entries<PlayBallUser>()
            .Where(entry => entry.State == EntityState.Modified
                            &&
                            entry.OriginalValues.GetValue<string>(UserNameProperty) !=
                            entry.CurrentValues.GetValue<string>(UserNameProperty))
            .Select(entry =>
                new OutboxMessage(occurredAt,
                    new UserUpdatedEvent
                    {
                        Id = Guid.NewGuid(),
                        OccurredAt = occurredAt,
                        UserId = entry.Entity.Id,
                        UserName = entry.Entity.UserName
                    }));
    }
}
```

`Infrastructure\Data\UserDeletedEventMapper.cs`

```csharp
public class UserDeletedEventMapper : IEventMapper
{
    public IEnumerable<OutboxMessage> Map(AuthDbContext db, DateTime occurredAt)
        => db
            .ChangeTracker
            .Entries<PlayBallUser>()
            .Where(entry => entry.State == EntityState.Deleted)
            .Select(entry =>
                new OutboxMessage(occurredAt,
                    new UserDeletedEvent
                    {
                        Id = Guid.NewGuid(),
                        OccurredAt = occurredAt,
                        UserId = entry.Entity.Id
                    }));
}
```

## Transactionally storing the events

This final section has the most pretentious title, but it is probably the most straightforward.

As we discussed some times already, to ensure reliability in event publishing, we need the events to be persisted in the same transaction as the actual changes. However, we don't need to do anything very special.

When calling `SaveChanges`, by default all the changes are persisted in a transaction, so all we need to do in our `SaveChangesAsync` override is add the outbox messages to the context before invoking the base class' implementation. We can see the complete (for now) implementation of `SaveChangesAsync` below.

`Data\AuthDbContext.cs`

```csharp
public class AuthDbContext : IdentityDbContext<PlayBallUser>
{
    // ...

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = new CancellationToken())
    {
        var eventsDetected = GetEvents();
        AddEventsIfAny(eventsDetected);

        var result = await base.SaveChangesAsync(cancellationToken);

        // TODO: publish the events persisted in the outbox

        return result;
    }

    // ...

    private void AddEventsIfAny(IReadOnlyCollection<OutboxMessage> eventsDetected)
    {
        if (eventsDetected.Count > 0)
        {
            Set<OutboxMessage>().AddRange(eventsDetected);
        }
    }
}
```

And with this we have both the user account related changes and the outbox messages persisted transactionally. If something goes wrong when persisting things, an exception is thrown and nothing is committed.

## Outro

That does it for this episode. We took care of the persistence part of the transactional outbox pattern, creating a new table to store the events, as well as storing things transactionally.

The main topics we looked at:

- PostgreSQL and EF Core/Npgsql support for JSON columns (we barely scratched the surface though)
- Transactionally persist changes and outbox messages with EF Core
- Continue taking advantage of overriding `SaveChanges` to centralize some logic

In the next episode, we'll implement the outbox publisher, which will read the messages stored in the outbox table to publish to the event bus.

Links in the post:

- [PostgreSQL documentation: 8.14. JSON Types](https://www.postgresql.org/docs/12/datatype-json.html)
- [Marten - Polyglot Persistence for .NET Systems using the Rock Solid PostgreSQL Database](https://github.com/JasperFx/Marten)
- [Npgsql - .NET data provider for PostgreSQL](https://github.com/npgsql/Npgsql)
- [System.Text.Json](https://docs.microsoft.com/en-us/dotnet/standard/serialization/system-text-json-overview)
- [Json.NET](https://github.com/JamesNK/Newtonsoft.Json)
- [Event-driven integration #1 - Intro to the transactional outbox pattern [ASPF02O|E040]](https://blog.codingmilitia.com/2020/04/13/aspnet-040-from-zero-to-overkill-event-driven-integration-transactional-outbox-pattern/)
- [Event-driven integration #2 - Inferring events from EF Core changes [ASPF02O|E041]](https://blog.codingmilitia.com/2020/04/20/aspnet-041-from-zero-to-overkill-event-driven-integration-inferring-events-from-efcore-changes/)

The source code for this post is in the [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode042) repository, tagged as `episode042`.

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!
