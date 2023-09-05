---
author: João Antunes
date: 2023-09-05 08:10:00+01:00
lastmod: 2023-09-05 20:00:00+01:00
layout: post
title: 'Observing .NET microservices with OpenTelemetry - logs, traces and metrics'
summary: 'Observability has been one of the hot topics, with the increasing complexity of knowing what is going on in our applications. Fortunately, .NET has been investing in making things easier with OpenTelemetry.'
images:
- '/images/2023/09/05/observing-dotnet-microservices-with-opentelemetry-logs-traces-metrics.png'
categories:
- csharp
- dotnet
tags:
- opentelemetry
- observability
- logs
- traces
- metrics
- grafana
- prometheus
- kafka
- postgresql 
slug: observing-dotnet-microservices-with-opentelemetry-logs-traces-metrics
---

## Intro

Observability has been one of the hot topics over the last few years, and with good reason, as with the proliferation of microservices and related architectural patterns, knowing what’s going on in our applications, has become increasingly complex.

With this in mind, I wanted to further explore how to make our .NET applications more observable. I’ve created a [video about distributed tracing](https://youtu.be/l1_i8p2hVlE) in the past, also briefly mentioning logging, so my main goal this time was to look at metrics in particular. In any case, the post will go into all three observability topics, as I'd like to have a central reference for it all.

The focus of the post is on the .NET side of things, but we also need some external tools to gather and analyze the telemetry data we collect. I’ll talk about these tools in the upcoming “Overview” section.

See this post as more of a practical overview, as I didn't go into more fine details and concepts, but focused on getting everything going, with some sample code to experiment with.

Note that I've wrote this post, and accompanying sample code, using .NET 8 while still in preview, so even though it's unlikely things change much, it might happen that some of the things described here actually aren't exactly like this in the final release.

## Overview

To explore these observability topics, we need a sample application. As usual, we want something complex enough to see things in action, but simple enough to not be a distraction from what we’re trying to learn.

So what we have is an application composed of two services:

- An HTTP API that receives POST requests, then pushes messages to a Kafka broker
- A background service that processes messages from Kafka, then writes something in a PostgreSQL database

Particularly for tracing, we could further complicate things to make it more interesting, but as mentioned, I already did that in a [previous video](https://youtu.be/l1_i8p2hVlE), so refer to that if you’re curious 😉.

For a quick visual, this is our application:

{{< embedded-image "/images/2023/09/05/01-application-components.drawio.png" "Application components" >}}

Regarding observability, the goal is to use as much of the [OpenTelemetry](https://opentelemetry.io) available tooling as possible. In the specific case of the .NET services developed, this means using both the built-in facilities that integrate with OpenTelemetry (e.g. using APIs from [System.Diagnostics](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics?view=net-7.0), like the [Activity](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.activity?view=net-7.0) class, to instrument the code), as well as the available NuGet packages (e.g. [OpenTelemetry.Exporter.OpenTelemetryProtocol](https://www.nuget.org/packages/OpenTelemetry.Exporter.OpenTelemetryProtocol/1.4.0) to export the collected information).

Outside of the application code, and as mentioned in the intro, we also need some external tools to gather and analyze the data we collect from the applications. For this task, we’ll use:

- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/), a vendor agnostic way to forward data from our applications to the target systems that’ll store them and provide analysis capabilities
- [Prometheus](https://prometheus.io) to store and provide querying capabilities over the metrics
- Grafana Labs collection of tools
  - [Grafana Loki](https://grafana.com/oss/loki/) to aggregate the logs
  - [Grafana Tempo](https://grafana.com/oss/tempo/) to aggregate the traces
  - [Grafana](https://grafana.com) itself, as the UI over the metrics, traces and logs

{{< embedded-image "/images/2023/09/05/02-application-components-plus-observability-tools.drawio.png" "Application components plus observability tools" >}}

### Aside: about the choice of tools

Note that I don’t have super strong opinion about these tools, so, as usual, I take advantage of doing these experiments to try out technologies I didn’t use yet.

Going in, I had in mind using as much OpenTelemetry stuff as possible (as mentioned earlier), as well as Prometheus and Grafana, which seem like a sort of de facto standard in each one’s typical use cases, so I wanted to try them out.

As for Grafana Loki and Tempo, since I had to pick something, I thought I might as well go with something that integrated easily with the other already made choices. Besides that, and an overarching theme for all choices, I also wanted something that I could setup easily and run locally, so the sample code is as self contained as possible.

All of this to say, don’t focus too much on the tools used here, they were just an easy way to get something going. There are some managed options I'd like to try at some point (e.g. [Honeycomb](https://www.honeycomb.io)), but given the desire to keep this post's sample code easy to run after a git clone, I decided against using them at this point. The most relevant things from this post are the overall concepts, how to do things with .NET (and C# in this particular case), as well as using OpenTelemetry, so we can keep our .NET bits the same, changing only the target systems for the telemetry data.

### Aside 2: summary of the 3 types of telemetry data

Before looking at how to implement things, it's probably worth it to summarize what each of these types of telemetry data represents:

- [Logs](https://opentelemetry.io/docs/concepts/signals/logs/) are the type of telemetry data that folks are more used to. Logs are messages our applications write at specific points in their execution, so we get some information about the status of the application. These messages can be anything we want, from debug information indicating a specific method is being executed, to error information, when an exception is raised.
- [Traces](https://opentelemetry.io/docs/concepts/signals/traces/) feel a bit like an evolution of logs. Traces can contain similar information to logs, but are designed in a way that can, well, trace, as the execution of an operation flows through our application, from receiving an HTTP request, to querying a database, invoking an external service or publishing an event to messaging infrastructure.
- [Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/), I'd say, differ a bit from logs and traces, in that logs and traces are typically focused on specific executions, e.g. "this happened during this HTTP request", while metrics are more focused on aggregations and things happening over time, e.g. HTTP requests per second, request latency percentiles, average CPU usage, and so on.

## Logs

Regarding logging, we can get it out of the way rather quickly. Nothing changes in regards to using logging, we can continue to use `Microsoft.Extensions.Logging.ILogger` as usual, we just need to change a bit the configuration.

Firstly, to be able to export the logs to an OpenTelemetry compliant target, we need the `OpenTelemetry.Exporter.OpenTelemetryProtocol.Logs` package. Then, we need to, first configure logging, followed by the export itself.

The following code, shows the logging configuration for the `Worker` service.

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Logging.AddOpenTelemetry(options =>
{
    options.SetResourceBuilder(
            ResourceBuilder
                .CreateDefault()
                .AddService(
                    serviceName: "Worker",
                    serviceVersion: typeof(Program).Assembly.GetName().Version?.ToString() ?? "unknown",
                    serviceInstanceId: Environment.MachineName));

    options.IncludeScopes = true;
    options.IncludeFormattedMessage = true; 
    options.ParseStateValues = true;
    
    // TODO: configure export
});
```

As we’ll see later regarding traces and metrics as well, the OpenTelemetry SDK requires us to configure a `ResourceBuilder`, which, among other things I’m yet to find out, allows us to set some metadata that’ll be included in the exported data and, as we can see above, we can include information about the service producing said data.

After that, we can configure a bit more the behavior of the logger, such as including [scopes](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-8.0#log-scopes), including the formatted message (by default, the log message sent will not replace the placeholders with the values) or parsing the state values (i.e. parsing the properties of the state object that’s passed to the `logger.Log` method, when it isn’t a collection of key value pairs - for more info, check out the sample code, I added some comments there).

Then we need to configure the exporter:

```csharp
builder.Logging.AddOpenTelemetry(options =>
{
    // ...

	options.AddOtlpExporter(exporterOptions =>
    {
        exporterOptions.Endpoint =
            builder
                .Configuration
                .GetSection(nameof(OpenTelemetrySettings))
                .Get<OpenTelemetrySettings>()!
                .Endpoint;
    });
});
```

In a production scenario, maybe we need to improve this configuration a bit, but in this demo code, I’m simply setting the endpoint where the exporter should send the telemetry data to. As we saw in the earlier diagram, the target endpoint will be the OpenTelemetry Collector, which will eventually forward the logs to Grafana Loki, so we can see them in a central place.

Heading to Grafana, we can query Loki, to see the logs produced by the `Worker` service.

{{< embedded-image "/images/2023/09/05/03-grafana-loki-query-logs-list.png" "Grafana Loki query logs list" >}}

{{< embedded-image "/images/2023/09/05/04-grafana-loki-query-logs-detail.png" "Grafana Loki query logs detail" >}}

## Traces

Now lets look at traces. As mentioned earlier, I already did a [video on the topic](https://www.youtube.com/watch?v=l1_i8p2hVlE), so maybe it has some info I don't include here, or vice versa.

Setting up traces may be as easy as setting up logs, or a bit more complex, depending on our needs. What does this mean? Basically, we have a lot of instrumentation already done for us, so all we have to do is to configure it. For example, both ASP.NET Core and Npgsql already have things instrumented, so all we have to do is initialize things, and we’ll get a lot of details as part of the traces. However, if we use other libraries that aren’t instrumented, or want to add additional context to the traces, specific to our application, then we need to manually instrument things.

### Setting things up

Let’s start with setting things up, as usual, by configuring the services in DI, in this case, for the `Worker` service.

```csharp
builder.Services
    .AddOpenTelemetry()
    .ConfigureResource(resourceBuilder =>
    {
        resourceBuilder.AddService(
            serviceName: "Worker",
            serviceVersion: typeof(Program).Assembly.GetName().Version?.ToString() ?? "unknown",
            serviceInstanceId: Environment.MachineName)
    })
    .WithTracing(providerBuilder =>
    {
        providerBuilder
            .AddAspNetCoreInstrumentation()
            .AddNpgsql()
            .AddSource(nameof(EventConsumer)) 
            .AddOtlpExporter(options =>
            {
                options.Endpoint =
                    builder
                        .Configuration
                        .GetSection(nameof(OpenTelemetrySettings))
                        .Get<OpenTelemetrySettings>()!
                        .Endpoint;

                options.Protocol = OtlpExportProtocol.Grpc;
            });
    })
    .WithMetrics(/* we'll look at this later */);
```

So, we start by calling `AddOpenTelemetry`, so we get an `OpenTelemetryBuilder` and can start configuring things.

The function we passed to `SetResourceBuilder` should look familiar, as it’s the same code as we saw earlier, when talking about the logs, when we invoked the `SetResourceBuilder` method.

Then there’s a call to `WithTracing`, where we setup things for tracing, and  `WithMetrics`, which we’ll look at later, to setup things for metrics. `AddAspNetCoreInstrumentation` and `AddNpgsql` indicate that we want to enable and make use of the instrumentation built-in to each corresponding library. `AddSource` is how we indicate that we’ll have a custom activity source, in this case associated with our `EventConsumer` class. The Kafka .NET library doesn’t have, at the time of writing, instrumentation built-in, so we need to do it ourselves, hence this custom activity source. We’ll see how this works in a bit.

After setting up built-in instrumentation, as well as custom, we're calling `AddOtlpExporter`, which should again feel familiar, as we already setup something like it for the logs.

### Querying traces

Before discussing how to implement custom tracing, let's take a look at the Grafana UI, querying the traces exported to Grafana Tempo, starting with listing the traces for the API, with a span named `/do-stuff`, which is the span created by ASP.NET Core when we invoke the endpoint on the same route.

{{< embedded-image "/images/2023/09/05/05-grafana-tempo-query-trace-list.png" "Grafana Tempo query trace list" >}}

We can then click a trace to get an overview of it, the spans that compose it, the time it took per span and overall.

{{< embedded-image "/images/2023/09/05/06-grafana-tempo-query-trace-overview.png" "Grafana Tempo query trace overview" >}}

And then, we can drill-down on each span, to get further information. Notice how we can even see the SQL query associated with the span created by Npgsql.

{{< embedded-image "/images/2023/09/05/07-grafana-tempo-query-trace-details.png" "Grafana Tempo query trace details" >}}

Before going into the custom trace implementation, and to make it easier to understand the above screenshots, a quick reminder of some of the concepts:

- A trace is composed by one or more spans
- We can add spans to increase the level of detail we get from a trace, i.e., instead of just getting one trace with one span for the whole HTTP request, we add child spans to get details about database queries, HTTP requests made to other services, publishing events to a message broker, etc
- We can include detailed information in spans, to assist us when troubleshooting our application, e.g. the SQL sent to the database, the endpoint of an HTTP request, the id and type of a published event, etc
- Traces and spans have ids, not only to use when querying them, but also to be able to relate them, like to which trace does a span belong to or what, if any, is the parent span of another span (e.g. like the HTTP request span is the parent to the event publishing span in the example above)
- When we want a trace to span (no pun intended 😅) across multiple services, we need to make these ids flow across the service boundaries, for example, using headers when sending an HTTP request or publishing an event into our messaging infrastructure

For much more complete info, checkout OpenTelemetry's [docs on traces](https://opentelemetry.io/docs/concepts/signals/traces/).

### Custom traces

As I mentioned earlier, ASP.NET Core and Npgsql have built-in OpenTelemetry tracing, so the first and the last spans we see in the above trace, are provided just by configuring things as shown in the earlier code. The Kafka .NET client however, doesn't have this built-in support, so the second and third traces in the example, were obtained by writing some code.

There are, in experimental status, [some semantic conventions regarding tracing instrumentation of messaging systems with OpenTelemetry](https://opentelemetry.io/docs/specs/otel/trace/semantic_conventions/messaging/), but for this post, I didn't really took it much into consideration. For implementing this in a production system, I'd advise to try to follow these (hopefully they advance past experimental soon).

Now let's quickly go through the implementation, first on the publisher side, then consumer (as I've been doing so far, I'll use minimum code here in the post, for the whole picture checkout the accompanying GitHub repo).

Wrapping Kafka usage on the API side, I created an `EventPublisher` class. This class exposes the following `PublishAsync` method:

```csharp
public async Task PublishAsync(StuffHappened @event)
{
    using var activity = EventPublisherActivitySource.StartActivity(
        _kafkaSettings.Topic,
        @event);

    await _producer.ProduceAsync(
        _kafkaSettings.Topic,
            new Message<Guid, StuffHappened>
            {
                Key = @event.Id,
                Value = @event,
                Headers = EventPublisherActivitySource.EnrichHeadersWithTracingContext(
            activity,
                new Headers())
            });

    _metrics.EventPublished(_kafkaSettings.Topic);
}
```

Regarding tracing, the relevant bits here are the creation of the `Activity` (which corresponds to a span in OpenTelemetry terms) and the composition of the headers, to flow the trace information to the event consumer.

Before looking into the `EventPublisherActivitySource` class, something worth noting is the `using` statement when starting the activity. Disposing the activity, signals the end of the span, so by applying the `using` statement as we're doing here, means the span ends when the `PublishAsync` method ends.

`EventPublisherActivitySource` is a static class I created to encapsulate the trace instrumentation logic. I created it as static, because at least in the way I implemented it here, there is no need for instance members. On this note, thinking of dependency injection, unlike metrics, which we will look at later, that have an `IMeterFactory` to create meters, activities are created with an `ActivitySource`, which isn't really DI friendly, so I just instantiated it directly in `EventPublisherActivitySource`.

Let's look at the `EventPublisherActivitySource` class.

```csharp
public static class EventPublisherActivitySource
{
    private const string Name = "event publish";
    private const ActivityKind Kind = ActivityKind.Producer;
    private const string EventTopicTag = "event.topic";
    private const string EventIdTag = "event.id";
    private const string EventTypeTag = "event.type";

    private static readonly ActivitySource ActivitySource
        = new(nameof(EventPublisher));

    private static readonly TextMapPropagator Propagator
        = Propagators.DefaultTextMapPropagator;

    public static Activity? StartActivity(string topic, IEvent @event)
    {
        if (!ActivitySource.HasListeners())
        {
            return null;
        }

        return ActivitySource.StartActivity(
            name: Name,
            kind: Kind,
            tags: new KeyValuePair<string, object?>[]
            {
                new (EventTopicTag, topic),
                new (EventIdTag, @event.Id),
                new (EventTypeTag, @event.GetType().Name),
            });
    }

    public static Headers EnrichHeadersWithTracingContext(Activity? activity, Headers headers)
    {
        if (activity is null)
        {
            return headers;
        }

        var contextToInject = activity?.Context
            ?? Activity.Current?.Context
            ?? default;
        
        Propagator.Inject(
            new PropagationContext(contextToInject, Baggage.Current),
            headers,
            InjectTraceContext);

        return headers;

        static void InjectTraceContext(Headers headers, string key, string value)
            => headers.Add(key, Encoding.UTF8.GetBytes(value));
    }
}
```

We have a bunch of constants which will be used when creating the activities, an `ActivitySource`, used to create the activities, and a propagator, which we'll use to propagate the tracing context through Kafka headers. Regarding the `ActivitySource`, note that the name we pass in its constructor, is the name we need to use when configuring OpenTelemetry tracing in the DI container (earlier we saw `AddSource(nameof(EventConsumer))` because I used the worker service as an example instead of the API).

After this class initialization, we have the `StartActivity` method, which starts by checking if there are listeners for the activity source, as there's no need to run the tracing code if nobody cares. Then, it starts the activity giving it a name and kind, followed by setting some tags to help us out when troubleshooting.

Finally, we have the `EnrichHeadersWithTracingContext` method, which we use to flow the tracing information to the event consumer, through Kafka headers. The propagator we created earlier helps us implementing this logic, so we pass it the [context](https://opentelemetry.io/docs/concepts/signals/traces/#context-propagation) and [baggage](https://opentelemetry.io/docs/concepts/signals/baggage/). The `InjectTraceContext` local function acts as the glue between the generic context propagator, and Kafka headers specifics.

With things ready on the publishing side, we need to implement something similar, but in reverse, on the consumer side, where we need to extract the tracing information from the headers to create the activity.

Wrapping the Kafka usage on the consumer side, we have a class named `EventConsumer`. In this class, we find a method named `HandleConsumeResultAsync`, which looks like the following:

```csharp
private async Task HandleConsumeResultAsync(
    IConsumer<Guid, StuffHappened> consumer,
    ConsumeResult<Guid, StuffHappened> consumeResult,
    CancellationToken stoppingToken)
{
    var startingTimestamp = _metrics.EventHandlingStart(_kafkaSettings.Topic);
    
    try
    {
        using var activity = EventConsumerActivitySource.StartActivity(
            _kafkaSettings.Topic,
            consumeResult.Message.Value,
            consumeResult.Message.Headers);

        await HandleEventAsync(consumeResult.Message.Value, stoppingToken);

        consumer.Commit();
    }
    finally
    {
        _metrics.EventHandlingEnd(_kafkaSettings.Topic, startingTimestamp);
    }
}
```

Regarding tracing, activity creation is the only relevant piece of code from the snippet (the parts with `_metrics` we'll look at in the metrics section). Again, notice the activity will be disposed as the method finishes, meaning the span we'll see in Grafana is only associated with the code running in this scope.

The `EventConsumerActivitySource` class is similar to the `EventPublisherActivitySource` we just analyzed, doing mostly the inverse work: creating an activity out of the tracing context present in the Kafka message headers.

```csharp
public class EventConsumerActivitySource
{
    private const string Name = "event handle";
    private const ActivityKind Kind = ActivityKind.Consumer;
    private const string EventTopicTag = "event.topic";
    private const string EventIdTag = "event.id";
    private const string EventTypeTag = "event.type";

    private static readonly ActivitySource ActivitySource
        = new(nameof(EventConsumer));

    private static readonly TextMapPropagator Propagator
        = Propagators.DefaultTextMapPropagator;

    public static Activity? StartActivity(
        string topic,
        IEvent @event,
        Headers? headers)
    {
        if (!ActivitySource.HasListeners())
        {
            return null;
        }

        // Extract the context injected in the headers by the publisher
        var parentContext = Propagator.Extract(
            default,
            headers,
            ExtractTraceContext);

        // Inject extracted info into current context
        Baggage.Current = parentContext.Baggage;

        return ActivitySource.StartActivity(
            Name,
            Kind,
            parentContext.ActivityContext,
            tags: new KeyValuePair<string, object?>[]
            {
                new(EventTopicTag, topic),
                new(EventIdTag, @event.Id),
                new(EventTypeTag, @event.GetType().Name),
            });

        static IEnumerable<string> ExtractTraceContext(
            Headers? headers,
            string key)
        {
            if (headers is not null
                && headers.TryGetLastBytes(key, out var value)
                && value is { } bytes)
            {
                return new[] { Encoding.UTF8.GetString(bytes) };
            }

            return Enumerable.Empty<string>();
        }
    }
}
```

Won't spend much time describing this class because, as you can see, it's pretty similar to the one we saw for the publisher, using the same helpers and whatnot, just doing the inverse logic.

With all this in place, we get the traces to look like the screenshots shown earlier.

## Metrics

To wrap things up, let's look at metrics. This part took me a bit more time to get ready, not only because I was less familiar with the subject, but also because things were changing as I was trying to understand them 😅. When I started exploring things, before the first preview bits of .NET 8 were available, the approach to expose metrics was different, for example using [EventCounters](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/event-counters). However, with .NET 8, there were changes regarding how to expose the metrics, namely using the `System.Diagnostic.Metrics` APIs. The "old" way would still work, but I thought it would be more interesting to focus on the latest approach, which was developed with OpenTelemetry in mind. So I ended up letting this whole sample stew for a bit, waiting for later preview releases of .NET 8, to have a better grasp of how things would look like going forward.

> A quick note, before getting into more details, I think it's worth mentioning [dotnet-monitor](https://github.com/dotnet/dotnet-monitor), which is a tool that can be run as a sidecar of a running .NET application, extracting telemetry data plus other functionalities (e.g. process dumps). When I started exploring, some metrics seemed to be missing if I just used the available OpenTelemetry libraries, so I tried dotnet-monitor to complement the metrics exposed directly by the application. However, since then, and with the changes in .NET 8, most things I was looking at are now easy to get with just the OpenTelemetry libraries, so I ended up removing dotnet-monitor from this sample, but it still might be worth a look, depending on your needs.

### Setting things up

Regarding the amount of work we need to do to get metrics going, we're basically in the same spot as regarding traces: there's a lot of stuff built into the framework and some libraries, but for things that aren't built-in, we need to write some custom code to get it going.

Let's take a look at the base metrics configuration of the Worker project.

```csharp
builder.Services
    .AddOpenTelemetry()
    .ConfigureResource(configureResource)
    .WithTracing( /* we already looked at this */)
    .WithMetrics(metrics =>
    {
        metrics
            .AddRuntimeInstrumentation()
            .AddProcessInstrumentation()
            .AddAspNetCoreInstrumentation()
            // we're not using, but important to know it exists
            // .AddHttpClientInstrumentation()
            .AddView(
                "kafka_consumer_event_duration",
                new ExplicitBucketHistogramConfiguration
                {
                    Boundaries = new[]
                    {
                        0,
                        0.005,
                        0.01,
                        // ...
                        10
                    }
                })
            .AddMeter(
                "System.Runtime",
                "Microsoft.AspNetCore.Hosting",
                "Microsoft.AspNetCore.Server.Kestrel",
                "Npgsql",
                EventConsumerMetrics.MeterName)
            .AddOtlpExporter(/* same thing as we saw for tracing */);
    });
```

As expected, we can see that configuration is similar to tracing, which we saw earlier, in particular, we setup a bunch of built-in instrumentation we want to use, and we setup the OpenTelemetry exporter. We have a couple of slightly different things though: 

- `AddView`, which in this case is being used to define buckets for event handling duration (there's something similar in the API project as well, for HTTP request handling duration)
- `AddMeter`, where we setup the meters we want to be active and exported. As we can see, there are some built into .NET and ASP.NET Core, some from Npgsql, and finally, a custom one we define to have metrics from event handling, which aren't provided out of the box by the Kafka library

### Metrics dashboards

Before looking at implementing custom metrics, let's take a look at a couple of example dashboards we can create in Grafana with these metrics.

API dashboard:

{{< embedded-image "/images/2023/09/05/08-grafana-api-metrics-dashboard.png" "Grafana API metrics dashboard" >}}

API requests per second per instance query:

{{< embedded-image "/images/2023/09/05/09-grafana-api-metrics-inbound-requests-per-second.png" "Grafana API metrics inbound requests per second" >}}

Worker dashboard:

{{< embedded-image "/images/2023/09/05/10-grafana-worker-metrics-dashboard.png" "Grafana Worker metrics dashboard" >}}

Worker event handling duration query:

{{< embedded-image "/images/2023/09/05/11-grafana-worker-metrics-event-handling-duration.png" "Grafana Worker metrics inbound event handling duration" >}}

I won't go into detail on all of these, you can explore these dashboards by running things locally, using the accompanying Git repo. However, we can see there are a bunch of interesting metrics, like inbound requests and events per second, handling duration percentiles, plus threading, CPU and GC metrics, among others. 

> Note that this was the first time I played with metrics, as well as creating Prometheus queries to analyze them, so it's not impossible I either misinterpreted some metric, or messed some query up, so, as you should always do, don't just take the word from a random dude on the internet, do your due diligence 😉. Also, if you notice something wrong, let me know so I can try and fix it 🙂.

### Custom metrics

Now let's look at implementing some custom metrics. As you might have noticed, there are similar metrics for inbound HTTP requests and Kafka events in the above dashboards. As I mentioned earlier, ASP.NET Core comes with a bunch of metrics built-in, among them, metrics for inbound HTTP requests, while Confluent's Kafka library does not have any built-in metrics. As such, to get the metrics as we saw above, I created custom metrics for the Kafka consumer, following the same approach as ASP.NET Core.

Earlier in the post, there was a code snippet for the `HandleConsumeResultAsync` method. We didn't look at that part then, but there was already code there for the metrics:

```csharp
private async Task HandleConsumeResultAsync(
    IConsumer<Guid, StuffHappened> consumer,
    ConsumeResult<Guid, StuffHappened> consumeResult,
    CancellationToken stoppingToken)
{
    var startingTimestamp = _metrics.EventHandlingStart(_kafkaSettings.Topic);
    
    try
    {
        using var activity = EventConsumerActivitySource.StartActivity(
            _kafkaSettings.Topic,
            consumeResult.Message.Value,
            consumeResult.Message.Headers);

        await HandleEventAsync(consumeResult.Message.Value, stoppingToken);

        consumer.Commit();
    }
    finally
    {
        _metrics.EventHandlingEnd(_kafkaSettings.Topic, startingTimestamp);
    }
}
```

We want to expose a few metrics, namely:
- Duration of event handling
- Amount of handled events
- Events currently being handled (though this is not super helpful, as the Kafka consumer is naively implemented, it only consumes one message at a time)

Due to event handling duration metric. we need to save the starting timestamp, and pass it when event handling finishes, as we see in the code above.

The `_metrics` field is of the type `EventConsumerMetrics`, a class created to encapsulate the metric generation logic.

```csharp
public class EventConsumerMetrics : IDisposable
{
    public const string MeterName = "Sample.KafkaConsumer";

	private readonly TimeProvider _timeProvider;
    private readonly Meter _meter;
    private readonly UpDownCounter<long> _activeEventHandlingCounter;
    private readonly Histogram<double> _eventHandlingDuration;

    public EventConsumerMetrics(
        IMeterFactory meterFactory,
        TimeProvider timeProvider)
    {
        _timeProvider = timeProvider;
        _logger = logger;
        _meter = meterFactory.Create(MeterName);

        _activeEventHandlingCounter = _meter.CreateUpDownCounter<long>(
            "kafka_consumer_event_active",
            unit: "{event}",
            description: "Number of events currently being handled");

        _eventHandlingDuration = _meter.CreateHistogram<double>(
            "kafka_consumer_event_duration",
            unit: "s",
            description: "Measures the duration of inbound events");
    }

    public long EventHandlingStart(string topic)
    {
        if (_activeEventHandlingCounter.Enabled)
        {
            var tags = new TagList { { "topic", topic } };
            _activeEventHandlingCounter.Add(1, tags);
        }

        return _timeProvider.GetTimestamp();
    }

    public void EventHandlingEnd(string topic, long startingTimestamp)
    {
        var tags = _activeEventHandlingCounter.Enabled 
                  || _eventHandlingDuration.Enabled
            ? new TagList { { "topic", topic } }
            : default;

        if (_activeEventHandlingCounter.Enabled)
        {
            _activeEventHandlingCounter.Add(-1, tags);
        }

        if (_eventHandlingDuration.Enabled)
        {
            var elapsed = _timeProvider.GetElapsedTime(startingTimestamp);
            
            _eventHandlingDuration.Record(
                elapsed.TotalSeconds,
                tags);
        }
    }

    public void Dispose() => _meter.Dispose();
}
```

In the class constructor, we get an `IMeterFactory`, which we use to create a meter, with a known name that is used when configuring the metrics, as we saw earlier. With the meter in hand, we can create instances of metrics instruments, depending on the types of metrics we want to expose. In this case, we're creating an `UpDownCounter`, which, as the name implies, allows us to increment and decrement, representing the number of actively being handled events in this example. We also create an `Histogram`, that allows us to record arbitrary values, which in this case we're using to record time elapsed handling an event.

In the `EventHandlingStart` method, we do the usual checks for enabled meters, then increment the active event handling counter. Regarding the request duration, all we do is return the current timestamp, as we can't register any metric until we have the elapsed time.

In `EventHandlingEnd`, we decrement the active handling counter, and record the duration of request handling, using the initial timestamp.

As you might have noticed, in both methods we get the Kafka topic as a parameter, which we then use to enrich the metrics with tags. These tags are useful because they allow us to do more specific queries, for example, we can query the global average event handling duration, or we can further drill-down, and see that same average but per topic.

> It goes without saying, and I failed to mention earlier, but it's the same for logs and traces, the more info we include, the more info we have to work with, but also the more info we'll have to send over the network to the observability systems, more storage is taken and more capacity is needed to process it. So we probably don't want to include every single thing, but choose wisely.

### Querying metrics with PromQL

With these metrics created, and being exported, we can query them. In the earlier screenshots, we could already see a couple of examples, but I'll just drop here some more example queries, using the custom metrics created. These queries are written in [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/), or Prometheus Query Language.

As I mentioned earlier, I'm pretty far from being even well versed in this querying metrics topic, so take this more like an inspiration, rather than assuming this is all perfect and created by an expert, it's not the case.

To query the currently being handled events, per worker instance, we can do:

```promql
kafka_consumer_event_active{job="Worker"}
```

In which `kafka_consumer_event_active` is the metric, and `Worker` is a tag/label associated with it.

If we wanted to filter by topic, we could do:

```promql
kafka_consumer_event_active{job="Worker", topic="stuff"}
```

Making use of the `topic` tag we saw earlier.

If instead of per instance, we wanted global number of currently being handled events, we could sum it all:

```promql
sum(kafka_consumer_event_active{job="Worker"})
```

To get the number of events handled per second, per instance, we can use the count of all the durations collected, and apply an [`irate` function](https://prometheus.io/docs/prometheus/latest/querying/functions/#irate), to account for the increase over a period of time. This period is defined by the [`$__rate_interval` variable](https://grafana.com/blog/2020/09/28/new-in-grafana-7.2-__rate_interval-for-prometheus-rate-queries-that-just-work/).

```promql
irate(kafka_consumer_event_duration_count{job="Worker"}[$__rate_interval])
```

Keep in mind that, due to the way this increase over time works, we don't have exact numbers for the amount of events that were handled in a given second, but an average over that period. For example, if we considered the interval to be 30 seconds, when during 15 seconds we handled 100 events per second, and during the other 15 seconds we handled no events, the reported requests per second over that period would be 50 events per second.

As a final example, if we wanted to get the 99 percentile of the event handling duration, we could do:

```promql
histogram_quantile(0.99, sum by(le) (rate(kafka_consumer_event_duration_bucket{job="Worker"}[$__rate_interval])))
```

Where `le` is a tag where its values are the buckets we defined earlier, in the call to `AddView`.

### On non-.NET metrics

One final note on metrics. In this post, I focused on exposing .NET specific metrics, but there are more metrics you can get from your systems.

For example, if you're running your applications in Kubernetes, you can push [Kubernetes metrics](https://kubernetes.io/docs/reference/instrumentation/metrics/) into your observability systems and query them alongside the .NET ones. These can be overall Kubernetes cluster metrics, as well as pod specific ones, that can complement the ones from .NET.

## Outro

And with this, we reached the end of this post. I hope it's useful to understand how we can get started with OpenTelemetry to make our distributed .NET applications observable.

In this post, we took a look at how to:

- Configure logging to export into OpenTelemetry protocol targets
- Configure tracing and metrics, also exporting to OpenTelemetry protocol targets, making use of built-in traces and metrics, as well as creating our own telemetry data points

It took me a while to complete the sample code and the post, particularly due to starting with a .NET 7 approach and noticing things were changing in .NET 8, but it was really interesting to experiment with all of this, explore .NET, ASP.NET Core and other open source projects source code to understand how the observability bits were implemented, and also have a first look at metrics, which was something I wanted to try out for a bit now (those fancy dashboards 😍).

Given the investment the .NET team is making in observability, I expect that we'll have more to learn about it when .NET 8 is released in November, so then there will probably be better designed dashboards than the ones I came up with, so be on the lookout for that.

Finally, if you get into exploring OpenTelemetry, you'll notice that .NET, along Java, is looking like the platform with the more advanced implementation of the OpenTelemetry feature set, which means it's one of the most easily observable platforms we could work with, which is awesome! Along with other improvements coming to .NET, like NativeAOT, that could eventually help with building leaner microservices, the platform is in a great place if these are the kinds of applications you're building.

Relevant links:

- [Sample code](https://github.com/joaofbantunes/DotNetMicroservicesObservabilitySample)
- [Exploring distributed tracing with ASP NET Core 6](https://youtu.be/l1_i8p2hVlE)
- [OpenTelemetry](https://opentelemetry.io)
- [OpenTelemetry - Trace Semantic Conventions - Messaging Systems](https://opentelemetry.io/docs/specs/otel/trace/semantic_conventions/messaging/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [Prometheus](https://prometheus.io)
- [Grafana](https://grafana.com)
- [Grafana Loki](https://grafana.com/oss/loki/)
- [Grafana Tempo](https://grafana.com/oss/tempo/)
- [Honeycomb](https://www.honeycomb.io)
- [Npgsql - Tracing with OpenTelemetry](https://www.npgsql.org/doc/diagnostics/tracing.html)
- [Querying Prometheus](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Kubernetes Metrics Reference](https://kubernetes.io/docs/reference/instrumentation/metrics/)

Thanks for stopping by, cyaz! 👋

> Update 2023-09-05 20:00:00+01:00
>
> A couple of tweaks to starting activity, based on a couple of pointers from Martin Thwaites [on Mastodon](https://hachyderm.io/@Martindotnet/111012279774395865) (thanks Martin!)
>
> - Remove duplicated instance tags from activities - besides the fact they were duplicated in the activities and the resources, they should be part of the resources, as they're general attributes, not specific to an activity (making things more efficient in the process)
> - Passing the tags in when invoking ActivitySource.StartActivity instead of setting later with `SetTag`, as this allows to them to be considered for [sampling](https://opentelemetry.io/docs/concepts/sampling/) (a topic I didn't touch in this post)
