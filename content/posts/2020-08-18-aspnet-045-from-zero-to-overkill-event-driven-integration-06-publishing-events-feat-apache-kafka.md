---
author: João Antunes
date: 2020-08-18 11:00:00+01:00
layout: post
title: "Event-driven integration #6 - Publishing events (feat. Apache Kafka) [ASPF02O|E045]"
summary: "In this episode, we implement event publishing to Apache Kafka from the auth service, making use of Confluent's .NET client package."
images:
- '/assets/2020/08/18/aspnet-core-from-zero-to-overkill-e045.jpg'
categories:
- fromzerotooverkill
tags:
- kafka
- events
- messaging
slug: aspnet-045-from-zero-to-overkill-event-driven-integration-06-publishing-events-feat-apache-kafka
---

In this episode, we implement event publishing to Apache Kafka from the auth service, making use of Confluent's .NET client package.

**Note:** depending on your preference, you can check out the following video, otherwise, skip to the written version below.

{{< yt T2Dy7cH486c >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

In the previous episode, we took a break from code to introduce the basics of [Apache Kafka](https://kafka.apache.org/). In this one, we'll start implementing things to integrate Kafka in our system.

The first step we'll take is create an event publishing abstraction, to decouple our publishers, in this case the auth service, from the actual Kafka implementation. With the abstraction ready, we'll introduce it to the auth service and invoke it in our existing outbox publishers, introduced in [episode 43](https://blog.codingmilitia.com/2020/05/07/aspnet-043-from-zero-to-overkill-event-driven-integration-04-outbox-publisher-feat-ihostedservice-channels/).

For the Kafka publisher implementation, we'll use [Confluent's .NET Client](https://github.com/confluentinc/confluent-kafka-dotnet).

Before getting on with business, to situate ourselves in the event-driven integration path, we can take a look at the diagram introduced in [episode 40](https://blog.codingmilitia.com/2020/04/13/aspnet-040-from-zero-to-overkill-event-driven-integration-transactional-outbox-pattern/):

[![situating ourselves](/assets/2020/08/18/e045-outbox-pattern-event-bus.png)](/assets/2020/08/18/e045-outbox-pattern-event-bus.png)

## Event publisher interface

As we'll likely want to publish events from multiple services, not just the auth service, instead of adding the event publishing abstraction to this project, we can create it somewhere to be shared. With this in mind, there is a [Shared](https://github.com/AspNetCoreFromZeroToOverkill/Shared) repository in the `AspNetCoreFromZeroToOverkill` organization we can use to add this code.

After creating the required project to keep our developments, we can create the interface. The goal is for the interface to be very simple, to hide unneeded complexity from the services that need to publish events. For this reason, the `IEventPublisher` exposes just a couple of methods, one to publish one event, another to publish a collection of events.

`CodingMilitia.PlayBall.Shared.EventBus\IEventPublisher.cs`

```csharp
public interface IEventPublisher<in TEvent>
{
    Task PublishAsync(TEvent @event, CancellationToken ct);

    Task PublishAsync(IEnumerable<TEvent> events, CancellationToken ct);
}
```

Well, that's about all for the interface 🙂. Like I said, as simple as possible, for the services to be able to easily publish an event.

When configuring the implementation to use in the auth service, which will happen at the dependency injection level, there will be more Kafka details visible (as we'll see in a bit), but for the remaining code to request an event being published, it can be simplified to this point.

## Kafka event publisher implementation

Now to implement the `IEventPublisher` interface. First thing to do, install Confluent's .NET client.

```bash
dotnet add package Confluent.Kafka
```

Created a sub-folder named `Kafka` in the project, then created a new class named `KafkaEventPublisher` (the `Kafka` prefix is redundant given the namespace, but let's ignore that 😛).

The class itself doesn't do too much, as it's basically just an abstraction on top of the client provided by the `Confluent.Kafka` package.

Let's skim through the main parts of the class.

`CodingMilitia.PlayBall.Shared.EventBus\Kafka\KafkaEventPublisher.cs`

```csharp
public class KafkaEventPublisher<TKey, TEvent> : IEventPublisher<TEvent>, IDisposable
{
    // ...
    
    public KafkaEventPublisher(
        string topic,
        KafkaSettings settings,
        ISerializer<TKey> keySerializer,
        ISerializer<TEvent> valueSerializer,
        Func<TEvent, TKey> keyProvider)
    {
        _topic = topic;
        _keyProvider = keyProvider;

        var config = new ProducerConfig
        {
            BootstrapServers = string.Join(",", settings.BootstrapServers),
            Partitioner = Partitioner.Consistent
        };

        var producerBuilder = new ProducerBuilder<TKey, TEvent>(config)
            .SetValueSerializer(valueSerializer);

        if (keySerializer != null)
        {
            producerBuilder.SetKeySerializer(keySerializer);
        }

        _producer = producerBuilder.Build();
    }
    
    // ...
}
```

We get some things we need as constructor parameters:

- The topic name where events will be published
- Some general Kafka settings, which right now are only comprised of the servers to connect to
- Serializers for the key and the value, so the publisher remains agnostic to the format in which things are stored in Kafka
- As the class is generic, we don't know which property should be used as the key (or maybe it's not a single property but something computed), so we also get a key provider function

Then we make use of these things to initialize the NuGet package provided client.

The `ProducerConfig` we can see being instantiated, has far more options than the ones used here, so be sure to check them out. Right now, we're only setting up the Kafka servers to connect to, as well as the partitioner operating mode, which is set to `Consistent`, which will use a hash of the key to consistently deliver the records to the partitions, enabling records with the same key going to the same partition.

As for the rest, setting up the provided serializers and building the producer instance.

After that, we have the publish methods.

`CodingMilitia.PlayBall.Shared.EventBus\Kafka\KafkaEventPublisher.cs`

```cs
public class KafkaEventPublisher<TKey, TEvent> : IEventPublisher<TEvent>, IDisposable
{
    // ...

    public async Task PublishAsync(TEvent @event, CancellationToken ct)
    {
        await _producer.ProduceAsync(
            _topic,
            new Message<TKey, TEvent>
            {
                Key = _keyProvider(@event),
                Value = @event,
                Timestamp = Timestamp.Default
            },
            ct);
    }

    public async Task PublishAsync(IEnumerable<TEvent> events, CancellationToken ct)
    {
        // could be interesting to improve if there's some batch optimized alternative

        foreach (var @event in events)
        {
            await _producer.ProduceAsync(
                _topic,
                new Message<TKey, TEvent>
                {
                    Key = _keyProvider(@event),
                    Value = @event,
                    Timestamp = Timestamp.Default
                },
                ct);
        }
    }
    
	// ...
}
```

The publish methods simply make use of the producer instance to send the events to Kafka.

The first one is a direct call to `ProduceAsync`, while the second one iterates over the provided collection of events. This second one is a naïve implementation, as doing things this way will result in worse throughput, so it's probably worth it to investigate ways to not make everything sequentially, while keeping in mind that trying to parallelize everything can cause ordering guarantees to be lost.

## An indirection to simplify clients: topic distributor

Warning ⚠: this section is about overengineering 😅.

As you might have noticed, the `KafkaEventPublisher` we just saw is bound to a specific Kafka topic. This means, given the way it was implemented, we need different instances to publish to different topics.

This could be avoided, for example, by passing in the topic as a parameter of the publish method. Instead, I wanted to simplify the client code as much as possible, so my idea is to have an `IEventPublisher` instance injected, the client calls publish with the event(s) and everything else is taken care of. This results in a good amount of overengineering.

Given the likely unneeded complexity of this part, with reflection and expression trickery, I'm not going to put the code for the `TopicDistributor` class here, but as always it's in the [GitHub repo](https://github.com/AspNetCoreFromZeroToOverkill/Shared/blob/episode045/src/CodingMilitia.PlayBall.Shared.EventBus/TopicDistributor.cs).

In case you check out the code, the gist of it is:

- `TopicDistributor` implements the `IEventPublisher` interface, acting like a proxy between the client application and the `KafkaEventPublisher` 
- `TopicDistributor` gets a collection of types, where each is the base type for events that should go to the same topic
  - e.g. `BaseUserEvent` is the base class for events related to user changes
- All these types should have a shared base as well, in order to use it as `IEventPublisher` generic argument
  - e.g. `BaseAuthEvent` is the base for `BaseUserEvent` and any other events published by the auth service
- `KafkaEventPublisher` is configured normally in the dependency injection container, the `TopicDistributor` gets the correct instance from it
- When the client application calls `PublishAsync`, the `TopicDistributor` matches that type to the correct `KafkaEventPublisher`, forwarding the event(s) to it

## Running Kafka locally

I have no desire to invest too much time in setting up Kafka, so I tried to find the easiest and quickest way to get it running locally 🙂.

My idea, like we did for the PostgreSQL database, is to use Docker to quickly get things running. It is however not as simple, because Kafka has at least one dependency, ZooKeeper, and it would be nice if we had some way to inspect what's going on in Kafka.

While investigating the subject, came across a [repository by Confluent](https://github.com/confluentinc/cp-all-in-one) with sample Docker compose files, to get Kafka and related services up and running.

To get things running for our project, copied the contents of `cp-all-in-one` relative to Kafka, ZooKeeper and the Control Center, so now the Docker compose file I use to start dependencies [looks like this](https://github.com/AspNetCoreFromZeroToOverkill/Tools/blob/master/run/docker-compose-dependencies.yml):

```docker
version: "3"
services:
    postgres-db:
        image: "postgres"
        ports:
            - "5432:5432"
        environment:
            POSTGRES_USER: "user"
            POSTGRES_PASSWORD: "pass"
    zookeeper:
        image: confluentinc/cp-zookeeper:5.5.0
        hostname: zookeeper
        container_name: zookeeper
        ports:
            - "2181:2181"
        environment:
            ZOOKEEPER_CLIENT_PORT: 2181
            ZOOKEEPER_TICK_TIME: 2000
    broker:
        image: confluentinc/cp-kafka:5.5.0
        hostname: broker
        container_name: broker
        depends_on:
            - zookeeper
        ports:
            - "29092:29092"
            - "9092:9092"
        environment:
            KAFKA_BROKER_ID: 1
            KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
            KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
            KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
            KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
            KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    
    control-center:
        image: confluentinc/cp-enterprise-control-center:5.5.0
        hostname: control-center
        container_name: control-center
        depends_on:
            - zookeeper
            - broker
        ports:
            - "9021:9021"
        environment:
            CONTROL_CENTER_BOOTSTRAP_SERVERS: 'broker:29092'
            CONTROL_CENTER_ZOOKEEPER_CONNECT: 'zookeeper:32181'
            CONTROL_CENTER_REPLICATION_FACTOR: 1
            CONTROL_CENTER_INTERNAL_TOPICS_PARTITIONS: 1
            CONTROL_CENTER_MONITORING_INTERCEPTOR_TOPIC_PARTITIONS: 1
            CONFLUENT_METRICS_TOPIC_REPLICATION: 1
            PORT: 9021

# Kafka bits seen on https://github.com/confluentinc/cp-all-in-one/blob/5.5.0-post/cp-all-in-one-community/docker-compose.yml
```

Don't ask me about all these options, I just copied them from the sample Compose files 😅.

## Bringing it all together

Now that we have the core bits ready, we can go through some remaining details to get things working.

### Serializing events

One of the things we saw was needed, was to provide a serializer for the keys and values of the records we want to push to Kafka.

Ideally, we should go with something like [Apache Avro](https://avro.apache.org/) or [Protocol Buffers (protobuf)](https://developers.google.com/protocol-buffers/), but to keep it simple for now, we'll just use JSON, particularly the Newtonsoft.Json package, so we have inheritance issues figured out for us. Inheritance support is helpful because we want to publish events of different types to the same topic, and this is a way to achieve it.

To be used by the Kafka client library we're using, we need to implement the `ISerializer` and `IDeserializer` interfaces provided by it.

`CodingMilitia.PlayBall.Shared.EventBus\Serialization\JsonEventSerializer.cs`

```cs
public class JsonEventSerializer<T> : ISerializer<T>, IDeserializer<T>  where T : class
{
    private static readonly JsonSerializerSettings Settings = new JsonSerializerSettings
    {
        TypeNameHandling = TypeNameHandling.All
    };
    
    private JsonEventSerializer()
    {
    }

    public static JsonEventSerializer<T> Instance { get; } = new JsonEventSerializer<T>();

    public byte[] Serialize(T data, SerializationContext context)
        => Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(data, Settings));
    
    public T Deserialize(ReadOnlySpan<byte> data, bool isNull, SerializationContext context)
        => isNull
            ? null
            : JsonConvert.DeserializeObject<T>(Encoding.UTF8.GetString(data), Settings);
}
```

### Dependency injection

First thing, setup things in DI. In the `EventExtensions.cs`  file, added a couple of calls to helper methods defined in the EventBus library:

`CodingMilitia.PlayBall.Auth.Web\IoC\EventExtensions.cs`

```csharp
services.AddTopicDistributor<BaseAuthEvent>(new[] {typeof(BaseUserEvent)});

services.AddKafkaTopicPublisher(
    "UserAccountEvents",
    configuration.GetSection(nameof(KafkaSettings)).Get<KafkaSettings>(),
    Serializers.Utf8,
    JsonEventSerializer<BaseUserEvent>.Instance,
    @event => @event.UserId);
```

In both cases, the methods simply call the classes' constructor, passing it the given parameters, then adding them as singletons to the `IServiceCollection`.

### Using IEventPublisher

Using the `IEventPublisher` should be pretty straightforward with everything in place. In the outbox publishers we created in previous episodes, we inject an instance of `IEventPublisher<BaseAuthEvent>`, then use it where we previously had a log.

`CodingMilitia.PlayBall.Auth.Web\Infrastructure\Events\OutboxPublisher.cs`

```cs
public class OutboxPublisher
{
	// ...
    
    public OutboxPublisher(
        // ...
        IEventPublisher<BaseAuthEvent> eventPublisher)
    {
    	// ...
        _eventPublisher = eventPublisher;
    }

    public async Task PublishAsync(long messageId, CancellationToken ct)
    {
       // ...
            
        	var message = await db.Set<OutboxMessage>().FindAsync(new object[] {messageId}, ct);

            if (await TryDeleteMessageAsync(db, message, ct))
            {
                await _eventPublisher.PublishAsync(message.Event.ToBusEvent(), ct);

                await transaction.CommitAsync();
            }

        // ...
    }
    
    // ...
}
```

That `ToBusEvent` method maps the database event type to a type contained in a separate project, `CodingMilitia.PlayBall.Auth.Events`, which contains all the events that can be published by the auth service, that can be shared with other services which want the consume said events.

### Seeing it in action

Now we can run the application to see things in action. In the application we can do some action that causes an event (e.g. register a new user) then head to the Confluent Control Center, look at the topics and see what we can find there.

[![control center topic contents](/assets/2020/08/18/e045-control-center-1.png)](/assets/2020/08/18/e045-control-center-1.png)

We can see we have a message there. If we scroll to the right, we can see the contents of the message.

[![control center topic contents scrolled](/assets/2020/08/18/e045-control-center-2.png)](/assets/2020/08/18/e045-control-center-2.png)

Looking at it, we also notice that the record key matches the user id in the event, as we set things up like that to ensure the events for the same user go to the same partition.

## Outro

That does it for this episode, where we finally got events published from the auth service to Kafka. In the next episode, we'll implement the consuming end on the group management service.

In summary, in this post we looked at:

- Create an interface to abstract not only the usage of Confluent's .NET client, but other concerns that our publishing applications don't need to care
- Implement event publishing with Confluent's .NET client
- Skimmed through an overengineered way to handle multiple topics
- Start a Kafka instance locally
- Get everything working with
  - Event serialization
  - Dependency injection
  - Make use of the event publishing interface

Links in the post:

- [Apache Kafka](https://kafka.apache.org/)
- [Confluent's .NET Client for Apache Kafka](https://github.com/confluentinc/confluent-kafka-dotnet)
- [Confluent launch services repository](https://github.com/confluentinc/cp-all-in-one)
- [Apache Avro](https://avro.apache.org/)
- [Protocol Buffers (protobuf)](https://developers.google.com/protocol-buffers/)
- [Event-driven integration #4 - Outbox publisher (feat. IHostedService & Channels) [ASPF02O|E043]](https://blog.codingmilitia.com/2020/05/07/aspnet-043-from-zero-to-overkill-event-driven-integration-04-outbox-publisher-feat-ihostedservice-channels/)

The source code for this post is in the [Auth](https://github.com/AspNetCoreFromZeroToOverkill/Auth/tree/episode045) and [Shared](https://github.com/AspNetCoreFromZeroToOverkill/Shared/tree/episode045) repositories, tagged as `episode045`.

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!