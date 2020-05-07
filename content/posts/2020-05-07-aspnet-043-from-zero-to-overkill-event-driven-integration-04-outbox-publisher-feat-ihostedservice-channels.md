---
author: João Antunes
date: 2020-05-07 10:30:00+01:00
layout: post
title: "Event-driven integration #4 - Outbox publisher (feat. IHostedService & Channels) [ASPF02O|E043]"
summary: "In this episode, we'll implement the outbox publisher, or better yet, two versions of it, one better suited for lower latency and another for reliability. As we continue our event-driven path, this will be a good opportunity to introduce a couple of interesting .NET Core features: IHostedService (and BackgroundService) and System.Threading.Channels."
images:
- '/assets/2020/05/07/aspnet-core-from-zero-to-overkill-e043.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- efcore
- channels
- ihostedservice
slug: aspnet-043-from-zero-to-overkill-event-driven-integration-04-outbox-publisher-feat-ihostedservice-channels
---

In this episode, we'll implement the outbox publisher, or better yet, two versions of it, one better suited for lower latency and another for reliability. As we continue our event-driven path, this will be a good opportunity to introduce a couple of interesting .NET Core features: `IHostedService` (and `BackgroundService`) and `System.Threading.Channels`.

**Note:** depending on your preference, you can check out the following video, otherwise, skip to the written version below.

{{< yt xnn6AnYyC5g >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

In the [previous episode](https://blog.codingmilitia.com/2020/04/28/aspnet-042-from-zero-to-overkill-event-driven-integration-03-storing-events-in-the-outbox-table/), we implemented the outbox, as well as storing the messages in it transactionally. In this episode, we'll implement the outbox publisher (two versions in fact) that's responsible for reading the messages from the table, push them to the event bus and delete them after they're published successfully.

Something we'll see that the outbox publisher takes into consideration is that multiple instances might be running concurrently. Due to this, the publisher is coded in a way to try to avoid publishing the same message multiple times, "try" being the keyword here, as it's not a guarantee we can achieve with this kind of solution.

As briefly pointed out, we'll in fact implement two versions of the outbox publisher, the first geared towards reducing event publishing latency, while the second aimed at reliability, ensuring all the events are published even in the face of transient failures. As you might be suspecting from this quick intro, we could live with just the second one, simplifying our work, but the first allows us to play with a .NET Core feature we haven't used so far, [`System.Threading.Channels`](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/).

Another .NET Core feature we'll use in this episode is running [tasks in the background](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-3.1), by implementing `IHostedService`, which can be done directly or by inheriting from the `abstract` `BackgroundService` class.

Before getting on with business, to situate ourselves in the event-driven integration path, we can take a look at the diagram introduced in [episode 40](https://blog.codingmilitia.com/2020/04/13/aspnet-040-from-zero-to-overkill-event-driven-integration-transactional-outbox-pattern/):

[![situating ourselves](/assets/2020/05/07/e043-outbox-pattern-publisher.png)](/assets/2020/05/07/e043-outbox-pattern-publisher.png)

## Outbox publisher

Let's start with our main outbox publisher, which is triggered every time a new message is stored. This is the implementation that gives us lower event publishing latency, as it doesn't rely on polling, but on being listening for new work.

As introduced, this implementation is not completely reliable by itself. The reason for this, is that from the time the publisher is triggered, to the time it publishes the events, something might go wrong, like the server going down, and the event that caused the publisher execution would remain in the outbox pending publishing. Due to this, we need additional strategies to ensure all events are published, regardless of transient failures. As a result, this outbox publisher implementation becomes more of an optimization, to try to publish the events as soon as possible, as well as an opportunity to play with Channels 🙂.

### Notify when a new message is stored

Picking up where we left in the previous episode, in terms of implementation, we had a comment in the `AuthDbContext`'s `SaveChangesAsync` method to "publish the events persisted in the outbox".

What we'll do is not actually publish the events, as the comment mentioned, but instead notify some interested component that messages were persisted and it can proceed to publish them. As publishing the event is not required to fulfill the user's request, this reduces the time spent waiting for the request to complete.

We could create an interface to represent this notification behavior, but lately I've been more adept of creating delegates instead of single-method interfaces (depending on the scenario of course). With this in mind, we can create an `OnNewOutboxMessages` delegate.

`Data\OnNewOutboxMessages.cs`

```csharp
public delegate void OnNewOutboxMessages(IEnumerable<long> messageIds);
```

We could also achieve the same with a `Func`, but not only giving it a name can make it easier to understand, it also helps when configuring things in the dependency injection container, as we could have different `Func`s with the same signature but different purposes.

Now we can inject the delegate in the `AuthDbContext` and use it when new messages are persisted in the outbox table.

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

        NotifyEventsIfAny(eventsDetected);

        return result;
    }

    // ...

    private void NotifyEventsIfAny(IReadOnlyCollection<OutboxMessage> eventsDetected)
    {
        if (eventsDetected.Count > 0)
        {
            _onNewOutboxMessages(eventsDetected.Select(e => e.Id));
        }
    }
}
```

### Outbox listener

The `AuthDbContext` is ready to notify when a new message is added to the outbox, now we need to create the glue between said notification and some component that runs in the background and actually publishes things to the event bus.

This is where we'll make use of `System.Threading.Channels`. `Channels` help us implement in-memory producer/consumer scenarios, optimized for async code. This fits our problem very nicely, as we want to notify (produce) when a new message is available in persistence, while having another component listening (consume) to that notification to act on it.

To encapsulate this, we can create a class `OutboxListener` (not very happy with the name, but it'll do for now 😛).

#### Creating a channel

Firstly, let's look at the constructor. In there, we're creating the channel we'll use to publish the id of the message stored in the outbox, hence the `Channel<long>` type, meaning we'll have a channel that can contain `long`s, the type of our message ids.

`Infrastructure\Events\OutboxListener.cs`

```csharp
public class OutboxListener
{
    private readonly ILogger<OutboxListener> _logger;
    private readonly Channel<long> _messageIdChannel;

    public OutboxListener(ILogger<OutboxListener> logger)
    {
        _logger = logger;

        // If the consumer is slow, this should be a bounded channel to avoid memory growing indefinitely.
        // To make an informed decision we should instrument the code and gather metrics.
        _messageIdChannel = Channel.CreateUnbounded<long>(
            new UnboundedChannelOptions
            {
                SingleReader = true,
                SingleWriter = false
            });
    }

    // ...
}
```

We can have bounded and unbounded channels, where the first is limited in size and we should elect a strategy to handle a full channel (e.g. wait for space or drop new items), while the latter doesn't have a size restriction. An unbounded channel can be a bit dangerous because if the consumer is slow to process items, memory will grow indefinitely. We'll go with unbounded for now, but keep in mind bounded is likely a better idea.

When creating a channel, we can provide some options, in the unbounded channel case, through the `UnboundedChannelOptions` class. In this case, we're indicating that we'll have a single reader/consumer and multiple writers/producers. With these options, the channel instance we'll get can be optimized for our use case. If we were using a bounded channel, it would be through these options (using the `BoundedChannelOptions` class) that we would be able to set the capacity and the behavior of the channel when full.

#### Writing to a channel

With a channel instance in hand, we can start writing to it. This is done in the `OnNewMessages` method. Notice the method signature matches the delegate we created for `AuthDbContext` to use. This is no coincidence, as this will be configured in the dependency injection container to be provided to the `AuthDbContext`.

`Infrastructure\Events\OutboxListener.cs`

```csharp
public class OutboxListener
{
    // ...

    public void OnNewMessages(IEnumerable<long> messageIds)
    {
        foreach (var messageId in messageIds)
        {
            // we don't care too much if it succeeds because we'll have a fallback to handle "forgotten" messages
            if (!_messageIdChannel.Writer.TryWrite(messageId) && _logger.IsEnabled(LogLevel.Debug))
            {
                _logger.LogDebug("Could not add outbox message {messageId} to the channel.", messageId);
            }
        }
    }

    // ...
}
```

A channel exposes two properties, `Writer` and `Reader` (of types `ChannelWriter<T>` and `ChannelReader<T>` respectively), which provide the methods to write to/read from it. For either case we have multiple options, not a single method for writing and reading, to adapt to our needs.

Concerning the `ChannelWriter`, the methods available are:

- `TryWrite`, as the name implies, tries to write to the channel, returning a boolean to indicate whether it wrote or not. Reasons for not writing may be that the channel is full or completed (no longer accepting new writes).
- `WaitToWriteAsync` doesn't actually write to the channel, instead returning `ValueTask<bool>` that can be awaited to know when space is available to write. If the boolean returned is false, it means it isn't be possible to write anymore.
- `WriteAsync` is a mix between `TryWrite` and `WaitToWriteAsync`. If there is space to write, it writes, otherwise waits for space to be available.
- `TryComplete` is used when we don't want to write to the channel anymore, be it because we have nothing more to write or an exception happened and we want to stop all the things.

Looking at the `OutboxListener` code, we're simply using `TryWrite`. There are a couple of factors for this decision.

The most immediate explanation is, being an unbounded channel, `TryWrite` will always succeed because there are no space issues (the only way to return false, is if the channel is completed).

Additionally, even if we were using a bounded channel, we could still ignore when not being able to write because, as introduced in the beginning of the post, we will have a fallback publishing any pending messages. If we didn't have this fallback, then we'd need to approach things differently. In this case we're be making a tradeoff between the time a user needs to wait for a request to complete and the latency of event publishing.

#### Reading from a channel

Like the `ChannelWriter`, the `ChannelReader` also exposes a number of methods with similar behavior, just applied to reading:

- `TryRead` reads an item from the channel if there is one available, returning true in such a case, otherwise returns false.
- `WaitToReadAsync` doesn't actually read, instead returning a `ValueTask<bool>` that can be awaited to know when an item is available to read. If the boolean returned is false, it means it isn't possible to read anymore (channel completed).
- `ReadAsync` is a mix between `TryRead` and `WaitToReadAsync`. If there is an item to read, it reads, otherwise waits for an item to be available.

If you look at our `OutboxListener` code, you'll notice we're not using any of these.

`Infrastructure\Events\OutboxListener.cs`

```csharp
public class OutboxListener
{
    // ...

    public IAsyncEnumerable<long> GetAllMessageIdsAsync(CancellationToken ct)
        => _messageIdChannel.Reader.ReadAllAsync(ct);
}
```

Besides the methods previously mentioned, `ChannelReader` also exposes a `ReadAllAsync`, used above, which returns an `IAsyncEnumerable`. If you never seen an `IAsyncEnumerable`, which wouldn't be surprising as it's a recent feature (introduced with .NET Core 3.0) like the name implies, it's like an `IEnumerable` but tailored for async scenarios. With it we can use a feature introduced in C# 8, `await foreach`, which allows us to handle async streams in a similar way to traditional iteration on collections. There's a section in ["What's new in C# 8.0"](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-8) about [asynchronous streams](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-8#asynchronous-streams).

### Running in the background

With the `OutboxListener` ready, we can now use it to be notified when new messages are stored in the outbox. To do this, we'll create a background task that starts the process of listening to these notifications. In .NET Core, we can create these kinds of background tasks by implementing an `IHostedService`, either directly or by inheriting from the `BackgroundService` class.

The responsibility of this component, a class named `OutboxPublisherBackgroundService`, will be to listen for notifications and forward to an `OutboxPublisher` class that'll implement the remaining logic.

`Infrastructure\Events\OutboxPublisherBackgroundService.cs`

```csharp
public class OutboxPublisherBackgroundService : BackgroundService
{
    private readonly OutboxPublisher _publisher;
    private readonly OutboxListener _listener;
    private readonly ILogger<OutboxPublisherBackgroundService> _logger;

    public OutboxPublisherBackgroundService(
        OutboxPublisher publisher,
        OutboxListener listener,
        ILogger<OutboxPublisherBackgroundService> logger)
    {
        _publisher = publisher;
        _listener = listener;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // TODO: one message at a time might hinder throughput, consider batching
        await foreach (var messageId in _listener.GetAllMessageIdsAsync(stoppingToken))
        {
            try
            {
                await _publisher.PublishAsync(messageId, stoppingToken);
            }
            catch (Exception ex)
            {
                // We don't want the background service to stop while the application continues,
                // so catching and logging.
                // Should certainly have some extra checks for the reasons, to act on it.
                _logger.LogWarning(ex, "Unexpected error while publishing pending outbox messages.");
            }
        }
    }
}
```

As we can see, we're inheriting from `BackgroundService`, which means we have a single method we need to implement, `ExecuteAsync`. This method returns a `Task`, that when completed means the service has finished its job. In our case, we want it to be running during the whole lifetime of the application, but in other cases we might just want to run some things asynchronously when starting the application.

As for the implementation of `ExecuteAsync`, we're doing the `await foreach` mentioned earlier, handling each message id as it arrives. As noted in the comment, executing the event publishing logic one by one will likely hurt event publishing throughput, so we should consider batching, but we'll keep it simple for now.

For each iteration, we make use of the `OutboxPublisher` class (which we'll see in the next section) to handle the event publishing logic.

Besides that, we catch and log exceptions, because we don't want the service to stop while the application keeps running. Depending on the type of error though, we could  probably improve this.

### Publish an event

Publishing an event happens in the previously mentioned `OutboxPublisher` class.

The `OutboxPublisher` logic consists of:

- Reading the message for the given id from the outbox.
- Publish the event to the event bus.
- Delete the message pertaining the published event from the outbox.

The code to implement this logic is a bit more complex then we would expect from this description, as we want to take some precautions due to the fact multiple publishers might be running concurrently, not in this service, where we have a single one, but we might have multiple instances of the auth service running (e.g. multiple servers or multiple containers).

`Infrastructure\Events\OutboxPublisher.cs`

```csharp
public class OutboxPublisher
{
    private readonly IServiceScopeFactory _serviceScopeFactory;
    private readonly ILogger<OutboxPublisher> _logger;

    public OutboxPublisher(IServiceScopeFactory serviceScopeFactory, ILogger<OutboxPublisher> logger)
    {
        _serviceScopeFactory = serviceScopeFactory;
        _logger = logger;
    }

    public async Task PublishAsync(long messageId, CancellationToken ct)
    {
        using var scope = _serviceScopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AuthDbContext>();

        await using var transaction = await db.Database.BeginTransactionAsync(ct);

        try
        {
            var message = await db.Set<OutboxMessage>().FindAsync(new object[] {messageId}, ct);

            if (await TryDeleteMessageAsync(db, message, ct))
            {
                // TODO: actually push the events to the event bus
                _logger.LogInformation(
                    "Event with id {eventId} (outbox message id {messageId}) published -> {event}",
                    message.Event.Id,
                    message.Id,
                    Newtonsoft.Json.JsonConvert.SerializeObject(message.Event));

                await transaction.CommitAsync();
            }
            else
            {
                await transaction.RollbackAsync(ct);
            }
        }
        catch (Exception)
        {
            await transaction.RollbackAsync();
            throw;
        }
    }

    private async Task<bool> TryDeleteMessageAsync(AuthDbContext db, OutboxMessage message, CancellationToken ct)
    {
        try
        {
            db.Set<OutboxMessage>().Remove(message);
            await db.SaveChangesAsync(ct);
            return true;
        }
        catch (DbUpdateConcurrencyException)
        {
            _logger.LogDebug($"Delete message {message.Id} failed, as it was done concurrently.");
            return false;
        }
    }
}
```

Let's go through `PublishAsync`.

The first thing that comes up is actually not logic related, but needed, which is creating a dependency injection scope and getting a `DbContext` instance from there. We need to do this, because we passed the `OutboxPublisher` to the `OutboxPublisherBackgroundService` through the constructor, and `OutboxPublisherBackgroundService` will live for as long as the application lives. As a `DbContext` shouldn't live for that long (e.g. the change tracker keeps things in memory), we need to control its lifetime manually.

As for actual publishing logic, the first thing we do is starting a transaction. As you might be suspecting, this is due to the precautions I mentioned regarding concurrent publishing.

Immediately after querying the database to get the message with the provided id, we call a `TryDeleteMessageAsync` method, that not only tells the `DbContext` the message should be removed, it actually calls `SaveChangesAsync` to make it so in the database, not just in-memory. Remember though, that we're in a transaction, so even if the deletion is done in the database, it's not committed yet. We do this because if there's a concurrent publisher executing, which for some reason tries to delete the same message, it will be locked until the current transaction is committed or rolled back. This way we minimize the likelihood of publishing the same event multiple times.

`TryDeleteMessageAsync` returns a boolean, where true means the message was successfully deleted and we can proceed with publishing the event, while false is returned when deletion wasn't successful, as we can see in the code, due to a `DbUpdateConcurrencyException`. `DbUpdateConcurrencyException` is the exception that's thrown when a change fails in the database due to another happening concurrently, in this case, another component beat the current executing code to deleting the outbox message.

When deletion of the message is successful, we can publish the event and commit the changes to the database. In the code above there's a log representing the actual publishing to the event bus, as we'll implement that in the coming episodes using [Apache Kafka](https://kafka.apache.org/).

If the message wasn't successfully deleted (or if an unexpected exception occurs), we rollback the transaction.

With this, we wrap up the latency oriented outbox publisher implementation, we can proceed to the reliability oriented version.

## Fallback outbox publisher

Before getting into the implementation details, let's review why do we need to have a fallback for the outbox publisher we just implemented.

The most important reason is to handle cases where a transient failure makes us lose the message ids that were written to the in-memory channel used in the outbox publisher flow. An example of such a failure is the server (or container) going down.

Additionally, having this fallback allows us, as we saw, to have a more naive implementation. Examples of this are:

- If we used a bounded channel and items were dropped, we didn't worry because the fallback would pick them up.
- If the event bus is temporarily down, causing an error to occur when publishing an event, we didn't worry with retries and related patterns, the fallback would pick things up.

This is not to say that the current implementation couldn't use some extra improvements, it likely could, but having this fallback lets us get away with some less thought out approaches.

### Read and publish events

The `OutboxFallbackPublisher` class, which implements the logic to publish any events that got left behind, has many similarities to the `OutboxPublisher` seen previously, being the major difference that it looks for any messages left on the outbox table, instead of just for a given message id.

Let's start with the core logic.

`Infrastructure\Events\OutboxFallbackPublisher.cs`

```csharp
public class OutboxFallbackPublisher
{
    // ...

    public async Task PublishPendingAsync(CancellationToken ct)
    {
        // Invokes PublishBatchAsync while batches are being published, to exhaust all pending messages.

        while (!ct.IsCancellationRequested && await PublishBatchAsync(ct)) ;
    }

    // returns true if there is a new batch to publish, false otherwise
    private async Task<bool> PublishBatchAsync(CancellationToken ct)
    {
        using var scope = _serviceScopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AuthDbContext>();

        await using var transaction = await db.Database.BeginTransactionAsync(ct);

        try
        {
            var messages = await GetMessageBatchAsync(db, ct);

            if (messages.Count > 0 && await TryDeleteMessagesAsync(db, messages, ct))
            {
                // TODO: actually push the events to the event bus
                _logger.LogInformation(
                    "Events with ids {eventIds} (outbox message ids [{messageIds}]) published -> {events}",
                    string.Join(", ", messages.Select(message => message.Event.Id)),
                    string.Join(", ", messages.Select(message => message.Id)),
                    Newtonsoft.Json.JsonConvert.SerializeObject(messages.Select(message => message.Event)));

                await transaction.CommitAsync();

                return await IsNewBatchAvailableAsync(db, ct);
            }

            await transaction.RollbackAsync(ct);

            // if we got here, there either aren't messages available or are being published concurrently
            // in either case, we can break the loop
            return false;
        }
        catch (Exception)
        {
            await transaction.RollbackAsync();
            throw;
        }
    }

    // ...
}
```

As we want to publish all pending events, not just some, `PublishPendingAsync`, which is the only public method of the class, keeps looping while there are pending messages in the outbox, moving the batch publishing logic to `PublishBatchAsync`.

Looking at `PublishBatchAsync`, it's very similar to what we saw in the original `OutboxPublisher`. The main differences we can spot are a call to `GetMessageBatchAsync`, which will provide a number of messages, not a single specific one, as well as returning a boolean indicating if there are more messages available to publish.

Let's now drill down into the methods used to support this logic.

`Infrastructure\Events\OutboxFallbackPublisher.cs`

```csharp
public class OutboxFallbackPublisher
{
    private const int MaxBatchSize = 100;
    private static readonly TimeSpan MinimumMessageAgeToBatch = TimeSpan.FromSeconds(30);

    // ...

    private static Task<List<OutboxMessage>> GetMessageBatchAsync(AuthDbContext db, CancellationToken ct)
        => MessageBatchQuery(db)
            .Take(MaxBatchSize)
            .ToListAsync(ct);

    private static Task<bool> IsNewBatchAvailableAsync(AuthDbContext db, CancellationToken ct)
        => MessageBatchQuery(db).AnyAsync(ct);

    private static IQueryable<OutboxMessage> MessageBatchQuery(AuthDbContext db)
        => db.Set<OutboxMessage>()
            .Where(m => m.CreatedAt < GetMinimumMessageAgeToBatch());

    private async Task<bool> TryDeleteMessagesAsync(
        AuthDbContext db,
        IReadOnlyCollection<OutboxMessage> messages,
        CancellationToken ct)
    {
        try
        {
            db.Set<OutboxMessage>().RemoveRange(messages);
            await db.SaveChangesAsync(ct);
            return true;
        }
        catch (DbUpdateConcurrencyException)
        {
            _logger.LogDebug(
                $"Delete messages [{string.Join(", ", messages.Select(m => m.Id))}] failed, as it was done concurrently.");
            return false;
        }
    }

    private static DateTime GetMinimumMessageAgeToBatch()
    {
        return DateTime.UtcNow - MinimumMessageAgeToBatch;
    }
}
```

Both `GetMessageBatchAsync` and `IsNewBatchAvailableAsync` use `MessageBatchQuery` to have the base query to obtain pending messages. The rationale I used was, if the message is there for more than 30 seconds, it probably means it got left behind, so we should publish it. Using this base query, `GetMessageBatchAsync` fetches a batch of messages, while `IsNewBatchAvailableAsync` simply checks if there are any messages pending that match the defined criteria.

`TryDeleteMessagesAsync` is the same as we saw in the `OutboxPublisher`, differing just in that it deletes multiple rows, not just one.

`GetMinimumMessageAgeToBatch` is a helper method to calculate the minimum age a message should be to qualify as pending (side note, using `DateTime.UtcNow` directly is not great for unit testing).

### Scheduling execution

To wrap things up about the `OutboxFallbackPublisher`, we need to schedule its execution. To do this, we can again resort to a `BackgroundService`.

`Infrastructure\Events\OutboxPublisherFallbackBackgroundService.cs`

```csharp
public class OutboxPublisherFallbackBackgroundService : BackgroundService
{
    private readonly OutboxFallbackPublisher _fallbackPublisher;
    private readonly ILogger<OutboxPublisherFallbackBackgroundService> _logger;

    public OutboxPublisherFallbackBackgroundService(
        OutboxFallbackPublisher fallbackPublisher,
        ILogger<OutboxPublisherFallbackBackgroundService> logger)
    {
        _fallbackPublisher = fallbackPublisher;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await _fallbackPublisher.PublishPendingAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                // We don't want the background service to stop while the application continues,
                // so catching and logging.
                // Should certainly have some extra checks for the reasons, to act on it. 
                _logger.LogWarning(ex, "Unexpected error while publishing pending outbox messages.");
            }

            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }
}
```

Similarly to the `OutboxPublisherBackgroundService`, we just want to get the publisher to do its work. In this case, as we don't subscribe to anything, we take a polling approach. We call the publisher to process any pending messages, and when it's done, we "sleep" for 30 seconds, instead of hammering the database continuously.

## Wiring everything together

To get everything working together, what's left is setting things up in the dependency injection container. This is done in an `EventExtensions` class created to keep the `Startup` class clean.

`IoC\EventExtensions.cs`

```csharp
public static class EventExtensions
{
    public static IServiceCollection AddEvents(this IServiceCollection services)
    {
        services.Scan(
            scan => scan
                .FromAssemblyOf<UserRegisteredEventMapper>()
                .AddClasses(classes => classes.AssignableTo(typeof(IEventMapper)))
                .AsImplementedInterfaces()
                .WithSingletonLifetime()
        );

        services.AddSingleton<OutboxListener>();
        services.AddSingleton<OnNewOutboxMessages>(s => s.GetRequiredService<OutboxListener>().OnNewMessages);
        services.AddSingleton<OutboxPublisher>();
        services.AddSingleton<OutboxFallbackPublisher>();

        services.AddHostedService<OutboxPublisherBackgroundService>();
        services.AddHostedService<OutboxPublisherFallbackBackgroundService>();

        return services;
    }
}
```

The scan for event mappers was already there, from previous episodes, so the new stuff is what comes after.

`OutboxListener`, `OutboxPublisher` and `OutboxFallbackPublisher` are registered as usual. They're all singletons, `OutboxListener` really needs to be, because we need to keep using the same channel to notify of new messages. `OutboxPublisher` and `OutboxFallbackPublisher` don't need to be singleton by themselves, but as they'll be used by the background services that have the same the lifetime as the application, as we already discussed, it makes sense to make them singleton as well.

The registration of `OnNewOutboxMessages` might be slightly different from what's common, because we want to associate a specific instance method with the delegate. That's why we're making use of overload that accepts a `Func`, where we get an `IServiceProvider` to obtain the `OutboxListener` from which we want to bind the `OnNewMessages` method with the delegate used by the `AuthDbContext`.

Finally, `OutboxPublisherBackgroundService` and `OutboxPublisherFallbackBackgroundService` are registered using the `AddHostedService`, which internally registers the background service as a singleton.

## Outro

That does it for this episode. We implemented the outbox publisher, two versions of it to be more precise, while playing with some interesting features of .NET Core - channels and background services.

Summarizing, the main topics we looked at were:

- Using channels to implement in-memory producer/consumer scenarios, optimized for async code.
- Implementing background tasks using `IHostedService`/`BackgroundService`.
- Reading and publishing messages from the outbox, taking concurrent execution into consideration.

As a quick reminder, the achieved solution might be a bit overkill, as we could get away with just the polling solution, but we wouldn't have the opportunity to play with all the things we did 🙂.

In the next episodes, we'll introduce Apache Kafka and implement event publishing/subscription on top of it.

Links in the post:

- [An Introduction to System.Threading.Channels](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/)
- [Background tasks with hosted services in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-3.1)
- [BackgroundService](https://github.com/dotnet/runtime/blob/master/src/libraries/Microsoft.Extensions.Hosting.Abstractions/src/BackgroundService.cs)
- ["What's new in C# 8.0"](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-8)
- [Event-driven integration #1 - Intro to the transactional outbox pattern [ASPF02O|E040]](https://blog.codingmilitia.com/2020/04/13/aspnet-040-from-zero-to-overkill-event-driven-integration-transactional-outbox-pattern/)
- [Event-driven integration #3 - Storing events in the outbox table [ASPF02O|E042]](https://blog.codingmilitia.com/2020/04/28/aspnet-042-from-zero-to-overkill-event-driven-integration-03-storing-events-in-the-outbox-table/)

The source code for this post is in the [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode043) repository, tagged as `episode043`.

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!