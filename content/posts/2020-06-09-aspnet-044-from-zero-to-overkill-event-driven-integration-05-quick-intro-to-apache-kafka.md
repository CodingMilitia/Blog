---
author: João Antunes
date: 2020-06-09 18:30:00+01:00
layout: post
title: "Event-driven integration #5 - Quick intro to Apache Kafka [ASPF02O|E044]"
summary: "In this episode, we briefly introduce Apache Kafka, as we'll use it to implement our event bus. We'll focus on our specific use case, as Kafka can be used in a variety of scenarios. We'll keep it in a developer's perspective, not going much into more infrastructural subjects."
images:
- '/assets/2020/06/09/aspnet-core-from-zero-to-overkill-e044.jpg'
categories:
- fromzerotooverkill
tags:
- kafka
- events
- messaging
- queues
slug: aspnet-044-from-zero-to-overkill-event-driven-integration-05-quick-intro-to-apache-kafka
---

In this episode, we briefly introduce Apache Kafka, as we'll use it to implement our event bus. We'll focus on our specific use case, as Kafka can be used in a variety of scenarios. We'll keep it in a developer's perspective, not going much into more infrastructural subjects.

**Note:** depending on your preference, you can check out the following video, otherwise, skip to the written version below.

{{< yt tUzCxZdKEr4 >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

In this episode, we take a short break from code to introduce [Apache Kafka](https://kafka.apache.org/), the technology we'll use as the base to implement our event bus.

Kafka's website (and [intro page](https://kafka.apache.org/intro)) are the best places to get started, but I'll try to summarize the main concepts we should be aware when developing applications with it, focusing on PlayBall's current use case of implementing publish-subscribe between services (Kafka can be used in more diverse scenarios).

### Why Kafka?

You may be asking, why use Kafka instead of something like [RabbitMQ](https://www.rabbitmq.com/), which is more common in scenarios like the application we've been developing?

The reason is very simple: because I wanted to try it out 🙂.

As many things in this series, I don't just want to write and share some approaches, I also want to keep learning and try new stuff. I already used RabbitMQ (and other queuing systems) in the past, so I wanted to try Kafka.

That doesn't mean we won't try other things, namely RabbitMQ, as Kafka is not a traditional queuing system, so it may make sense to introduce a different kind of technology for other types of messaging requirements.

## In a nutshell

The best way to start is probably with a copy-paste from [Kafka's intro page](https://kafka.apache.org/intro):

> ### Apache Kafka® is *a distributed streaming platform*. What exactly does that mean?
>
> A streaming platform has three key capabilities:
>
> - Publish and subscribe to streams of records, similar to a message queue or enterprise messaging system.
> - Store streams of records in a fault-tolerant durable way.
> - Process streams of records as they occur.
>
> Kafka is generally used for two broad classes of applications:
>
> - Building real-time streaming data pipelines that reliably get data between systems or applications
> - Building real-time streaming applications that transform or react to the streams of data  

I think the most important takeaway for now, is how Kafka is introduced, a "distributed streaming platform", not as a message queue, which it is not - and I'll probably repeat this a bunch of times through this post, as it's an important distinction.

## Main concepts

Let's dive right into some Kafka concepts, focusing on the most important ones for our specific needs, from a developer point of view, not worrying with subjects more related to infrastructure.

We'll look at:

- Records
- Topics & partitions
- Producers
- Consumers & consumer groups

### Records

Records are the things we store in Kafka, so, what in other systems we would call messages.

A record is comprised of a key, value and timestamp.

The value is where the payload is stored. In our use case of sending events, the payload will be a serialized representation of an event. Kafka doesn't care about the format, so it can be XML, JSON, [Apache Avro](https://avro.apache.org/) (very commonly used with Kafka), [Protocol Buffers (protobuf)](https://developers.google.com/protocol-buffers/), or whatever else comes to mind. It's common to use binary formats such as Avro or ProtoBuf for their performance benefits when compared to text based formats.

The timestamp usage should be self-explanatory, and as for the key, we'll see a use case for it when we look at producers.

### Topics & partitions

A topic is the logical representation of a stream of related records. Taking our PlayBall use case again as an example, where we'll want to send events related to changes in user accounts, we could have a topic named something like "user account changes", where the auth service would "publish events" (or being more correct, store records), while the group management service would "consume events" (read the records).

Topics are composed of one or more partitions, where the records are stored. A partition behaves like a append-only log, inserting new records at the end. This means each partition is an ordered, immutable sequence of records. When added to the partition, the records can be identified by an offset.

[![topics and partitions](/assets/2020/06/09/044-0-topics-and-partitions.png)](/assets/2020/06/09/044-0-topics-and-partitions.png)

We'll revisit this part, but I'd say it's this log like behavior that really differentiates Kafka from queuing systems. Records are kept in the partitions according to a retention policy, which in the extreme case can be forever. Unlike queuing systems, records are not removed after being read.

Cleaning up old records is a way to save storage space and not a performance optimization, given the append-only log behavior makes Kafka's performance independent of what's already stored.

Quick side note, remembering that Kafka is itself distributed, so partitions can be spread across multiple running instances of Kafka (brokers) that are part of the same cluster, even if belonging to the same topic.

### Producers

Simply put, producers are responsible for publishing data to topics. In our PlayBall example, the producer will be implemented in auth service.

Producers have a bit more responsibility than it seems at first glance, as choosing the partition in which a record should be stored is on them.

Choosing the partition can be as simple as round-robin, but it can also be used more strategically. As an example, we can route the records based on their key. Knowing that a partition is an "ordered, immutable sequence of records", when using this to distribute events, we can use this to enforce the order of events based on a key.

Going back to the this series' example, when the auth service publishes an account related event, it can use the account id as the record key. This way, the events related to a user account are guaranteed to be ordered in a Kafka partition, as long as they're published in order. Given the way we implemented the outbox publisher in the [previous episode](https://blog.codingmilitia.com/2020/05/07/aspnet-043-from-zero-to-overkill-event-driven-integration-04-outbox-publisher-feat-ihostedservice-channels/), we don't have that guarantee, but we could improve the solution to achieve it.

**Side note:** when working with messaging systems, we should ideally avoid relying on message ordering, as it adds complexity, but sometimes it may be needed.

### Consumers & consumer groups

Consumers, as expected, consume the records 🙃. They're grouped together using a name, so that each record published to a topic is read by a single consumer in a group.

Mapping this to our usual example, our group management service could label its consumer group as "group management", then imagining we have multiple instances of the service running, a record would be read by just one of them. If we had another service interested in the auth service's events, said service would use a different group name, so it would also read the same record. 

**Side note:** hope using the word "group" in regards to the group management service and Kafka consumer groups doesn't make the example too confusing 😐.

[![topics and consumer groups](/assets/2020/06/09/044-1-topics-and-consumer-groups.png)](/assets/2020/06/09/044-1-topics-and-consumer-groups.png)

Within a consumer group, each consumer is exclusively responsible for one or more partitions, so records from one partition always go to the same consumer instance. This is nice to take advantage of the record order mentioned earlier, as we won't have consumers concurrently handling related records (as long as we used a strategy to ensure the related records are put in the same partition).

[![partitions and consumer groups](/assets/2020/06/09/044-2-partitions-and-consumer-groups.png)](/assets/2020/06/09/044-2-partitions-and-consumer-groups.png)

Consumers keep track of what they already read by storing the partition offset mentioned earlier. The consumers keep this information in memory while processing records, but should also commit it back to Kafka from time to time, so if the consumer instance goes down, itself when coming back up, or another instance of the same consumer group (after a partition rebalancing), can pickup where it left of.

## Consolidating the differences to traditional queuing systems

I've mentioned some differences between Kafka and queuing systems along the way, but I'll use this section to consolidate the core ideas that differentiate these two types of systems.

Before anything else, to reiterate, Kafka **IS NOT** a queuing system. We can, however, implement some patterns traditionally used with queuing systems on top of it, but we need to be aware of the underlying differences.

As we saw, the main difference between Kafka and a queuing system, is that Kafka stores data as an append-only log, accessed with the usage of an offset, while queuing systems push messages to consumers, removing them after getting an acknowledgement of its successfully processing.

[![message queue](/assets/2020/06/09/044-3-message-queue.png)](/assets/2020/06/09/044-3-message-queue.png)

### Use cases

This log vs queue difference is so fundamental, that it greatly affects the way these systems are used, which means they can't really be used interchangeably, it depends on the use case.

For example, if we want to implement an event bus, as is our current task in the series, we can use both types of systems, adapting the underlying implementation.

In Kafka we'd have a topic where we publish a specific category of events, having the consumers/consumer groups manage event access using the offset. On the other hand, if we used something like RabbitMQ, we would publish messages to an exchange, that would then fanout to multiple queues, one per consumer type (check out the docs for more info about [RabbitMQ and AMQP protocol](https://www.rabbitmq.com/tutorials/amqp-concepts.html)).

Events are an interesting thing to implement on top of Kafka, again, because of its log like behavior. Remembering that extreme scenario where the retention policy for Kafka records is forever, we could at any moment create a new service, and it would be able to consume all events ever published (not sure it's a good idea, but it's possible 😛).

Another common pattern to implement on top of a message queue is the sending of commands (e.g. send an email). Unlike events, from which we can extract some value from being able to replay them, commands don't really benefit from that.

Events represent something that already happened, a fact that can't be changed, so storing them in an immutable journal makes sense (hence the existence of [event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)). Commands on the other hand, are not facts, but instead a request to perform some action. This conceptual difference makes using Kafka for commands, even if possible, not very sensible.

Although I'm talking a lot about events, even making a quick reference to event sourcing, keep in mind that Kafka is also not really suitable as an event store, at least not in the same way as something like the [Event Store](https://eventstore.com/) project, missing concepts like projections or concurrency control for events targeting the same entity.

Outside of this more common message based scenarios, Kafka is also suitable for streaming scenarios, where the is a concept of stream processor, which takes a continuous flow of data, does something with it and outputs itself a flow of data. This is outside the scope of this article, but you can read more about it in the [Kafka Streams documentation](https://kafka.apache.org/documentation/streams/).

Another possible scenario, also outside the scope of this article, is the usage of [Kafka connectors](https://kafka.apache.org/documentation.html#connect), which are useful to move large collections of data from/to existing systems. As an example (mentioned in the docs), we could use a connector capture all changes to a table in a relational database.

### Other properties

To wrap up this section, some other interesting properties that differentiate these kinds of systems:

- Due to the log behavior and the consumers keeping their state with the offset, Kafka keeps a single copy of each record (ignoring copies related to redundancy), while if we used something like RabbitMQ, the message would be copied to the queue of all interested subscribers.
- Due to the parallelism when reading from a topic being explicitly represented in the form of partitions, Kafka can offer the previously mentioned ordering guarantees. Using RabbitMQ again as an example, if we have multiple consumer instances for the same queue, order is not guaranteed. Maybe there are ways to play with exchanges and queues to achieve a similar behavior, but it isn't native.

Hopefully this (not so thorough) comparison between Kafka and traditional message queue systems, sheds some light not only on this specific subject, but also if you at some point in time wondered why cloud providers have a bunch of different messaging infrastructure options. For the case of Azure, there's a nice article describing the [different messaging options](https://azure.microsoft.com/en-us/blog/events-data-points-and-messages-choosing-the-right-azure-messaging-service-for-your-data/), which can be a great complement to this post, even if not targeting the exact same technologies.

## Outro

That's about it for this episode. If you didn't know much about Kafka, hope this was a good enough intro, but remember there are much more details, I just went through the basics that will be helpful as we use it to implement our event bus.

Some quick takeaways on what we looked at:

- Kafka can be thought of as a log, where producers append records to a topic/partition
- Records are accessed with an offset within a topic's partition
- Consumers do not take records off the topic, they read them, then advance their offset
- Kafka is **not** a message queue (and also not an event store)

As this post was very developer focused, I left out important infrastructure subjects like, for example, [ZooKeeper](https://zookeeper.apache.org/), which has responsibilities in managing the Kafka cluster. Be sure to investigate much more than just read this post if you're considering using Kafka in production 🙂.

Links in the post:

- [Apache Kafka](https://kafka.apache.org/)
- [RabbitMQ](https://www.rabbitmq.com/)
- [Apache Avro](https://avro.apache.org/)
- [Protocol Buffers (protobuf)](https://developers.google.com/protocol-buffers/)
- [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Event Store](https://eventstore.com/)
- [Kafka Streams](https://kafka.apache.org/documentation/streams/)
- [Kafka Connect](https://kafka.apache.org/documentation.html#connect)
- [Events, Data Points, and Messages - Choosing the right Azure messaging service for your data](https://azure.microsoft.com/en-us/blog/events-data-points-and-messages-choosing-the-right-azure-messaging-service-for-your-data/)
- [Apache ZooKeeper](https://zookeeper.apache.org/)
- [Event-driven integration #4 - Outbox publisher (feat. IHostedService & Channels) [ASPF02O|E043]](https://blog.codingmilitia.com/2020/05/07/aspnet-043-from-zero-to-overkill-event-driven-integration-04-outbox-publisher-feat-ihostedservice-channels/)

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!