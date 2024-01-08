---
author: João Antunes
date: 2024-01-08 19:00:00+00:00
layout: post
title: A .NET parallel Kafka consumer proof of concept (feat. Akka.NET and the actor model)
summary: For some time now, I've been thinking about implementing a parallel Kafka consumer proof of concept in C#. I finally got around to it, and this post describes the process.
images:
  - /images/2024/01/08/a-dotnet-parallel-kafka-consumer-proof-of-concept-feat-akkadotnet-and-the-actor-model.png
categories:
  - csharp
  - dotnet
tags:
  - event-driven
  - kafka
slug: a-dotnet-parallel-kafka-consumer-proof-of-concept-feat-akkadotnet-and-the-actor-model
---
## Intro

If you've worked with [Apache Kafka](https://kafka.apache.org) and .NET, you're likely aware that the out of the box experience of consuming messages, using [Confluent's client library](https://github.com/confluentinc/confluent-kafka-dotnet) is geared towards processing records sequentially. If you want to process multiple records in parallel, you've got a few of options that come to mind:

1. Create multiple instances of your service, e.g. scale to multiple Kubernetes pods - works, but it's a waste of resources, and we're limited by the number of Kafka partitions
2. Create multiple instances of the Kafka consumer inside your service - less wasteful, but still not ideal, as we have multiple open connections unnecessarily, plus we're still limited by the number of partitions
3. Use a single consumer, forward the records to be processed in parallel by multiple threads - the best solution in terms of resources, but it means you now need to implement some non-trivial logic to ensure no records are lost (i.e. offset committed before it was actually processed), and that order is maintained (if that's relevant)

For some time now, I've been thinking about implementing a proof of concept that implemented option 3, not only because there aren't that many options for .NET (but there are some, more on that later), but also because it felt like an interesting problem to tackle. Well, this post is about this proof of concept, a little library I called [YakShaveFx.KafkaClientTurbocharger](https://github.com/YakShaveFx/YakShaveFx.KafkaClientTurbocharger/tree/introductory-blog-post) 😁.

This post will be mostly a high-level look at how this proof of concept was implemented, not super detailed on the code, which you're free to explore. I think it ended up not being super complex in terms of implementation, it just took a bit to figure out a good design approach, but the code itself isn't super complex (though I built on top of some solid foundations 😉).

Note that this proof of concept is based on the premise that we want to handle Kafka records as discrete events, one at a time, much like we would do with other types of messaging systems like RabbitMQ, Azure Service Bus, and others of the sort. This PoC is not targeted at processing a continuous flow of events, in a streaming fashion, which benefits from doing things differently. To know more about discrete events vs continuous flow of events, check out [this page](https://developer.confluent.io/courses/event-design/terminating-vs-non-terminating/).

## Where do I start? Enter the actor model

I imagine most, when thinking about this kind of problem - consuming multiple records, process in parallel, manage offsets, manage ordering, etc - start to consider the typical concurrency primitives and auxiliar data structures (locks, semaphores, auto reset events, concurrent dictionaries, ...) that would need to be used to implement this sort of thing. While not wrong, there are alternatives, higher level ones, that I feel are often forgotten.

Enter the [actor model](https://en.wikipedia.org/wiki/Actor_model)!

As soon as I started to think about how to implement this, the actor model was the first thing that came to mind - probably because I've been trying for years to find a scenario where I could use it 😅.

Without going into much detail, a few bullets to summarize the main aspects of the actor model that make it interesting for this kind of problem:

- An actor is the basic building block of concurrent computation
- Actors are single threaded and communicate through message passing, meaning, an actor has a mailbox - which is basically an in memory message queue - from which it processes messages sequentially
- Processing messages may cause changes to the actors internal state or side effects, like sending messages to other actors, persisting some data in a database, etc
- The only way actors communicate is through message passing, and each actor's internal state is private, i.e. if an actor needs information from another actor, it needs to get it through a message
- Everything is asynchronous, i.e. when an actor sends a message to another actor, it's not blocked waiting a reply, it can continue its work, sending more messages, or processing the next message in its own mailbox

If it makes it easier to picture, even though it's a silly analogy, imagine that an actor is like a nanoservice inside your microservice - something running autonomously, with its own private data and API to interact with others.

These simple characteristics I mentioned, make the actor model great to build highly concurrent applications, or, in this case, a library to consume Kafka records in parallel. We can start to think about what autonomous components would make sense, how they'll interact and what data they need to do their job:

- Maybe an actor to interact with Kafka
- Another actor to keep track of consumed offsets
- One actor to coordinate distribution of records to process
- And so on...

And with all this, there are no locks, no concurrent dictionaries, none of that sort of stuff - I mean, somewhere in our actor framework of choice (I used [Akka.NET](https://getakka.net) for this development) these primitives are used, but we're working at a higher level of abstraction, so that's handled for us. Plus, with this shared nothing/message passing approach, many of the typical concurrency problems simply go away.

There's much, much more to talk about the actor model (I didn't even mention the super important [location transparency](https://getakka.net/articles/concepts/location-transparency.html) concept), but hopefully this quick summary is enough to get you through the rest of the post, and maybe also to pique your interest to learn more - maybe knowing that [Halo 4 and 5 online services are powered by Orleans](https://thenewstack.io/project-orleans-the-net-framework-from-microsoft-research-used-in-halo-4/) (another .NET actor framework), or being aware of Ericsson's [impressive reliability](https://dockyard.com/blog/2018/07/18/all-for-reliability-reflections-on-the-erlang-thesis), using Erlang and the actor model, can be further motivation.

## Base system design and flow

So, we know we're going to use the actor model to design our system, because it simplifies the concurrent aspect of it, now we need to understand what are our exact needs and how to distribute them.

In summary, we need to:

- Interact with Confluent's Kafka consumer, to fetch records and commit offsets after processing them
- Manage offsets of the processed records, and decide when they should be committed
- Coordinate parallel work, to ensure a maximum degree of parallelism, as well as maintain order when required
- Actually process a record

To implement these requirements, I created an actor for each of them:

- [Client proxy](https://github.com/YakShaveFx/YakShaveFx.KafkaClientTurbocharger/blob/introductory-blog-post/src/YakShaveFx.KafkaClientTurbocharger.Core/Consumer/ClientProxying/ClientProxy.cs)
- [Offset controller](https://github.com/YakShaveFx/YakShaveFx.KafkaClientTurbocharger/blob/introductory-blog-post/src/YakShaveFx.KafkaClientTurbocharger.Core/Consumer/OffsetManagement/OffsetController.cs)
- [Parallelism coordinator](https://github.com/YakShaveFx/YakShaveFx.KafkaClientTurbocharger/blob/introductory-blog-post/src/YakShaveFx.KafkaClientTurbocharger.Core/Consumer/Running/ParallelismCoordinator.cs)
- [Runner](https://github.com/YakShaveFx/YakShaveFx.KafkaClientTurbocharger/blob/introductory-blog-post/src/YakShaveFx.KafkaClientTurbocharger.Core/Consumer/Running/Runner.cs)

Besides these 4 actors, I created a fifth one, which I named the [consumer orchestrator](https://github.com/YakShaveFx/YakShaveFx.KafkaClientTurbocharger/blob/introductory-blog-post/src/YakShaveFx.KafkaClientTurbocharger.Core/Consumer/Orchestration/ConsumerOrchestrator.cs). Actors can communicate directly without an orchestrator, but I felt like it would be simpler to manage if I centralized how the whole thing operates.

Besides introducing the orchestrator, I went with a publish/subscribe like approach, where the 4 actors mentioned earlier would receive commands to execute, and would publish events when something happened. Note that this is just a semantic thing, as these commands and events are just normal messages between actors, I simply decided to follow such a convention to make it easier to understand. Also note that this has nothing to do with the actor model, it's just the way I felt like implementing things, and the actor model can accommodate it.

To better understand the design, and how would the basic flow of messages look like, let's take a look at a diagram.

{{< embedded-image "/images/2024/01/08/01-base-flow.png" "Base flow" >}}

Let's follow the flow described in the diagram:

1. The parallelism coordinator notifies the orchestrator when it has available runners (and buffer space, we'll detail this in a bit)
2. The orchestrator requests that the client proxy fetches a record from Kafka
3. The client proxy notifies the orchestrator that there's a record available for handling
4. The orchestrator issues a couple of commands
    1. To the offset controller to keep track of the new record offset
    2. To the parallelism coordinator to handle the new record
5. Depending on the parallelism strategy (more on that later), the parallelism coordinator eventually issues a command to one of its runners to handle the inbound record
6. When handling the record is complete, the runner notifies the parallelism coordinator
7. The parallelism coordinator notifies the orchestrator that the record was handled
8. The orchestrator tells the offset controller to mark the offset of the handled record as completed
9. After checking its internal state, the offset controller eventually notifies the orchestrator that there's an offset ready to commit, which may include the just completed offset, but also others (more detail in the following section)
10. When the orchestrator is notified of an offset ready to commit, it tells client proxy to communicate it to Kafka

Remember that, even though there's a sequence of commands/events, a full sequence doesn't need to be completed before another starts. Instead, there are multiple such sequences happening at the same time, with actors processing their mailbox messages sequentially, but these messages may pertain to a different sequence, which effectively means the system, as whole, is executing the various sequences concurrently, much like CPU cores interleave work from multiple threads/processes/applications.

## Committing processed records offsets

Now that we looked at how, at a high level, the whole parallel consumer works, let's drill down on a couple of specific subjects, starting with committing processed record offsets.

At first glance, one would think there is nothing special about committing offsets, in particular if we're used to handling records sequentially: handle record, commit, handle another, commit, and so on. Besides the fact that this is not very efficient (batching commits would be better, to minimize I/O), this doesn't work when we're processing records in parallel. Or better yet, it works, but we lose the at least once delivery guarantee that we typically want.

Let's imagine that we configure our consumer with a maximum degree of parallelism of 2. We grab 2 records from Kafka, and push them onto the runners. Let's consider that both records came from the same Kafka partition, one with offset 33, and the other with offset 34. If 33 completes its processing before 34, it can be immediately committed, no worries. However, if 34 is the first to conclude processing, we can't just commit it, because we're not sure 33 will be processed successfully. If for some reason record with offset 33 fails processing, but the offset 34 was already committed, we "lost" a record. The record is still in the Kafka topic, so we didn't actually lose the data, hence the quotes, but because the last committed offset is greater than 33, it will never be processed again, so it's basically the same as lost.

{{< embedded-image "/images/2024/01/08/02-incorrect-commit.png" "Incorrect offset commit" >}}

This is where the offset controller actor comes it. Every new record we start handling, we ask this actor to keep track of it. When we complete handling a record, we again inform the offset controller. When there are offsets ready to commit, i.e. all records with offsets until a certain point were handled successfully, the offset controller notifies the orchestrator, which in turn will forward that information to the client proxy.

Continuing on the previous example, but applying the offset controller logic, the flow would be:

1. Reads records from offsets 33 and 34
2. Starts processing record with offset 33
3. Starts processing record with offset 34
4. Completes processing record with offset 34, but holds it, no commit yet
5. Completes processing record with offset 33, can now commit everything until 34

{{< embedded-image "/images/2024/01/08/03-correct-commit.png" "Correct offset commit" >}}

With this approach, we can achieve at least once delivery, and avoid losing data. Remember though that at least once means exactly that, **at least once**, meaning the same record can end up being processed more than once, so we need to make sure our record handlers are idempotent, but that has always been the case, we just increased a bit the likelihood of it happening.

Before wrapping up this commit offsets section, just a quick note on the subject of committing immediately. As I briefly alluded, committing every time a record is processed successfully is not very efficient, as we end up doing multiple network round trips to Kafka. Ideally we should batch commits, to minimize these round trips.

The client proxy actor implements batching of commits. As the actor receives offsets to commit, it stores them in its internal state. The actual commit of the offsets is triggered in one of two ways:

- When the amount of stored offsets reaches a threshold, a commit is done
- Every few seconds, even if the threshold isn't reached, any amount of stored offsets is committed, to avoid having commits waiting forever, in lower volume situations

Confluent's Kafka consumer has some features to help implementing this, but I had already implemented things manually when I discovered 😅. Maybe something to revisit later.

## Coordinating parallel record handling

Next up is coordinating how records are handled in parallel. By default, in Kafka, as alluded in the intro, the unit of parallelism is the partition, which normally means that we can handle in parallel as many records as the number of partitions. With this PoC that limitation is removed, and we can manage the parallelism however we want: want to still parallelize per partition but without the need of multiple consumer instances? Fine! Don't care about order and everything can run in parallel? Also fine.

With this in mind, the parallelism coordinator actor was developed in such a way, that we can configure a parallelism strategy, in order to meet the needs of a particular application. There are 3 parallelism strategies implemented:

- Per partition - records from different partitions may be handled in parallel, but records from the same partition will be handled sequentially
- Per key - records with different keys may be handled in parallel, but records with the same key will be handled sequentially
- [Leeroy Jenkins](https://www.youtube.com/watch?v=hooKVstzbz0) - records may be handled in parallel without any restrictions

Let's take the per key strategy as an example, to see how parallelism coordinator would behave:

{{< embedded-image "/images/2024/01/08/04-per-key-parallelism-coordination.png" "Per key parallelism coordination" >}}

1. We get a new record `x` with key `k1`
2. No other record is being handled, so we can grab any runner and tell it to handle `x`
3. We get a new record `y` with key `k2`
4. No other record with the same key is being handled, so we can grab an available runner and tell it to handle `y`
5. We get a new record `z` with key `k1`
6. There's already a record with the key `k1` being handled, so we send `z` to the same runner, for the records to be handled sequentially

As for the other parallelism strategies, the per partition strategy is pretty much the same, but instead of using the record key to decide how to distribute a record, it uses the partition, while the Leeroy Jenkins strategy ignores all of this, and always distributes new records to whatever runners are idle.

And this is the big picture of how the parallelism coordination works. There's a couple of other relevant tidbits going on, namely:

- The coordinator only accepts new records when there are available runners, so, for example, if we have a maximum degree of parallelism of 2, and the 2 runners are already handling records, the coordinator doesn't accept new records, which also means it doesn't ask the orchestrator for them in the first place
- To minimize potential problems when there are a lot of records to handle sequentially (either due to the same key or partition, depending on parallelism strategy), there's also a limit to the amount of overall records that can be kept in memory besides the ones being handled
  - This means that, taking the per key strategy as an example, if some keys appear repeatedly, we can end up with idle runners, wasting parallelism potential

## Handling errors

One important thing about the actor model, that I didn't mention in the section I briefly described it, is the concept of a hierarchy of actors, in which an actor can spawn child actors and is then responsible for [supervising them](https://getakka.net/articles/concepts/supervision.html). This goes hand in hand with error handling.

If a child error faces an error, i.e. throws an exception and doesn't handle it, the error will be propagated to its parent, which will then need to decide what to do, from the following options (in the context of Akka.NET):

- Resume the child - keep everything as is, don't clear any state, just try to continue
- Restart the child - recreate the child actor, clearing its internal state (though the mailbox is preserved), then try to continue
- Stop the child - completely stop the child actor
- Escalate - pass on the responsibility of handling the error to its own parent (so, the failed child actor's grand parent), also meaning that the actor is also failed

Looking at this PoC, we have the following actor hierarchy:

- Consumer orchestrator is the root of the developed actors hierarchy
  - Client proxy is a child of the orchestrator
  - Offset controller is a child of the orchestrator
  - Parallelism coordinator is a child of the orchestrator
    - The runners are children of the parallelism coordinator

Because in terms of an actor system, this proof of concept is rather small and focused on a small problem domain, I decided that if any unhandled error occurred, the actors would escalate all the way up to root, and the whole system should be restarted, to be sure that no data could be lost.

Also, due to the way the work is spread out among these actors, trying to recover in a different way would be more complex than it's worth, so this supervision approach fits like a glove. If this set of actors was part of a larger actor system, we could implement something similar, though we would only restart a particular branch of the hierarchy, leaving other unaffected actors alone.

Note that, from the point of view of this PoC, the responsibility to handle record processing errors lies with the users of the library, implementing retries, dead letter topics or whatever strategies they want. So, from the library's point of view, if an unexpected error occurs, anywhere in the hierarchy, the safest route is to just blow everything up and try again later.

To implement this restart everything approach, I actually added another actor into the mix, as parent of the consumer orchestrator, a backoff supervisor actor, though this one is [built-in to Akka.NET](https://getakka.net/articles/concepts/supervision.html#delayed-restarts-with-the-backoffsupervisor-pattern). With this actor in place, I configured other actors with children, namely the orchestrator and the parallelism coordinator, to escalate any errors their children encounter, causing the backoff supervisor to restart the whole thing.

{{< embedded-image "/images/2024/01/08/05-error-handling-with-supervision-hierarchy.png" "Error handling with supervision hierarchy" >}}

## Cons of using Kafka like this

If by now you're not questioning the complexity of implementing all of this, versus using something other than Kafka for discrete events, maybe you should, cause I certainly do 😅.

The fact is, there are other message brokers tailored for exactly this sort of use case, like [RabbitMQ](https://www.rabbitmq.com), [ActiveMQ](https://activemq.apache.org), [Azure Event Grid](https://learn.microsoft.com/en-us/azure/event-grid/overview), [Azure Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview), [AWS EventBridge](https://docs.aws.amazon.com/eventbridge/), just to name a few. With any of these, we get message handling parallelism out of the box, without jumping through hoops, or using some library that does it for us. Besides, these brokers also tend to have additional out of the box features (e.g. retries, dead letters, message filtering, ...) that Kafka doesn't provide (again, we'd need to build them or use a library that does it for us).

Regardless of there being options better tailored for discrete event distribution, there can be valid reasons to want to use Kafka like this, for example because it's needed for other scenarios, where we actually want to do stream processing, and instead of further complicating infrastructure, and having folks support multiple brokers, Kafka is used for the various purposes.

Besides the inherent complexity we witnessed throughout the post, another con of using Kafka like this, when compared to just using an alternative message broker, is the way we commit offsets. Given that committing the offsets may have some delay, to ensure at least once delivery, this may cause that messages are reprocessed more often. However, assuming relatively consistent event handling timings, good error handling strategies, and idempotency correctly implemented, this shouldn't be a big deal (plus, good error handling strategies and idempotency were always required anyway 😉).

In any case, I'd say, when possible, it’s probably better to use the best tool for the job, meaning, use the message broker that better fits your application’s patterns. However, if you prefer to focus your efforts on Kafka, you should also be just fine, as long as you're ok with the potential added complexity.

## Will this ever be a real library? Any alternatives?

Will this proof of concept ever be a real library? Dunno ¯\_(ツ)_/¯

This proof of concept was motivated mainly by the parallel processing problem, which seemed something interesting to experiment with, plus a good opportunity to play with the actor model, as well as learn more about Apache Kafka. Regarding these objectives, I'm happy with the result 🙂.

On my own personal time, it's unlikely that I'll continue to work on this, as the fun part is done, and there's always more interesting stuff to learn and explore.

If, however, I need something like this for work, and can't find an alternative that works for our use cases, then I might revisit this and build upon this core.

So, what about alternatives?

There's a couple I'd start with:

- If you're looking for something for discrete event handling, like we've been talking in this post, [KafkaFlow](https://github.com/Farfetch/kafkaflow) looks like an interesting option
- If you're actually more interested in stream processing, and would appreciate a declarative approach to it, take a look at [Akka.NET Streams Kafka](https://github.com/akkadotnet/Akka.Streams.Kafka)

There are also other libraries that offer a broader feature set, other than just message handling, and also seem to have some support for Kafka, like [MassTransit](https://masstransit.io) or [Brighter](https://www.goparamore.io) (haven't tried them though).

If you have a more complex system in mind and would benefit from a highly concurrent, stateful approach, then I'd suggest to take a closer at the actor model, and the multiple available options. As mentioned, for this PoC I went with [Akka.NET](https://getakka.net), but [Proto.Actor](https://proto.actor) and [Microsoft Orleans](https://github.com/dotnet/orleans) are also interesting options, each with different approaches to similar concepts, so one might be a better fit for a given project or team than the other.

## Outro

That's a wrap for this post. Hope there was something interesting in here worth reading 😅. I'm sure there are a lot of improvements worth implementing in this turbocharged Kafka consumer proof of concept, but I'm happy with the result and the learnings that came with it.

In this post, we looked at:

- Why would it make sense to implement record handling parallelization with Kafka
- Took a brief look at the actor model and how it could help us in this scenario
- The overall design of this proof of concept
- How committing offsets for processed records is addressed
- How to implement parallel processing coordination, with record ordering in mind
- The error handling approach adopted
- The cons of using Kafka this way
- Some existing options for optimizing usage of Kafka and create more efficient applications

Not that it's super interesting, but if you want a quick look at the proof of concept in action, when compared to the out of the box experience with just the Confluent Kafka consumer, take a look at the brief video that follows.

{{< youtube 47oBBMqqm_I >}}

In this demo, the applications are consuming 1000 records from Kafka. For each record handled, there's a 100ms sleep. The turbocharged consumer is configured to use the per key parallelism strategy, with a maximum degree of parallelism of 10. You can check out both the samples and the rest of the source code in the [GitHub repo](https://github.com/YakShaveFx/YakShaveFx.KafkaClientTurbocharger/tree/introductory-blog-post).

Relevant links:

- [YakShaveFx.KafkaClientTurbocharger](https://github.com/YakShaveFx/YakShaveFx.KafkaClientTurbocharger/tree/introductory-blog-post) (the source code for this proof of concept)
- [Apache Kafka](https://kafka.apache.org)
- [confluent-kafka-dotnet](https://github.com/confluentinc/confluent-kafka-dotnet)
- [Modeling As Discrete vs. Continuous Event Flows](https://developer.confluent.io/courses/event-design/terminating-vs-non-terminating/)
- [Actor model](https://en.wikipedia.org/wiki/Actor_model)
- [Erlang](https://en.wikipedia.org/wiki/Erlang_(programming_language))
- [Akka.NET](https://getakka.net)
- [Proto.Actor](https://proto.actor)
- [Microsoft Orleans](https://github.com/dotnet/orleans)
- [KafkaFlow](https://github.com/Farfetch/kafkaflow)
- [Akka.NET Streams Kafka](https://github.com/akkadotnet/Akka.Streams.Kafka)
- [MassTransit](https://masstransit.io)
- [Brighter](https://www.goparamore.io)
- [Project Orleans, the .NET Framework from Microsoft Research Used for Halo 4](https://thenewstack.io/project-orleans-the-net-framework-from-microsoft-research-used-in-halo-4/)
- [All For Reliability: Reflections on the Erlang Thesis](https://dockyard.com/blog/2018/07/18/all-for-reliability-reflections-on-the-erlang-thesis)

Thanks for stopping by, cyaz! 👋