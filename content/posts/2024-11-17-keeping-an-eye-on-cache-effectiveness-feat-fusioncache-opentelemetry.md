---
author: João Antunes
date: 2024-11-17 18:15:00+00:00
layout: post
title: Keeping an eye on cache hit ratio (feat. FusionCache, OpenTelemetry & friends)
summary: In this post, we'll take a very quick look at how to use the awesome FusionCache library, and how to observe its impact on our services.
images:
  - /images/2024/11/17/keeping-an-eye-on-cache-effectiveness-feat-fusioncache-opentelemetry.png
categories:
  - csharp
  - dotnet
tags:
  - caching
  - fusioncache
  - opentelemetry
slug: keeping-an-eye-on-cache-effectiveness-feat-fusioncache-opentelemetry
---

## Intro

At work, we're currently building a service which requires the lowest latency we can provide, as it'll be on the critical path of several features.

Fortunately, the service itself isn't very complex, and the data required to fulfill its role is on the smaller side, while also not changing all that often, so a simple memory cache with conservative expiration times should be more than fine for the current needs, sparing us from a more complex cache eviction design.

While the out of the box [ASP.NET Core memory cache](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/memory?view=aspnetcore-8.0) would suffice given our modest requirements, I got thinking that a couple of extra features could be nice, like:

- [Stampede protection](https://en.wikipedia.org/wiki/Cache_stampede) - we have some scenarios where there might be a burst of requests that require the same data to fulfill, so if we could avoid hitting the database with the same query multiple times in parallel, that would be sweet
- Eager refresh - it would be ideal if the data was kept in cache as long as it's being used, instead of expiring in the middle of it, forcing a database query and the client to wait for it. We could also simply use a sliding window approach, where each time the data is fetched from cache, its duration is extended, but that could exacerbate issues of stale data
- Observability (with OpenTelemetry) - while we expect that introducing this cache will have a very positive impact on the service, it's always better to actually know that things are working as we think they are

As the out of the box memory cache doesn't support these extra features (at the time of writing, at least), I was thinking we'd have to build them ourselves on top of it, because I wasn't aware of an alternative that implemented it. That was until I came across [FusionCache](https://github.com/ZiggyCreatures/FusionCache) 🙂.

In this post, we'll take a very quick look at how to use FusionCache, and how to observe its impact on our services.

## Enter FusionCache

Fortunately, while researching a bit more how to implement the caching features described earlier, I came across [FusionCache](https://github.com/ZiggyCreatures/FusionCache), and I guess I no longer need to implement them, cause FusionCache has them all, plus a bunch more!

- Stampede protection - check ✅ ([docs](https://github.com/ZiggyCreatures/FusionCache/blob/main/docs/CacheStampede.md))
- Eager refresh - check ✅ ([docs](https://github.com/ZiggyCreatures/FusionCache/blob/main/docs/EagerRefresh.md))
- OpenTelemetry support - check ✅ ([docs](https://github.com/ZiggyCreatures/FusionCache/blob/main/docs/OpenTelemetry.md))

It even supports a two level approach ([docs](https://github.com/ZiggyCreatures/FusionCache/blob/main/docs/CacheLevels.md)), with first level in memory and a second level in a distributed cache (such as Redis or one of its 30 forks that are popping up around these times). Right now in memory will be good enough for us, but it's nice to know that if we need to go distributed, it'll require very little effort.

Configuring and using FusionCache is pretty straightforward. There are a bunch of options, but for now I kept it very simple.

> The following code excerpts are from a quick sample API I created to go along with this post, which you can check out [here](https://github.com/joaofbantunes/FusionCacheOpenTelemetrySample)

```csharp
builder.Services
    .AddFusionCache("SampleCache")
    .WithOptions(o =>
    {
        // nothing to tweak here right now,
        // but there are a bunch of interesting options to play with
    })
    // if we didn't provide anything,
    // it would create one, but I wanted to limit the cache size
    .WithMemoryCache(new MemoryCache(new MemoryCacheOptions
    {
        SizeLimit = 20
    }));
```

A builder pattern is used to configure things, and there are more extension methods available, though as I mentioned, I kept things relatively simple.

As we can see, we can provide a name (it's not mandatory though), to have multiple caches in the same application. We can also tweak a bunch of options in `WithOptions`, and we can provide a specific `IMemoryCache` instance using `WithMemoryCache`, which I did to limit the size of the cache (of course 20 is a bit low, just for demo).

Then, using the cache is also pretty straightforward, as we can see in the following endpoint example:

```csharp
app.MapGet("/{id}", async (string id, IFusionCacheProvider cacheProvider) =>
{
    var cache = cacheProvider.GetCache("SampleCache");
    var value = await cache.GetOrSetAsync(
        "root_" + id,
        async ct =>
        {
            await Task.Delay(TimeSpan.FromSeconds(1), ct);
            return Guid.NewGuid(); 
        },
        new FusionCacheEntryOptions
        {
            Duration = TimeSpan.FromSeconds(3),
            Size = 1,
            EagerRefreshThreshold = 0.5f
        });
  return value;  
});
```

If we weren't using a named cache, we could simply inject an `IFusionCache` dependency, but as I gave the cache a name, we need to instead inject an `IFusionCacheProvider` and then request a cache with a given name.

In any case, when we have a cache instance in hand, we can start using it. In this example, we're using `GetOrSetAsync`, where we provide a key, a factory lambda to be invoked to compute the value if it's not available (or is up for refreshing), as well as entry options, where we have:

- `Duration` - how long should the entry remain in cache
- `Size` - to calculate how many entries can be in cache. If you recall, I had set a limit of size 20, so, with a size of 1, we should be able to have 20 entries cached
- `EagerRefreshThreshold` - a percentage value, indicating at what point in an entry's lifetime should a query trigger a background refresh. In this case, if one and a half seconds have elapsed since the original set/refresh, a background refresh will be triggered

There's a bunch more to learn about FusionCache, but it's besides the point of this post (which I'm trying to keep small 😅), so be sure to head to its [GitHub repo](https://github.com/ZiggyCreatures/FusionCache/) and explore!

## Observing cache effectiveness

To see if our cache is having the positive effect we expect it to, we have some tools at our disposal.

### Checking response times

The most obvious way to check the positive effect, particularly if we're adding cache to a service that was already running without it, is checking if the response times have been reduced. One way to check our response times, is using ASP.NET Core metrics for request durations. Not going to detail that here though, as I've previously written a [comprehensive post]({{< ref "2023-09-05-observing-dotnet-microservices-with-opentelemetry-logs-traces-metrics" >}}) and [sample](https://github.com/joaofbantunes/DotNetMicroservicesObservabilitySample), where these metrics are used.

### Analyzing traces

Another interesting tool to see how the cache affects our service, are the traces captured (and exported in OpenTelemetry format), particularly as FusionCache plays well with them. To see FusionCache details in the traces, we just to need add a line to our tracing configuration:

```csharp
builder.Services.AddOpenTelemetry()  
    .ConfigureResource(ConfigureResource)  
    .WithTracing(traceBuilder => traceBuilder  
        .AddAspNetCoreInstrumentation()  
        .AddFusionCacheInstrumentation(o => o.IncludeMemoryLevel = true)  
        .AddOtlpExporter(options => options.Endpoint = new Uri("http://localhost:4317")));
```

Calling `AddFusionCacheInstrumentation` is all that's needed (assuming all the rest was already set up). There are a couple tweaks we can do, like the `IncludeMemoryLevel` I did there, so details about the memory level are included in the traces.

With this in place, we can check out our traces. We can see when there is a cache miss:

{{< embedded-image "/images/2024/11/17/01-trace-cache-miss.png" "Trace with a cache miss" >}}

Where we see the various steps FusionCache goes through, including invoking the factory.

We can also see when there is a cache hit:

{{< embedded-image "/images/2024/11/17/02-trace-cache-hit.png" "Trace with a cache hit" >}}

In which case there are less steps, as the entry is immediately retrieved from the cache.

We can even see when there's an eager refresh, in which case the API responds immediately, but the factory is invoked in the background.

{{< embedded-image "/images/2024/11/17/03-trace-eager-refresh.png" "Trace with eager refresh" >}}

In the trace, note that not only is a tag included in the span, indicating there was an eager refresh, but also that the top level span took only around 3ms, as the factory continued to execute in the background, after the response was already returned to the API client.

### Keeping an eye on the cache hit ratio

Besides tracing, FusionCache also defines metrics we can export using the OpenTelemetry protocol, allowing us to query them and extract useful insights. When talking about a cache, I'd say the likely most helpful insights we can get are the amount of cache hits and misses, so we can calculate the cache hit ratio.

Configuring metrics is pretty much the same as configuring the traces, which we saw earlier.

```csharp
// ...
.WithMetrics(traceBuilder => traceBuilder
    .AddAspNetCoreInstrumentation()
    // ...
    .AddFusionCacheInstrumentation(o => o.IncludeMemoryLevel = true)
    .AddOtlpExporter(options => options.Endpoint = new Uri("http://localhost:4317")));
```

With the metrics being exported, I created a couple of Grafana visualizations using Prometheus queries, which allows us to have this valuable information available at a glance.

> I'm no PromQL expert, so cut me some slack if I mess something up 😅

{{< embedded-image "/images/2024/11/17/04-metrics-cache-hit-ratio-and-others.png" "Dashboard with cache hit ratio and other related metrics" >}}

The top two panels use the same query, just presenting things in different ways (time series vs stat). The PromQL query is the following:

```promql
sum(irate(fusioncache_cache_hit_total{fusioncache_cache_name="SampleCache"}[$__rate_interval]))
/
sum(irate(fusioncache_cache_get_or_set_total{fusioncache_cache_name="SampleCache"}[$__rate_interval]))
```

As for the bottom panel, it has 4 queries, each over a single metric:

```promql
sum(irate(fusioncache_cache_get_or_set_total{fusioncache_cache_name="SampleCache"}[$__rate_interval]))
```

```promql
sum(irate(fusioncache_cache_hit_total{fusioncache_cache_name="SampleCache"}[$__rate_interval]))
```

```promql
sum(irate(fusioncache_cache_miss_total{fusioncache_cache_name="SampleCache"}[$__rate_interval]))
```

```promql
sum(irate(fusioncache_eager_refresh_total{fusioncache_cache_name="SampleCache"}[$__rate_interval]))
```

> Note that I used `irate` as it more rapidly shows changes in what's going on. As per the [docs](https://prometheus.io/docs/prometheus/latest/querying/functions/#irate), if we wanted to create alerts over these metrics, we should use `rate` instead, to smooth out spikes.

To see how things behaved, I created a simple [k6](https://k6.io) script to hammer the API, playing with the amount of possible keys. If you recall, I set the cache size limit to 20, and each entry size to 1. Knowing this, we can play with the amount of keys to see behaviors like:

- Less than 20 keys, we get virtually 100% cache hit ratio, because after the initial load into cache, all the keys just stay there throughout, not expiring due to the eager refresh doing its thing
- More than 20 keys and we start to see cache misses. As we increase the number, so do the cache misses, as there are more and more entries that don't fit the cache

## Outro

That does it for this quick little post. We took a look at the pretty interesting FusionCache project, some of its features that improve on what we could do with the out of the box .NET memory cache, with a particular focus on its built-in integration with OpenTelemetry.

Please do check out FusionCache, as I didn't do enough justice of the project with this post. Also, if you wish to experiment with the things I mentioned here, do take a look at the sample repository, as everything I mentioned in the post is included, like the k6 script or the Grafana dashboard.

> Side note: I've had this post written for some months now, but forgot to publish it 😅. In the meantime, .NET 9 was released, bringing a new [HybridCache library](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid?view=aspnetcore-9.0), which might be worth a look. In any case, we've been very happy with FusionCache, so not really considering moving from one to the other at this point.

Relevant links:

- [Sample code](https://github.com/joaofbantunes/FusionCacheOpenTelemetrySample)
- [Observing .NET microservices with OpenTelemetry - logs, traces and metrics]({{< ref "2023-09-05-observing-dotnet-microservices-with-opentelemetry-logs-traces-metrics" >}})
- [Cache in-memory in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/memory?view=aspnetcore-8.0)
- [Cache stampede](https://en.wikipedia.org/wiki/Cache_stampede)
- [FusionCache](https://github.com/ZiggyCreatures/FusionCache)
- [k6](https://k6.io)
- [HybridCache library in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid?view=aspnetcore-9.0)

Thanks for stopping by, cyaz! 👋
