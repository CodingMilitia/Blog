---
author: johnny
comments: true
date: 2016-11-01 18:29:00+00:00
layout: post
slug: boilerplate-hunting-with-castle-dynamicproxy
title: Boilerplate hunting with Castle DynamicProxy
categories:
- dotnet
- libraries
tags:
- benchmark
- decorator
- dotnet
- dynamicproxy
- goldenshovel
---

[Castle Core](https://github.com/castleproject/Core) is a library that provides some utilities but I'll just talk about using one of them, the DynamicProxy. If the post on [BenchmarkDotNet](https://blog.codingmilitia.com/2016/10/24/benchmarkdotnet/)  was in the race for golden shovel award, a post on Castle DynamicProxy is a sure winner, but I feel like it :)

Castle DynamicProxy is a a "lightweight runtime proxy generator", that enables you to do a kind of [aspect oriented programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming), allowing for some code to be executed before or after a method is invoked on a proxied interface. It's useful for some cases, and I'll talk about two of such cases: caching and timing operation execution times. We could also do this using for example decorators, implementing the same interface of the target class. The problem with the decorator approach is that we need to implement all the operations one by one, but with the DynamicProxy approach we can apply the same logic to all operations as long as it's sufficiently generic.

The sample code for this post is on [GitHub](https://github.com/joaofbantunes/CastleDynamicProxySample). I also included some benchmarks with these samples, using the BenchmarkDotNet library that I talked about in a previous post.


## Timing operations execution time


I'm starting with the simplest one (at least in my barebones sample implementation). I created a class `TimingInterceptor` that implements `IInterceptor`. This is the interface that should be implemented to be able to perform actions before and/or after (or even instead of) a method is invoked. It has only the `Intercept` method that gets and `IInvocation` instance as argument, which contains all the information about the invoked method.

The implementation of the interceptor in this case is fairly simple.

{% highlight csharp linenos %}
public class TimingInterceptor : IInterceptor
{
    private readonly ILogger<TimingInterceptor> _logger;
    public TimingInterceptor(ILoggerFactory loggerFactory)
    {
        _logger = loggerFactory?.CreateLogger<TimingInterceptor>();
    }

    void IInterceptor.Intercept(IInvocation invocation)
    {
        _logger?.LogDebug(string.Format("Entered {0}.{1}()",
            invocation.MethodInvocationTarget.DeclaringType,
            invocation.MethodInvocationTarget.Name));
        var watch = Stopwatch.StartNew();
        try
        {
            invocation.Proceed();
        }
        finally
        {
            watch.Stop();
            _logger?.LogDebug("Exiting {0}.{1}() - took around {2}ms to complete",
                invocation.MethodInvocationTarget.DeclaringType,
                invocation.MethodInvocationTarget.Name,
                watch.ElapsedMilliseconds);
        }
    }
}
{% endhighlight %}

Ok, now to be honest, this is without considering async methods. Considering them we get something a bit more complex, you can check the complete implementation [here](https://github.com/joaofbantunes/CastleDynamicProxySample/blob/master/CastleDynamicProxySample/Timing/TimingInterceptor.cs).

Now to use the interceptor we must create a proxy and provide it with an interceptor instance.

{% highlight csharp linenos %}
var proxyGenerator = new ProxyGenerator();
var service = new StuffService();
var timingInterceptor = new TimingInterceptor(null);
var proxiedService = proxyGenerator
    .CreateInterfaceProxyWithTarget<IStuffService>(service, timingInterceptor);
{% endhighlight %}

Then to use it you just need to invoke the methods on the `proxiedService` instance instead of the `service` instance.


## Caching


I'm not going into much detail about this one, because it has more logic related with caching than with the interception capabilities we're talking about in this post. If you feel like it, please check out the full code on [GitHub](https://github.com/joaofbantunes/CastleDynamicProxySample) and hit me with feedback, as this turned out a real cannon to kill a fly :)

I'm not copying the whole code to the article, it has a bunch of components, so I'll just paste a simplified version (no async support) of the main class `CacheInterceptor` (in this case `SimplifiedCacheInterceptor`) so you can get a general idea of the implementation.

{% highlight csharp linenos %}
public class SimplifiedCacheInterceptor : IInterceptor
{
    private readonly ICache _cacheProvider;
    private readonly ILogger<CacheInterceptor> _logger;
    private readonly ICacheKeyCreationStrategy _cacheKeyCreationStrategy;
    private readonly IConfigurationGetter _configurationGetter;
    private readonly TimeSpan _defaultTtl;

    public SimplifiedCacheInterceptor(ICache cacheProvider,
                            ILoggerFactory loggerFactory,
                            ICacheKeyCreationStrategy cacheKeyCreationStrategy,
                            IConfigurationGetter configurationGetter,
                            TimeSpan defaultTtl)
    {
        ThrowIfNoCacheProvider(cacheProvider);
        ThrowIfNoCacheKeyCreationStrategy(cacheKeyCreationStrategy);

        _cacheProvider = cacheProvider;
        _logger = loggerFactory?.CreateLogger<CacheInterceptor>();
        _cacheKeyCreationStrategy = cacheKeyCreationStrategy;
        _configurationGetter = configurationGetter;
        _defaultTtl = defaultTtl;
    }

    private static void ThrowIfNoCacheProvider(ICache cacheProvider)
    {
        if (cacheProvider == null)
        {
            throw new ArgumentException($"\"{nameof(cacheProvider)}\" is mandatory.");
        }
    }

    private static void ThrowIfNoCacheKeyCreationStrategy(ICacheKeyCreationStrategy cacheKeyCreationStrategy)
    {
        if (cacheKeyCreationStrategy == null)
        {
            throw new ArgumentException($"\"{nameof(cacheKeyCreationStrategy)}\" is mandatory.");
        }
    }

    public void Intercept(IInvocation invocation)
    {
        try
        {
            _logger?.LogDebug("Enter interceptor for {0}.{1} ", invocation.TargetType, invocation.Method.Name);
            var config = _configurationGetter.Get(invocation);
            var cacheKey = config.UseCache ? _cacheKeyCreationStrategy.Create(config.MethodId, invocation) : null;
            object value;
            if (config.UseCache && TryGetFromCache(cacheKey, out value))
            {
                invocation.ReturnValue = value;
                return;
            }
            invocation.Proceed();
            value = invocation.ReturnValue;
            AddToCache(cacheKey, config, value);
        }
        finally
        {
            _logger?.LogDebug("Exit interceptor for {0}.{1} ", invocation.TargetType, invocation.Method.Name);
        }
    }

    public bool TryGetFromCache(string cacheKey, out object cached)
    {
        var cachedValue = _cacheProvider.Get(cacheKey);
        var isInCache = cachedValue.HasValue;
        cached = isInCache ? cachedValue.Value : null;
        return isInCache;
    }

    public void AddToCache(string cacheKey, MethodCacheConfiguration config, object toCache)
    {
        //if there is no config attribute, then no cache is used, return immediately
        if (!config.UseCache)
            return;

        //if the return is null it's only cached if explicitly indicated in the attribute CacheNullValues
        if (!config.CacheNullValues && toCache == null)
            return;

        //if the return is an empty collection it's only cached if explicitly indicated in the attribute CacheEmptyCollectionValues
        if (!config.CacheEmptyCollectionValues && toCache is IEnumerable && !CollectionHasElements((IEnumerable)toCache))
            return;

        var ttl = config.Ttl ?? _defaultTtl;

        _cacheProvider.Add(cacheKey, toCache, ttl);
    }

    private static bool CollectionHasElements(IEnumerable collection)
    {
        return collection.Cast<object>().Any();
    }
}
{% endhighlight %}

If you take a look at the main method `Intercept`, you'll see the logic is fairly simple. I did however extract some logic to other classes, like the cache key generation (allowing for different strategies) and the operation's cache configuration fetching (allowing the configuration to be stored in different locations, in my implementation, I used a custom attribute).

For the creation of the cache key I implemented two strategies: `ConfigurationBasedCacheKeyCreationStrategy` and `ReflectionBasedCacheKeyCreationStrategy`. The former is provided with the logic to create the keys on the constructor, whilst the latter uses information about the invocation to create the key. I'll show you this second one to exemplify some of the information we have access when intercepting invocations, useful for cases like this.

{% highlight csharp linenos %}
public class ReflectionBasedCacheKeyCreationStrategy : ICacheKeyCreationStrategy
{
    private readonly ILogger<ReflectionBasedCacheKeyCreationStrategy> _logger;
    private readonly Func<string, IEnumerable<string>> _argumentsToIgnoreGetter;

    public ReflectionBasedCacheKeyCreationStrategy(Func<string,IEnumerable<string>> argumentsToIgnoreGetter, ILoggerFactory loggerFactory)
    {
        _argumentsToIgnoreGetter = argumentsToIgnoreGetter;
        _logger = loggerFactory?.CreateLogger<ReflectionBasedCacheKeyCreationStrategy>();

    }

    public string Create(string methodId, IInvocation invocation)
    {
        var methodArgumentsToIgnore = _argumentsToIgnoreGetter?.Invoke(methodId) ?? Enumerable.Empty<string>();
        //fetch generic arguments and parameters
        var genericArguments = invocation.GenericArguments ?? Array.Empty<Type>();
        var parameters = invocation.MethodInvocationTarget.GetParameters();

        //prepare parameters string representation "type name: value"
        var parametersString = new List<string>();
        for (var i = 0; i < parameters.Count(); ++i)
        {
            var parameterInfo = parameters[i];
            if (methodArgumentsToIgnore.Contains(parameterInfo.Name))
            {
                continue;
            }
            parametersString.Add(string.Format("{0} {1}:{2}", parameterInfo.ParameterType, parameterInfo.Name,
                invocation.Arguments[i]));
        }

        //construct the cache key, "<generic arguments>full type name.method name(parameters)"
        var cacheKey = string.Format("<{0}>{1}.{2}({3})",
            string.Join(",", genericArguments.Select(ga => ga.Name)),
            invocation.TargetType.FullName,
            invocation.MethodInvocationTarget.Name,
            string.Join(",", parametersString)
        );

        _logger?.LogDebug($"Created cache key: \"{cacheKey}\"");
        return cacheKey;
    }
}
{% endhighlight %}

As you can see I'm using a good amount of info from the invocation, like the generic arguments, the `MethodInfo` for the target method, the concrete arguments and the type of the target class (the one that is being proxied to).


## Performance


What about performance? Well a performance penalty can be expected, but it mostly comes down to the code we gotta create to make the thing generic, rather than caused by the DynamicProxy (although there is a more noticeable impact when we first instantiate the proxy).

Using BenchmarkDotNet I created some tests. You can see the results below.

**TimingInterceptor**

    
~~~~
// * Summary *

Host Process Environment Information:
BenchmarkDotNet.Core=v0.9.9.0
OS=Windows
Processor=?, ProcessorCount=8
Frequency=2740595 ticks, Resolution=364.8843 ns, Timer=TSC
CLR=CORE, Arch=64-bit ? [RyuJIT]
GC=Concurrent Workstation
dotnet cli version: 1.0.0-preview2-003133

Type=TimingBenchmark  Mode=Throughput

                        Method |        Median |     StdDev |
  ---------------------------- |-------------- |----------- |
                  DynamicProxy |   211.0648 ns |  1.0241 ns |
             DynamicProxyAsync |   316.2422 ns |  4.2481 ns |
   DynamicProxyWithResultAsync | 2,057.3374 ns | 28.1047 ns |
                     Decorator |    50.2034 ns |  0.6157 ns |
                DecoratorAsync |   116.6465 ns |  1.4087 ns |
      DecoratorWithResultAsync |   133.0129 ns |  1.9464 ns |
~~~~


**CacheInterceptor**

    
~~~~
// * Summary *

Host Process Environment Information:
BenchmarkDotNet.Core=v0.9.9.0
OS=Windows
Processor=?, ProcessorCount=8
Frequency=2740595 ticks, Resolution=364.8843 ns, Timer=TSC
CLR=CORE, Arch=64-bit ? [RyuJIT]
GC=Concurrent Workstation
dotnet cli version: 1.0.0-preview2-003133

Type=CacheBenchmark  Mode=Throughput

                        Method |         Median |        StdDev |
 ----------------------------- |--------------- |-------------- |
        ProxyWithGeneratedKeys | 42,423.5197 ns | 2,722.0279 ns |
       ProxyWithConfiguredKeys | 26,318.9704 ns |   270.8096 ns |
   ProxyWithGeneratedKeysAsync | 48,604.7269 ns |   787.1521 ns |
  ProxyWithConfiguredKeysAsync | 32,578.6152 ns | 1,675.7112 ns |
                     Decorator |    407.3324 ns |    16.3274 ns |
                DecoratorAsync |    524.9182 ns |     5.6404 ns |
~~~~


The best benchmark result to get insight of the impact of using the DynamicProxy is the first one on the `TimingInterceptor` results. It's the one that has less logic on the interceptor. The others, as we should expect, as complexity rises so does the execution time. The best example of this is the `CacheInterceptor` when it needs to create _auto-magically_ the cache key, resorting to reflection multiple times.

At the end of the day it comes down to whether the speed decrease is acceptable or not for the specific context. In general I don't think it's slow (we're on the microseconds order of magnitude), but it's a fact, mainly in the case of the `CacheInterceptor`, that it is noticeably slower than using a more straightforward approach.


## Wrapping up


So we can see that in terms of code that can be reused, we've got a win. It does, depending on what we want to do with the proxy, come with a penalty in terms of performance. It will depend on the type of system we're building if the impact is acceptable or not.

Any suggestions and/or improvements, don't hesitate, shout about it!

Cyaz
