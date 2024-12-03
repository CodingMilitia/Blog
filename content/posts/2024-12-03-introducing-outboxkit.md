---
author: João Antunes
date: 2024-12-03 08:45:00+00:00
layout: post
title: Introducing OutboxKit
summary: After talking about the outbox pattern so much, and implementing it from scratch a few times in a few different projects, finally figured it was high time for me to build something I could reuse.
images:
  - /images/2024/12/03/introducing-outboxkit.png
categories:
  - csharp
  - dotnet
tags:
  - outbox
  - mysql
slug: introducing-outboxkit
---

## Intro

If you've read this blog before, you possibly noticed multiple posts about the transactional outbox pattern. After talking about it so much, and implementing it from scratch a few times in a few different projects, finally figured it was high time for me to build something I could reuse.

This post starts with a quick refresher on what transactional outbox pattern is, then goes onto the motivations for building OutboxKit, its features and future plans.

If you know the pattern by heart and just want to know what the hell is this OutboxKit thing, feel free to skip the next section.

TLDR: [here's the code](https://github.com/YakShaveFx/outboxkit)

## Refresher on the transactional outbox pattern

First things first, what's the problem we're trying to address?

Let's imagine we have an HTTP API, exposed by *some service*. We make a `POST` request to an endpoint, which executes some logic, persists changes to a database, then produces a message to messaging infrastructure (typically this would be an event published to a topic, but could also be a command sent to a queue).

The following diagram presents this sample scenario.

{{< embedded-image "/images/2024/12/03/01-pitfall.png" "typical flow" >}}

At first, this looks pretty reasonable. However, due to the orchestration of different dependencies across transactional boundaries, there's potential to get the system in an inconsistent state.

Imagine that after we persist the changes to the database, we're unable to produce the message for some reason, be it the broker being temporarily down, a network hiccup, or even the service itself terminating abruptly (among many other possible failure scenarios). If this happens, we might find ourselves in an inconsistent state, as the downstream services will never get the message they were supposed to.

The transactional outbox pattern presents a possible solution to tackle the described problem. The gist of it is: when we persist the business data, we also persist the data for the message we want to produce, to the same data store, within the same transaction. Because we do both operations in a transactional way, we know that either everything succeeds or nothing succeeds. With the message data in the database, we can have another component read it and produce it to the message broker. If producing fails for some reason, this component will continue retrying until it succeeds.

The following diagram showcases this example.

{{< embedded-image "/images/2024/12/03/02-outbox-pattern.png" "outbox pattern flow flow" >}}

The outbox producer can be implemented in different ways, namely using polling or push approaches. When using polling, the producer checks for new messages at a regular time interval. Alternatively, when using push, the new messages are pushed to the producer. A typical way to implement push with a relational database would be by tailing the transaction log.

The outbox producer is presented as an internal component of the service, but it's not necessarily so, it can also be implemented out of process.

As for types of databases, while the example presented assumes a relational one (as we can infer by the usage of the "table" designation), the pattern is also applicable with non-relational databases, though it might differ a bit in implementation, depending on the capabilities of the specific database.

One final important note, is that the outbox pattern helps in guaranteeing at least once delivery. This means that it's possible some messages are produced more than once, e.g. if after producing the messages, there's a failure when (or before) marking them as processed. This shouldn't be a big deal, as the message consumers should be idempotent either way, given there are a bunch of ways for the messages to reach the consumers more than once, but it's still important to keep in mind.

## Enter OutboxKit

Now that we're on the same page regarding what is the transactional outbox pattern and why it's relevant, we can look into OutboxKit.

OutboxKit is a toolkit to help with implementing the transactional outbox pattern. Note, and this is important, it is not a completely ready to use, plug and play implementation of the pattern. It provides building blocks to (hopefully) greatly reduce the work of implementing things, but it's not a full blown implementation.

The focus of OutboxKit is on reading messages from an outbox and making them available to produce, in the most resilient way possible. Database setup (creating tables, indexes, etc) or even pushing messages into the outbox are outside the scope of the project.

With this in mind, in general I'd say it's best not use OutboxKit (yeah, you read it right 😅). Using more comprehensive libraries like [Wolverine](https://wolverinefx.net), [NServiceBus](https://particular.net/nservicebus), [Brighter](https://github.com/BrighterCommand/Brighter), [MassTransit](https://masstransit.io), etc, is probably a better idea, as they give a more integrated messaging experience (plus they've been at this for much longer, and in a wider range of scenarios). This toolkit is aimed at scenarios where greater degree of customization of the whole messaging approach is wanted.

I've implemented the outbox pattern a few times already over the years, some when exploring and experimenting with things for the blog, others in production systems. While implementing this isn't rocket science, it's also not trivial, so I guess the time as final come for me to build something I can reuse.

## Core features

At the time of writing, the core features of OutboxKit revolve around the implementation of a polling outbox approach. From the experiments I've been doing, the push approach seems less generalizable and more database specific, so possibly most of this logic will be on database specific libraries instead of the core, but it's still something to look into going forward.

Main features provided by OutboxKit Core can be grouped as: message production, cleanup and polling, where the first two are more generally applicable, while the latter is specific to polling implementations.

### Produce messages

On the subject of message production, the `IBatchProducer` interface is exposed, which should be implemented by any application using OutboxKit (or library built on top of it). Whenever a batch of messages is up for production, the `ProduceAsync` method is invoked, and its implementation is responsible for producing the messages to whatever message broker is used.

### Cleaning things up

Regarding cleanup, the need for it depends on the desired approach upon message production. If when a message is produced it's immediately deleted, then there's no need for cleanup. Alternatively, if there's a preference for marking a message as processed instead of deleting immediately, then the cleanup process is important to avoid having the outbox growing indefinitely.

Implementing the interaction with the database to actually clean things is up to a database specific provider, which should implement the `IOutboxCleaner` interface, that is orchestrated by the core logic.

### Polling

As for polling, things revolve around a polling background service which checks for messages to produce at a regular interval. Fetching the messages themselves, as well as later marking them as processed, lives outside the core, being implemented by database specific providers. These providers should implement the `IBatchFetcher` interface, dealing with the specifics of the database, and leaving the overall orchestration to the core.

As an optimization, to reduce the latency between inserting messages into the outbox and producing them, when the producer lives in the same process as the service that's inserting the messages into the outbox, the application may use either the `IOutboxTrigger` or the `IKeyedOutboxTrigger` interface to trigger the producer immediately after inserting a new message.

Common to everything discussed here, is the fact that OutboxKit is multi-provider and multi-database ready.

Regarding multi-provider, this means you can have more than one provider running at the same time in the same application, allowing you to have an outbox, for example, in MySQL and another in MongoDB, if you have your business data spread like that and it's useful to have the outbox living with each of them.

As for multi-database, similarly to multi-provider, it allows you to have multiple databases for the same provider at the same time in one application. This would be useful, for example, in multi-tenant scenarios, where business data for each tenant is stored in a different database, allowing for the outbox to live with its respective tenant.

### Setup

OutboxKit exposes some setup facilities, being most of them expected to be used by database specific providers, not directly by applications/libraries built on top of it.

To start setting things up, `AddOutboxKit`, an `IServiceCollection` extension method, should be invoked. As an argument it receives an `Action<IOutboxKitConfigurator>`, on top of which the database specific providers can bolt their setup on, allowing everything to feel consistent.

We'll see all of this in action in a later section.

## First provider: MySQL polling

For the first database specific provider I went with MySQL, for no other reason other than, it's what I need at work right now.

In this provider, there were three things needing implementation: message fetching/completion, cleanup and setup.

For this, the steps mentioned earlier were followed:

- Implementation of `IBatchFetcher`
- Implementation of `IOutboxCleaner`
- Extensions over `IOutboxKitConfigurator`, to setup MySQL polling

The `IBatchFetcher` uses a transaction, along with `SELECT ... FOR UPDATE` to fetch the messages from the outbox table and keep them locked until the request to mark them as completed comes in (I have some ideas to change this approach in the future, for better performance). When the completion request comes in, the messages are either deleted or updated with the date/time UTC at which they were processed.

The `IOutboxCleaner` implementation is pretty straightforward, as it simply issues a delete query, with a where clause finding any rows that exceeded the configured maximum age.

As for the configuration extensions, there's a lot going on (possibly there's more configuration code than actual code to implement the library's required logic 😅). The connection string is of course a mandatory configuration, but besides that, we can configure the outbox table to match what we have in our system, the polling interval, the size of the batch that should be fetched, as well as how the message completion should be done (delete vs update).

The most complex and important configuration here is the one for the outbox table. The library includes a default/reference implementation of a message. If the client application is happy with it, along with the default table and column names (which use the MySQL convention of `snake_case`), there's no setup to do. If, however, something is changed, then everything needs to be wired up in order for the provider to be able to do its thing.

## OpenTelemetry

If this isn't your first visit to this blog, you'll know I'm very interested in OpenTelemetry, so of course I'll shove it into anything I'm building, and OutboxKit isn't an exception 😁.

The library is instrumented with logs, traces and metrics. To setup traces and metrics, we add a reference to the `Core.OpenTelemetry` package (didn't need database specific versions so far, may change in the future), then using the `AddOutboxKitInstrumentation` extension methods available for the `TracerProviderBuilder` and `MeterProviderBuilder` types.

Besides the libraries being instrumented themselves, we also find a `TraceContextHelpers` type, which exposes some methods to help with the propagation of the trace context, in order for the message to retain the original context of its generation.

As an example, take the following trace, which includes the production of a few messages.

{{< embedded-image "/images/2024/12/03/03-trace-request-with-message-produce.png" "request trace with message production" >}}

If we didn't store the trace context in the outbox with the rest of the message, we wouldn't be able to have this correlation between the original request and the eventual message production, as instead the messages would be part of the trace associated with outbox producer.

Because we're restoring the trace context, the outbox producer trace doesn't have any indication of the messages being produced, only the interactions with the database.

{{< embedded-image "/images/2024/12/03/04-trace-outbox-produce.png" "outbox production trace" >}}

If we find having some way to correlate the message production span with the outbox producer trace, we can include a link to it, which we can then see in the span (notice the trace id in the span below, matches the trace id used in the query above).

{{< embedded-image "/images/2024/12/03/05-trace-request-link-to-outbox.png" "request trace with link to outbox" >}}

As for metrics, the ones related to actually producing the messages are up to the code that actually produces them, but OutboxKit exposes some related to the batches being produced, as we can see at the bottom of the following dashboard.

{{< embedded-image "/images/2024/12/03/06-metrics-dashboard.png" "sample metrics dashboard" >}}

## Putting it all together

So, I yapped a lot, time to see some sample code (though I'm unsure anyone will read all of this, let me know if you did 🤣). The best way to see some sample code, is going to the [samples folder in the repo](https://github.com/YakShaveFx/outboxkit/tree/main/samples), but I'll point out some potentially interesting bits here.

### Setup

Regardless of particular configuration needs, providers, etc, the thing that always needs to be setup is the batch producer. OutboxKit expects it to be registered in DI as a singleton, so we can do something like:

```csharp
services.AddSingleton<IBatchProducer, FakeBatchProducer>();
```

As mentioned earlier, the MySQL provider includes a default message implementation, so if we follow what it expects to a T, the setup is just a couple of lines long:

```csharp
services.AddOutboxKit(kit =>
    kit.WithMySqlPolling(p => p.WithConnectionString(connectionString)));
```

If we're in a multi-database scenario, we need to invoke the setup method once per each of them, this time explicitly including a key to identify the various outboxes (if nothing is passed, the string `default` is used, which is fine for single database, won't work for multi-database).

```csharp
services.AddOutboxKit(kit =>
    kit
        .WithMySqlPolling(
            tenantOne, p => p.WithConnectionString(connectionStringOne))
        .WithMySqlPolling(
            tenantTwo, p => p.WithConnectionString(connectionStringTwo)));
```

If we want something more custom, then it won't be so terse.

```csharp
services.AddOutboxKit(kit =>
    kit
        .WithMySqlPolling(p =>
            p
                .WithConnectionString(connectionString)
                .WithPollingInterval(TimeSpan.FromSeconds(30))
                .WithBatchSize(50)
                .WithTable(t => t
                    .WithName("OutboxMessages")
                    .WithColumnSelection(
                        ["Id", "Type", "Payload", "CreatedAt", "TraceContext"])
                    .WithIdColumn("Id")
                    .WithSorting([new SortExpression("Id")])
                    .WithIdGetter(m => ((OutboxMessage)m).Id)
                    .WithMessageFactory(static r => new OutboxMessage
                    {
                        Id = r.GetInt64(0),
                        Type = r.GetString(1),
                        Payload = r.GetFieldValue<byte[]>(2),
                        CreatedAt = r.GetDateTime(3),
                        TraceContext = r.IsDBNull(4)
                            ? null
                            : r.GetFieldValue<byte[]>(4)
                    })
                    .WithProcessedAtColumn("ProcessedAt"))
                .WithUpdateProcessed(u => u
                    .WithCleanUpInterval(TimeSpan.FromMinutes(1))
                    .WithMaxAge(TimeSpan.FromMinutes(2)))))
```

### Inserting into the outbox

As mentioned earlier, inserting messages into the outbox is outside the scope of OutboxKit itself, which doesn't mean it can't include some samples in the docs (when I eventually get to write them 😅).

As the most common data access technology in C# is Entity Framework Core, and we also use it at work (even though I complain often), I used EF for the first samples. With that in mind, inserting a message into the outbox can look something like this:

```csharp
app.MapPost("/publish", async (Faker faker, SampleContext db) =>  
{
    var message = new OutboxMessage
    {
        Type = "sample",
        Payload = Encoding.UTF8.GetBytes(faker.Hacker.Verb()),
        CreatedAt = DateTime.UtcNow,
        TraceContext = TraceContextHelpers.ExtractCurrentTraceContext()
    };
    await db.OutboxMessages.AddAsync(message);
    await db.SaveChangesAsync();
});
```

Keep in mind this is an example, hence not doing anything other than inserting a message into the outbox table, no business logic, no nothing.

As expected, nothing particularly interesting is going on. We're simply instantiating an `OutboxMessage` and initializing it with some values, adding to the `DbSet` and saving the changes. Remember that all of these properties are configurable, but the ones you see are what is commonly used, so they should be a good reference.

Notice that the `TraceContext` initialization is making use of the `TraceContextHelpers` I mentioned earlier, which will extract the information from the current `Activity` into a `byte[]`, so it can be stored in the outbox.

### Batch producer

Let's take a look at a sample batch producer implementation, which, as mentioned earlier, should implement the `IBatchProducer` interface.

```csharp
internal sealed class FakeBatchProducer(ILogger<FakeBatchProducer> logger)
    : IBatchProducer
{
    public Task<BatchProduceResult> ProduceAsync(
        OutboxKey key,
        IReadOnlyCollection<IMessage> messages,
        CancellationToken ct)
    {
        logger.LogInformation("Producing {Count} messages", messages.Count);
        foreach (var message in messages.Cast<OutboxMessage>())
        {
            using var activity = StartActivityFromContext(message.TraceContext);

            logger.LogInformation("Got a message!");
        }

        return Task.FromResult(new BatchProduceResult { Ok = messages });
    }
}
```

When a batch is up for production, OutboxKit will invoke the `ProduceAsync` method, including an `OutboxKey` and a collection of messages to produce (plus a `CancellationToken`, which is par for the course for async methods).

The `OutboxKey` is composed by a provider key `string`, which in this case will be `mysql_polling`, as well as a client key `string`, which we configure when invoking `WithMySqlPolling` (or `default` if we don't set anything). This key might be useful (or not), depending on if you rely on this information to do something in your producer.

If our messages have ordering requirements, it's up to us to ensure they are respected. OutboxKit will provide messages batches in the order we want, that's why we have the `WithSorting` method available when we're setting up the table. As we get the ordered messages passed in to the batch producer, it's again up to us to ensure we produce them in the order we should.

At the end of the method, we return a `BatchProduceResult`, indicating which messages were successfully produced, so OutboxKit can complete them.

If we want to have the message production retain the context of when it was created, this is the place to do it, as we can see with the call to `StartActivityFromContext`, which is implemented as follows, with the assistance of the `TraceContextHelpers`:

```csharp
internal sealed class FakeBatchProducer(/* ... */) : IBatchProducer
{
    public static ActivitySource ActivitySource { get; }
        = new(typeof(FakeBatchProducer).Assembly.GetName().Name!);

    public Task<BatchProduceResult> ProduceAsync(/* ... */)
    {
        // ...
    }

    private static Activity? StartActivityFromContext(byte[]? traceContext)
    {
        // using static from TraceContextHelpers
        var parentContext = RestoreTraceContext(traceContext);
        
        // keeping a link to the outbox producer activity
        var links = Activity.Current is { } currentActivity
            ? new[] { new ActivityLink(currentActivity.Context) }
            : default;

        return ActivitySource.StartActivity(  
            "produce message",
            ActivityKind.Producer,
            parentContext: parentContext?.ActivityContext ?? default,
            links: links);
    }
}
```

### Triggering the outbox producer

If we want to make use of the optimization mentioned earlier to reduce the latency between inserting messages into the outbox and producing them, we need to use either `IOutboxTrigger` or `IKeyedOutboxTrigger`, depending we're in a single database and single provider scenario or not.

A potential solution, when using EF, is to implement a `SaveChangesInterceptor` as follows:

```csharp
public sealed class OutboxInterceptor(IOutboxTrigger trigger)
    : SaveChangesInterceptor
{
    private bool _hasOutboxMessages;  
  
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(  
        DbContextEventData eventData,  
        InterceptionResult<int> result,  
        CancellationToken cancellationToken = new())  
    {
        if (eventData.Context is SampleContext db
            && db.ChangeTracker.Entries<OutboxMessage>().Any())
        {
            _hasOutboxMessages = true;
        }
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }  
  
    public override async ValueTask<int> SavedChangesAsync(  
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken cancellationToken = new())
    {
        if (_hasOutboxMessages)
        {
            trigger.OnNewMessages();
        }
  
        return await base.SavedChangesAsync(
            eventData,
            result,
            cancellationToken);
    }
}
```

In a multi-database and/or multi-provider scenario, it could look something like this:

```csharp
public sealed class OutboxInterceptor(
    IKeyedOutboxTrigger trigger, ITenantProvider tenantProvider)
    : SaveChangesInterceptor
{
    // ... (same as the other)

    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken cancellationToken = default)
    {
        if (_hasOutboxMessages)
        {
            trigger.OnNewMessages(
                MySqlPollingProvider.CreateKey(tenantProvider.Tenant));
        }
  
        return await base.SavedChangesAsync(
            eventData,
            result,
            cancellationToken);
    }
}
```

## Future plans

Along with this post, I'm releasing version 0.0.1 of OutboxKit, which as you might have understood from the features described until now, is focused on the base implementation of a polling outbox, with MySQL as the sole database provider.

Going forward, I have a bunch of ideas and things I want to implement, namely some improvements to the MySQL polling implementation, MongoDB polling and push(ish) implementations, as well as PostgreSQL polling and push implementations.

As should be obvious, being a side project, I'll get to it **when I can and feel like it**, so no promises regarding features or dates.

## Outro

That does it for this, not so quick, intro to OutboxKit.

I'm not expecting this set of libraries to be used much, if at all, by anyone but me, but if it does fit your use case and it looks interesting, have at it. Though, to reiterate a previous point, there's no shortage of great .NET libraries providing more complete feature sets around messaging.

If nothing else, besides being useful for me at work, it's been interesting to develop OutboxKit, as I've been exploring more possibilities, learning some new things about different databases and improving things I had implemented before.

Relevant links:

- [OutboxKit @ GitHub](https://github.com/YakShaveFx/outboxkit)
- [OutboxKit docs](https://outboxkit.yakshavefx.dev)
- [Wolverine](https://wolverinefx.net)
- [NServiceBus](https://particular.net/nservicebus)
- [Brighter](https://github.com/BrighterCommand/Brighter)
- [MassTransit](https://masstransit.io)

Thanks for stopping by, cyaz! 👋
