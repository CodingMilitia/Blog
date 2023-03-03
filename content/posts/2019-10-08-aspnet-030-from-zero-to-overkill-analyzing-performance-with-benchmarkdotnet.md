---
author: João Antunes
date: 2019-10-08 17:45:00+01:00
layout: post
title: "Episode 030 - Analyzing performance with BenchmarkDotNet - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we'll take a look at BenchmarkDotNet, to explore the performance characteristics of our code and help us make better decisions when trying to optimize it."
images:
- '/images/2019/10/08/aspnet-core-from-zero-to-overkill-e030.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- benchmarkdotnet
slug: aspnet-030-from-zero-to-overkill-analyzing-performance-with-benchmarkdotnet
---

In this episode, we'll take a look at BenchmarkDotNet, to explore the performance characteristics of our code and help us make better decisions when trying to optimize it.

For the walk-through you can check out the next video, but if you prefer reading, the written version is just below the video.

{{< youtube 8JOC8kN_WbU >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In the previous episode, we introduced [ProxyKit](https://github.com/damianh/ProxyKit) to our back for front, to simplify routing the requests to the backing APIs without having to manually write all of the HTTP requests required for those interactions.

In doing so, we introduced a new class named `ProxiedApiRouteEndpointLookup`, to help us route the incoming requests to the correct API - right now we only have the group management API, but we'll have more in the future.

A common rule of thumb is to avoid optimizing prematurely, but another one is to optimize for the hot paths. Considering that this routing helper is used in almost all requests, it makes sense that it's an important bit to optimize, making as fast as possible to limit the impact on all requests.

A good idea when we want to optimize things is to have metrics, otherwise we can't be sure we're really optimizing anything. To help us with that, in this episode we'll use [BenchmarkDotNet](https://benchmarkdotnet.org/), which is a tool to help us benchmark our code, not only by avoiding the use of the "old school" approach of writing a `for` wrapped in a `Stopwatch` to get some average running time, but adding more features on top of that, making the benchmarks not only simpler, but more reliable.

## Creating a new project for the benchmarks
First things first, we need a place to put our benchmark code. We don't need it as part of the BFF solution, as it's not something we'll use on the day to day of the development, so we can create another solution just for it.

Instead of putting things into the `src` folder (of `WebFrontend\server`), we'll create a sibling named `benchmarks`. In there we can create a new `CodingMilitia.PlayBall.WebFrontend.BackForFront.Benchmarks.sln`, plus a new project in a folder named `ApiRouting`. This project is a console one, so we can run it and have BenchmarkDotNet do its thing.

## The initial ideas
Even though the implementation of the `ProxiedApiRouteEndpointLookup` was done in [episode 029](https://blog.codingmilitia.com/2019/09/11/aspnet-029-from-zero-to-overkill-simplifying-the-bff-with-proxykit) (although it will be changed in this one), let's start with the initial couple of ideas I had to implement it, so we can then analyse the results and understand the next steps.

As we'll implement multiple versions of this class for benchmarking, we can start by creating an interface to make it easier.

`IProxiedApiRouteEndpointLookup.cs`
```csharp
public interface IProxiedApiRouteEndpointLookup
{
    bool TryGet(PathString path, out string endpoint);
}
```

The `IProxiedApiRouteEndpointLookup` has a single method `TryGet`, which given the request path, tries to match it to a backing API. If there's a match, it'll return `true` and have the path of the backing API set in the `endpoint` `out` parameter, otherwise it'll return false.

Now let's get to implementing this matching logic.

When thinking about implementing this kind of match between a route and the target API, the first thing that came to mind was to use a `Dictionary`. A simple implementation may be the following:

`Attempt01DictionaryPlusStringManipulation.cs`
```csharp
public class Attempt01DictionaryPlusStringManipulation : IProxiedApiRouteEndpointLookup
{
    private readonly Dictionary<string, string> _routeToEndpointMap;

    public Attempt01DictionaryPlusStringManipulation(Dictionary<string, string> routeToEndpointMap)
    {
        _routeToEndpointMap = routeToEndpointMap;
    }

    public bool TryGet(PathString path, out string endpoint)
    {
        var pathString = path.Value;
        var basePathEnd = pathString.Substring(1, pathString.Length - 1).IndexOf('/');
        var basePath = pathString.Substring(1, basePathEnd > 0 ? basePathEnd : pathString.Length - 1);
        return _routeToEndpointMap.TryGetValue(basePath, out endpoint);
    }
}
```

`Attempt01DictionaryPlusStringManipulation` receives a `Dictionary<string, string>` in the constructor, in which the key is the base route on the BFF and the value is the base route on a backing API. Then the logic is basically manipulating the path to get the base and then try matching it using the dictionary.

This approach seems simple enough and should also be fast. One thing that immediately popped into my head was the `string` manipulation part. As you might be aware, `string`s are immutable in .NET, so anytime we do these kinds of operations, we're creating new `string`s. That means, even if it's fast, we're creating some objects that the garbage collector needs to get rid of eventually. Maybe it's not such a big problem, but given this is code that will run on all requests, it won't harm if we minimize its impact on the application. Even if it doesn't have an immediate impact on a request, eventually a GC will need to clean things up, and that certainly has an impact.

So, what's another simple approach we could take that would minimize these string allocations? Instead of manipulating the `string`, we could iterate on the possible routes and use `PathString.StartsWithSegments(someRoute)` to check if there's a match.

`Attempt02ArrayIterationPlusPathBeginsWith.cs`
```csharp
public class Attempt02ArrayIterationPlusPathBeginsWith : IProxiedApiRouteEndpointLookup
{
    private readonly (string route, string endpoint)[] _routeCollection;

    public Attempt02ArrayIterationPlusPathBeginsWith(Dictionary<string, string> routeToEndpointMap)
    {
        _routeCollection = routeToEndpointMap.Select(e => (route: $"/{e.Key}", endpoint: e.Value)).ToArray();
    }

    public bool TryGet(PathString path, out string endpoint)
    {
        foreach (var e in _routeCollection)
        {
            if (path.StartsWithSegments(e.route))
            {
                endpoint = e.endpoint;
                return true;
            }
        }

        endpoint = null;
        return false;
    }
}
```

As we can see, we get the same `Dictionary<string, string>` in the constructor, but then convert it to an array of a `ValueTuple`, containing the route to match and the target API endpoint. Then in `TryGet` we iterate to try and find a match.

In comparison to the first approach, we can anticipate an upside and a downside. The upside is that there should be no object allocations. The downside is that iterating **might** be slower than a dictionary lookup.

Now we got a couple of solutions, but what's the best one? Enter BenchmarkDotNet so we can make informed decisions instead of guess work 🙂.

## Introducing BenchmarkDotNet
Before getting into code, maybe we should start with a quick presentation of BenchmarkDotNet, but to do it I'll just copy paste from its GitHub repo:

> Benchmarking is really hard (especially microbenchmarking), you can easily make a mistake during performance measurements. BenchmarkDotNet will protect you from the common pitfalls (even for experienced developers) because it does all the dirty work for you: it generates an isolated project per each benchmark method, does several launches of this project, run multiple iterations of the method (include warm-up), and so on. Usually, you even shouldn't care about a number of iterations because BenchmarkDotNet chooses it automatically to achieve the requested level of precision.

In summary, no more using a `Stopwatch` and running a bunch of iterations of some code to see how fast it runs. BenchmarkDotNet does that and much more, so not only we can be more confident about the results, we can get much more info than with that basic approach I mentioned.

With that out of the way, guess it's kind of obvious the first thing we need to do is to get BenchmarkDotNet installed on the project 🙂. `dotnet add package BenchmarkDotNet` will do the trick.

To run a benchmark with BenchmarkDotNet, we can simply call `BenchmarkRunner.Run<AClassWithBenchmarks>()` on our console application's `Main` method. For that, we need the class with the code to do the benchmarks, so we can create `ProxiedApiRouteEndpointLookupBenchmark`. 

There are a lot more configurations we can use, but we'll keep it simple and just run with the defaults (with a couple of attributes added to the benchmark class for some extra info, as we'll see in a bit).

### Setting up the benchmark
Let's start the `ProxiedApiRouteEndpointLookupBenchmark` class with some initial benchmark setup.

`Program.cs`
```csharp
public class ProxiedApiRouteEndpointLookupBenchmark
{
    [Params(10 , 100, 1000)] 
    public int MaxRoutes { get; set; }

    private string _path;
    private PathString _pathString;
    private static Attempt01DictionaryPlusStringManipulation _attempt01;
    private static Attempt02ArrayIterationPlusPathBeginsWith _attempt02;

    [GlobalSetup]
    public void Setup()
    {
        var routeMap = CreateRouteMap(MaxRoutes);
        _path = $"/route{MaxRoutes - 1}/some/more/things/in/the/path";
        _pathString = _path;
        
        _attempt01 = new Attempt01DictionaryPlusStringManipulation(routeMap);
        _attempt02 = new Attempt02ArrayIterationPlusPathBeginsWith(routeMap);
    }

    // ...

    private static Dictionary<string, string> CreateRouteMap(int maxRoutes)
        => Enumerable
            .Range(0, maxRoutes)
            .ToDictionary(i => $"route{i}", i => $"route{i}endpoint");

}
```

The `Setup` method, decorated with the `GlobalSetup` attribute will be called once before all benchmark iterations. This way we can do some setup that doesn't influence the actual benchmark results. 

In this `Setup` method, we're creating a test dictionary with routes, the `_path` that'll be used to try and match a route (using `MaxRoutes` to simulate the worst case scenario in which it needs to iterate over all routes) and finally instantiates the `IProxiedApiRouteEndpointLookup` implementations to test.

Another thing to note is that `MaxRoutes` is a property decorated with a `Params` attribute, with values 10, 100 and 1000. By using this attribute, BenchmarkDotNet will run the the benchmarks for all of the three options, so we can compare the impact the number of routes has on our code (i.e. we expect it to have more impact on the iteration based implementation).

With the setup done, we need the code to call our two `IProxiedApiRouteEndpointLookup` implementations, so we add the following methods:

`Program.cs`
```csharp
[Benchmark(Baseline = true)]
public string Attempt01DictionaryPlusStringManipulation()
{
    _attempt01.TryGet(_path, out var result);
    return result;
}

[Benchmark]
public string Attempt02ArrayIterationPlusPathBeginsWith()
{
    _attempt02.TryGet(_path, out var result);
    return result;
}
```

As we can see, the methods are pretty much the same, just calling a different implementation of our route matcher. We mark the methods with a `Benchmark` attribute, so BenchmarkDotNet knows those are the methods we want it to benchmark. Quick note about the `Baseline` property of the `Benchmark` on the first method, which means that's the "default" implementation we're considering, comparing the others to it to analyse the potential benefits and drawbacks.

With these methods we're mostly ready to run the benchmark, we'll just add a couple more attributes to the `ProxiedApiRouteEndpointLookupBenchmark` class:

`Program.cs`
```csharp
[RankColumn, MemoryDiagnoser]
public class ProxiedApiRouteEndpointLookupBenchmark
```

The `RankColumn` will add a column to the output of the benchmark, indicating which method was faster. The `MemoryDiagnoser` attribute will make BenchmarkDotNet also analyse memory, not only speed, including information in the results.

### Running the benchmark and analyzing the results
Now to run the benchmark, we can do `dotnet run -c Release`. Note that the configuration parameter is important, otherwise we'll be benchmarking non-optimized debug mode code.

When it starts running, we can pay some attention to the console and notice some of the things that are going on, like some warmup iterations and the actual benchmarking iterations.

It takes a bit to complete, but when it does, we get some simple results to look at (we can also get more complex stuff, even charts, but again, we're keeping it simple). Besides showing the results on the console, it also creates some files in a `BenchmarkDotNet.Artifacts` folder, being one of them a Markdown file which is great to share on GitHub and also on this post 🙂 (although the table is way larger than the normal post content).

``` ini

BenchmarkDotNet=v0.11.5, OS=Windows 10.0.18362
Intel Core i7-9700K CPU 3.60GHz (Coffee Lake), 1 CPU, 8 logical and 8 physical cores
.NET Core SDK=3.0.100
  [Host]     : .NET Core 3.0.0 (CoreCLR 4.700.19.46205, CoreFX 4.700.19.46214), 64bit RyuJIT
  DefaultJob : .NET Core 3.0.0 (CoreCLR 4.700.19.46205, CoreFX 4.700.19.46214), 64bit RyuJIT

```

|                                    Method | MaxRoutes |         Mean |       Error |      StdDev |  Ratio | RatioSD | Rank |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|------------------------------------------ |---------- |-------------:|------------:|------------:|-------:|--------:|-----:|-------:|------:|------:|----------:|
| **Attempt01DictionaryPlusStringManipulation** |        **10** |     **41.14 ns** |   **0.3486 ns** |   **0.3090 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
| Attempt02ArrayIterationPlusPathBeginsWith |        10 |    171.17 ns |   0.8355 ns |   0.6977 ns |   4.16 |    0.03 |    2 |      - |     - |     - |         - |
|                                           |           |              |             |             |        |         |      |        |       |       |           |
| **Attempt01DictionaryPlusStringManipulation** |       **100** |     **44.76 ns** |   **0.9294 ns** |   **0.9544 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
| Attempt02ArrayIterationPlusPathBeginsWith |       100 |  1,705.33 ns |  15.7526 ns |  14.7350 ns |  38.05 |    0.87 |    2 |      - |     - |     - |         - |
|                                           |           |              |             |             |        |         |      |        |       |       |           |
| **Attempt01DictionaryPlusStringManipulation** |      **1000** |     **44.46 ns** |   **0.6133 ns** |   **0.5737 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
| Attempt02ArrayIterationPlusPathBeginsWith |      1000 | 18,051.21 ns | 152.0076 ns | 142.1880 ns | 406.11 |    6.97 |    2 |      - |     - |     - |         - |

It starts with some context information about the conditions in which the benchmarks ran: BenchmarkDotNet version, operating system, computer hardware, .NET version (besides .NET Core, we could also run it in .NET Framework and Mono for example) and compiler information.

After that, we get to what interests us the most, the comparison between our two implementations. The method names appear, split into three groups for the number of `MaxRoutes` we configured. 

Comparing `Attempt01DictionaryPlusStringManipulation` to `Attempt02ArrayIterationPlusPathBeginsWith`, it probably falls in line with what we expected. The dictionary version is faster, but allocates memory. Also, the iteration version gets slower the more routes it needs to iterate over, while the dictionary version keeps its pace.

So, conclusion? The dictionary version is much faster - sure, with 10 routes the iteration version is still in nanoseconds land, but it's four times slower, even with only 10 routes, which isn't a lot.

In summary, what we'd like would be a dictionary version that could avoid allocations... perhaps we can work on that? 🙂

## Working on the dictionary solution
The dictionary solution seems like a good one, particularly in terms of speed, but if we could avoid allocations, it would be great. Is it possible? Well, if it weren't (or at least if I didn't know it was 😛) this part of the post wouldn't exist would it? 🙃

With .NET Core 2.1 some new types named `Span<T>` and `Memory<T>` were introduced, which allows for strongly-typed and allocation free management of contiguous memory, from a variety of different sources. With this description we can already tell these are very powerful constructs to help in a variety of scenarios. 

What interests us in the context of our problem, is that we can use these, particularly `Span` to work with `string`s, without always creating a new one, by providing a window over the `string`'s memory. In fact, one of the facts that makes these new types so important, even if we don't use them directly, is that they are heavily used behind the scenes by ASP.NET Core, to improve the performance of things like request parsing, which are heavy on `string` manipulation as you might expect. 

If we call `AsSpan()` on a `string`, we get a `ReadOnlySpan<char>` with which we can play around with the `string` without allocations - not being able to change the underlying `string` of course, as it's immutable hence the `ReadOnly` prefix on the actual `Span` type used.

With this in mind, let's see what we can do to improve our dictionary based solution.

### Double dictionary lookup
The first thing we might think, as we have `Span`, is we can just use it to replace the `string` as our key in our dictionary. Unfortunately not, as `Span` can only live in the stack, not the heap (more details [here](https://github.com/dotnet/corefxlab/blob/master/docs/specs/span.md#struct-tearing)).

So we can't put it in the dictionary, but given one of `Span`'s strengths is working with `string`s, maybe there's a way to keep the `string` in there but index it with a `Span`? Unfortunately no, at least for now, there's a [issue open on GitHub](https://github.com/dotnet/corefx/issues/31942) regarding that possibility.

Ok, we need more ideas... The dictionary depends on `object.GetHashCode` method to find the index of the value we want, if there's a way to get the hash code of a `Span` that matches that of the `string` it represents, we might be able to work with that.

And wouldn't you know, we can! There's a `static` `string` method named `GetHashCode` that gets a `ReadOnlySpan<char>` as an argument. With this, instead of having a `string` key, we'll move to an `int` key, which will be the hash code of the `string`. We can't however rely solely on this, as if we get some [collisions]((https://en.wikipedia.org/wiki/Collision_(computer_science))) with `GetHashCode` we'll forward to the wrong route.

For the first attempt, we'll use two dictionaries:
- One that matches the hash code to an array of `string`s, which are the ones that have that hash code. We then iterate on that array to find the actual match.
- With the actual match, we can index on the second dictionary and obtain the endpoint to forward to.

The following code implements this logic:

`Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation.cs`
```csharp
public class Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation : IProxiedApiRouteEndpointLookup
{
    private readonly Dictionary<string, string> _routeToEndpointMap;
    private readonly Dictionary<int, string[]> _routeMatcher;
    
    public Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation(Dictionary<string, string> routeToEndpointMap)
    {
        _routeToEndpointMap = routeToEndpointMap ?? throw new ArgumentNullException(nameof(routeToEndpointMap));
        _routeMatcher = _routeToEndpointMap
            .Keys
            .GroupBy(
                r => r.GetHashCode(),
                r => r)
            .ToDictionary(
                g => g.Key,
                g => g.ToArray());
    }

    public bool TryGet(PathString path, out string endpoint)
    {
        endpoint = null;
        var pathSpan = path.Value.AsSpan();
        var basePathEnd = pathSpan.Slice(1, pathSpan.Length - 1).IndexOf('/');
        var basePath = pathSpan.Slice(1, basePathEnd > 0 ? basePathEnd : pathSpan.Length - 1);

        if (_routeMatcher.TryGetValue(string.GetHashCode(basePath), out var routes))
        {
            var route = FindRoute(basePath, routes);
            return !(route is null) && _routeToEndpointMap.TryGetValue(route, out endpoint);
        }

        return false;
    }

    private static string FindRoute(ReadOnlySpan<char> route, string[] routes)
    {
        foreach(var currentRoute in routes)
        {
            if (route.Equals(currentRoute, StringComparison.InvariantCultureIgnoreCase))
            {
                return currentRoute;
            }
        }

        return null;
    }
}
```

Cool, we have a new solution. Now, is it any good? Let's add a new benchmark method to `ProxiedApiRouteEndpointLookupBenchmark` and rerun the program to see if we get some improvement.

|                                                     Method | MaxRoutes |         Mean |      Error |     StdDev |  Ratio | RatioSD | Rank |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|----------------------------------------------------------- |---------- |-------------:|-----------:|-----------:|-------:|--------:|-----:|-------:|------:|------:|----------:|
|                  **Attempt01DictionaryPlusStringManipulation** |        **10** |     **43.08 ns** |  **0.5195 ns** |  **0.4860 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
|                  Attempt02ArrayIterationPlusPathBeginsWith |        10 |    172.66 ns |  1.3142 ns |  1.2293 ns |   4.01 |    0.05 |    3 |      - |     - |     - |         - |
| Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation |        10 |     88.32 ns |  0.3901 ns |  0.3458 ns |   2.05 |    0.03 |    2 |      - |     - |     - |         - |
|                                                            |           |              |            |            |        |         |      |        |       |       |           |
|                  **Attempt01DictionaryPlusStringManipulation** |       **100** |     **45.96 ns** |  **0.9345 ns** |  **0.8741 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
|                  Attempt02ArrayIterationPlusPathBeginsWith |       100 |  1,688.49 ns |  2.7973 ns |  2.6166 ns |  36.75 |    0.72 |    3 |      - |     - |     - |         - |
| Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation |       100 |     91.95 ns |  0.5203 ns |  0.4345 ns |   2.01 |    0.04 |    2 |      - |     - |     - |         - |
|                                                            |           |              |            |            |        |         |      |        |       |       |           |
|                  **Attempt01DictionaryPlusStringManipulation** |      **1000** |     **44.82 ns** |  **0.6149 ns** |  **0.5752 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
|                  Attempt02ArrayIterationPlusPathBeginsWith |      1000 | 17,977.94 ns | 95.5907 ns | 89.4156 ns | 401.16 |    5.84 |    3 |      - |     - |     - |         - |
| Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation |      1000 |     97.31 ns |  0.4160 ns |  0.3892 ns |   2.17 |    0.03 |    2 |      - |     - |     - |         - |

In comparison with the previous two solutions, we can see some expected yet interesting results. 

This new solution is slower, about twice as much execution time, than the original dictionary solution. I guess expected, as we're doing two dictionary lookups. It however doesn't allocate, which is a nice perk, and in comparison to the array iteration solution, is about twice as fast for the 10 routes test and doesn't get much worse with the increase in routes.

So, maybe it's a good middle ground between the two original attempts?

### Single dictionary with complex value
Some time after implementing this third solution I got thinking, why in the world do I need the second dictionary?!? 🤪

Instead of indexing one dictionary, iterating the possible routes and getting the desired one to index on another dictionary, I could have just stored all the required information immediately on the first dictionary. That's why we should refactor our code, we don't always get it right at first 🙂.

The following code does away with the double dictionary, including a new class to hold all the required information.

`Attempt04HashCodeBasedDictionaryWithComplexValuePlusSpanManipulation.cs`
```csharp
public class Attempt04HashCodeBasedDictionaryWithComplexValuePlusSpanManipulation : IProxiedApiRouteEndpointLookup
{
    private readonly Dictionary<int, Holder[]> _routeMatcher;

    public Attempt04HashCodeBasedDictionaryWithComplexValuePlusSpanManipulation(Dictionary<string, string> routeToEndpointMap)
    {
        var tempRouteMatcher = new Dictionary<int, List<Holder>>();
        foreach (var entry in routeToEndpointMap)
        {
            var hashCode = entry.Key.GetHashCode();
            if (tempRouteMatcher.TryGetValue(hashCode, out var route))
            {
                route.Add(new Holder(entry.Key, entry.Value));
            }
            else
            {
                tempRouteMatcher.Add(hashCode, new List<Holder> {new Holder(entry.Key, entry.Value)});
            }
        }

        _routeMatcher = tempRouteMatcher.ToDictionary(e => e.Key, e => e.Value.ToArray());
    }

    public bool TryGet(PathString path, out string endpoint)
    {
        endpoint = null;
        var pathSpan = path.Value.AsSpan();
        var basePathEnd = pathSpan.Slice(1, pathSpan.Length - 1).IndexOf('/');
        var basePath = pathSpan.Slice(1, basePathEnd > 0 ? basePathEnd : pathSpan.Length - 1);

        if (_routeMatcher.TryGetValue(string.GetHashCode(basePath), out var routes))
        {
            endpoint = FindRoute(basePath, routes);
            return endpoint != null;
        }

        return false;
    }

    private static string FindRoute(ReadOnlySpan<char> route, Holder[] routes)
    {
        foreach(var currentRoute in routes)
        {
            if (route.Equals(currentRoute.route, StringComparison.InvariantCultureIgnoreCase))
            {
                return currentRoute.endpoint;
            }
        }
        return null;
    }

    private class Holder
    {
        public readonly string route;
        public readonly string endpoint;

        public Holder(string route, string endpoint)
        {
            this.route = route;
            this.endpoint = endpoint;
        }
    }
}
```

Let's get back to our `ProxiedApiRouteEndpointLookupBenchmark`, add a new benchmark and rerun the program.

|                                                               Method | MaxRoutes |         Mean |       Error |      StdDev |  Ratio | RatioSD | Rank |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|--------------------------------------------------------------------- |---------- |-------------:|------------:|------------:|-------:|--------:|-----:|-------:|------:|------:|----------:|
|                            **Attempt01DictionaryPlusStringManipulation** |        **10** |     **44.48 ns** |   **0.5870 ns** |   **0.5490 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
|                            Attempt02ArrayIterationPlusPathBeginsWith |        10 |    173.35 ns |   1.6154 ns |   1.5110 ns |   3.90 |    0.05 |    4 |      - |     - |     - |         - |
|           Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation |        10 |     88.18 ns |   0.6716 ns |   0.6282 ns |   1.98 |    0.03 |    3 |      - |     - |     - |         - |
| Attempt04HashCodeBasedDictionaryWithComplexValuePlusSpanManipulation |        10 |     73.74 ns |   0.7065 ns |   0.6263 ns |   1.66 |    0.03 |    2 |      - |     - |     - |         - |
|                                                                      |           |              |             |             |        |         |      |        |       |       |           |
|                            **Attempt01DictionaryPlusStringManipulation** |       **100** |     **45.70 ns** |   **0.5686 ns** |   **0.5319 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
|                            Attempt02ArrayIterationPlusPathBeginsWith |       100 |  1,711.32 ns |  13.8033 ns |  12.9116 ns |  37.45 |    0.47 |    4 |      - |     - |     - |         - |
|           Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation |       100 |     92.01 ns |   0.5264 ns |   0.4924 ns |   2.01 |    0.03 |    3 |      - |     - |     - |         - |
| Attempt04HashCodeBasedDictionaryWithComplexValuePlusSpanManipulation |       100 |     75.68 ns |   0.4511 ns |   0.4219 ns |   1.66 |    0.02 |    2 |      - |     - |     - |         - |
|                                                                      |           |              |             |             |        |         |      |        |       |       |           |
|                            **Attempt01DictionaryPlusStringManipulation** |      **1000** |     **43.45 ns** |   **0.5250 ns** |   **0.4910 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
|                            Attempt02ArrayIterationPlusPathBeginsWith |      1000 | 18,001.43 ns | 157.7412 ns | 147.5512 ns | 414.33 |    4.59 |    4 |      - |     - |     - |         - |
|           Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation |      1000 |     97.92 ns |   0.7619 ns |   0.7127 ns |   2.25 |    0.04 |    3 |      - |     - |     - |         - |
| Attempt04HashCodeBasedDictionaryWithComplexValuePlusSpanManipulation |      1000 |     79.54 ns |   0.9249 ns |   0.8651 ns |   1.83 |    0.03 |    2 |      - |     - |     - |         - |


So, still not as fast as the original, but better than our previous attempt. Not too bad. Good enough for now? Yes, but... there's one last little tweak we can try.

### Playing with aggressive inlining
Although in our regular day to day line of business code (I'm talking for myself at least) calling a method not being a problem - in fact it's recommended, to make code more readable to split things in as small as possible methods that make the most sense when looking at the code - this simple act, with which we don't really worry can have an impact on performance, as calling a method is more expensive than just having all the code in the same one.

Looking at `Attempt04HashCodeBasedDictionaryWithComplexValuePlusSpanManipulation`, we see that we have the `FindRoute` auxiliary method, to help keep the code cleaner in `TryGet`. Maybe pulling this code onto the `TryGet` method can provide some improvement? At the same time, I like keeping the code as readable as possible, and doing this will make it a little worse.

We can however try something in between. Depending on the characteristics of the code, some methods may be [inlined](https://en.wikipedia.org/wiki/Inline_expansion) by the [JIT](https://en.wikipedia.org/wiki/Just-in-time_compilation) compiler, which means the compiler itself puts the content of the method directly in a calling method, avoiding the "expensive" method call.

Given this `FindRoute` method is pretty simple, maybe it can be inlined? Or maybe the JIT is already doing it? Again, we'll resort to BenchmarkDotNet to help us find out.

To get information about inlining, BenchmarkDotNet has a diagnoser named `InliningDiagnoser`, which doesn't come out of the box, we need to install the `BenchmarkDotNet.Diagnostics.Windows` NuGet package. After installing it, we can decorate our benchmarks class with a new attribute:

`Program.cs`
```csharp
[RankColumn, MemoryDiagnoser, InliningDiagnoser(logFailuresOnly: false)]
public class ProxiedApiRouteEndpointLookupBenchmark
```

Now we can rerun the program and take a look at what we get in the output. Looking at the output, we get a looooot of information, so I'll just drop a couple of examples in here:

```md
--------------------
Inliner: CodingMilitia.PlayBall.WebFrontend.BackForFront.Benchmarks.ProxiedApiRouteEndpointLookup.Attempt01DictionaryPlusStringManipulation.TryGet - instance bool  (value class Microsoft.AspNetCore.Http.PathString,class System.String&)
Inlinee: System.String.IndexOf - instance int32  (wchar)
--------------------
Inliner: CodingMilitia.PlayBall.WebFrontend.BackForFront.Benchmarks.ProxiedApiRouteEndpointLookup.Attempt01DictionaryPlusStringManipulation.TryGet - instance bool  (value class Microsoft.AspNetCore.Http.PathString,class System.String&)
Inlinee: System.Collections.Generic.Dictionary`2[System.__Canon,System.__Canon].TryGetValue - instance bool  (!0,!1&)
Fail Reason: unprofitable inline
--------------------
```

On the first example, we see a successful inline, where the inliner is our `Attempt01DictionaryPlusStringManipulation.TryGet` method and the inlinee is a call to `string.IndexOf`. The second example is a failed inline, also from our `Attempt01DictionaryPlusStringManipulation.TryGet` method, but in this case for the `Dictionary.TryGetValue` method.

If we look for our `FindRoute` method in the output, it appears for our third approach (`Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation`) as a failed inline (reason "unprofitable inline"), but doesn't show up for the latest attempt. Let's try to give the JIT a hint that it would be nice of it to inline that method. To do this, let's create a fifth attempt class, basically the same as the fourth, but adding an hint to `FindRoute`:

`Attempt0504WithAggressiveInlining.cs`
```csharp
public class Attempt0504WithAggressiveInlining : IProxiedApiRouteEndpointLookup
{
    // ...

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    private static string FindRoute(ReadOnlySpan<char> route, Holder[] routes)
    {
        // ...
    }

    // ...
}
```

Using the `MethodImpl` attribute, with the parameter `MethodImplOptions.AggressiveInlining`, we're dropping in a hint for the JIT that it should inline the method. Of course it will only be inlined if possible, so let's add this new class to the benchmark and see what happens.

Analyzing the output now, we'll find a reference to `Attempt0504WithAggressiveInlining.FindRoute`, and it was inlined.

```
--------------------
Inliner: CodingMilitia.PlayBall.WebFrontend.BackForFront.Benchmarks.ProxiedApiRouteEndpointLookup.Attempt0504WithAggressiveInlining.TryGet - instance bool  (value class Microsoft.AspNetCore.Http.PathString,class System.String&)
Inlinee: CodingMilitia.PlayBall.WebFrontend.BackForFront.Benchmarks.ProxiedApiRouteEndpointLookup.Attempt0504WithAggressiveInlining.FindRoute - class System.String  (value class System.ReadOnlySpan`1<wchar>,class Holder[])
--------------------
```

Now the question is, did it actually improve something?

|                                                               Method | MaxRoutes |         Mean |      Error |     StdDev |  Ratio | RatioSD | Rank |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|--------------------------------------------------------------------- |---------- |-------------:|-----------:|-----------:|-------:|--------:|-----:|-------:|------:|------:|----------:|
|                            **Attempt01DictionaryPlusStringManipulation** |        **10** |     **44.59 ns** |  **0.6495 ns** |  **0.6076 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
|                            Attempt02ArrayIterationPlusPathBeginsWith |        10 |    167.00 ns |  0.5707 ns |  0.5059 ns |   3.75 |    0.05 |    5 |      - |     - |     - |         - |
|           Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation |        10 |     88.52 ns |  0.2429 ns |  0.2272 ns |   1.99 |    0.03 |    4 |      - |     - |     - |         - |
| Attempt04HashCodeBasedDictionaryWithComplexValuePlusSpanManipulation |        10 |     73.56 ns |  0.1517 ns |  0.1419 ns |   1.65 |    0.02 |    3 |      - |     - |     - |         - |
|                                    Attempt0504WithAggressiveInlining |        10 |     71.06 ns |  0.1551 ns |  0.1451 ns |   1.59 |    0.02 |    2 |      - |     - |     - |         - |
|                                                                      |           |              |            |            |        |         |      |        |       |       |           |
|                            **Attempt01DictionaryPlusStringManipulation** |       **100** |     **44.62 ns** |  **0.1878 ns** |  **0.1757 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
|                            Attempt02ArrayIterationPlusPathBeginsWith |       100 |  1,688.19 ns |  4.6872 ns |  4.3844 ns |  37.83 |    0.16 |    5 |      - |     - |     - |         - |
|           Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation |       100 |     91.50 ns |  0.3578 ns |  0.3347 ns |   2.05 |    0.01 |    4 |      - |     - |     - |         - |
| Attempt04HashCodeBasedDictionaryWithComplexValuePlusSpanManipulation |       100 |     75.37 ns |  0.1708 ns |  0.1514 ns |   1.69 |    0.01 |    3 |      - |     - |     - |         - |
|                                    Attempt0504WithAggressiveInlining |       100 |     71.43 ns |  0.1894 ns |  0.1772 ns |   1.60 |    0.01 |    2 |      - |     - |     - |         - |
|                                                                      |           |              |            |            |        |         |      |        |       |       |           |
|                            **Attempt01DictionaryPlusStringManipulation** |      **1000** |     **43.86 ns** |  **0.2840 ns** |  **0.2656 ns** |   **1.00** |    **0.00** |    **1** | **0.0216** |     **-** |     **-** |     **136 B** |
|                            Attempt02ArrayIterationPlusPathBeginsWith |      1000 | 17,866.32 ns | 50.6077 ns | 47.3385 ns | 407.36 |    1.87 |    5 |      - |     - |     - |         - |
|           Attempt03HashCodeBasedDoubleDictionaryPlusSpanManipulation |      1000 |     98.00 ns |  0.4173 ns |  0.3903 ns |   2.23 |    0.02 |    4 |      - |     - |     - |         - |
| Attempt04HashCodeBasedDictionaryWithComplexValuePlusSpanManipulation |      1000 |     78.88 ns |  0.3112 ns |  0.2911 ns |   1.80 |    0.01 |    3 |      - |     - |     - |         - |
|                                    Attempt0504WithAggressiveInlining |      1000 |     74.99 ns |  0.1855 ns |  0.1549 ns |   1.71 |    0.01 |    2 |      - |     - |     - |         - |


Looking at the results, apparently yes. Clearly not a lot, just a couple of nanoseconds, but still, we'll take it! 😀

## Outro
That's a wrap for this episode. We took the opportunity to play around with BenchmarkDotNet, allowing us to have a better understanding of the impact of our changes while trying to optimize a piece of code that's used in a hot path of our BFF, running in all (or almost all) requests.

There are probably other ways to improve this code, so feel free to suggest some changes or just head to the repo and open an issue/pull request, everything here is available on GitHub 😉.

Links in the post:
- [BenchmarkDotNet](https://benchmarkdotnet.org/)
- [ProxyKit](https://github.com/damianh/ProxyKit)
- [Span&lt;T&gt; spec](https://github.com/dotnet/corefxlab/blob/master/docs/specs/span.md)
- [Add string keyed dictionary ReadOnlySpan&lt;char&gt; lookup support](https://github.com/dotnet/corefx/issues/31942)
- [Hash collision](https://en.wikipedia.org/wiki/Collision_(computer_science))
- [Inline expansion](https://en.wikipedia.org/wiki/Inline_expansion)
- [Adventures in Benchmarking - Method Inlining](https://mattwarren.org/2016/03/09/adventures-in-benchmarking-method-inlining/)
- [When is a method eligible to be inlined by the CLR?](https://stackoverflow.com/q/4660004/4923902)
- [Just-in-time compilation](https://en.wikipedia.org/wiki/Just-in-time_compilation)
- [Episode 029 - Simplifying the BFF with ProxyKit - ASP.NET Core: From 0 to overkill](https://blog.codingmilitia.com/2019/09/11/aspnet-029-from-zero-to-overkill-simplifying-the-bff-with-proxykit)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode030).

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!