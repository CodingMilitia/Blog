---
author: johnny
comments: true
date: 2016-10-24 08:37:46+00:00
layout: post
slug: benchmarkdotnet
title: BenchmarkDotNet - Library for benchmarking .NET code
categories:
- dotnet
- libraries
tags:
- benchmark
- dotnet
- goldenshovel
---

This will probably earn me what some friends of mine call a golden shovel (an award for finding out something everyone already knows) but since I just recently found out about it, I'll write about it anyway.

Several times I felt the need to benchmark some portions of code so I could see if a way of doing something could be a slow down cause for an application (many of those times were when I wanted to use some reflection to simplify the code and some colleagues were afraid reflection was going to have a negative impact on performance). When I wanted to do that, I just wrote a console application, throw a bunch of [Stopwatches ](https://msdn.microsoft.com/en-us/library/system.diagnostics.stopwatch)around and put a loop around the code I wanted to test to get an average execution time.

That was a lot of boilerplate and no one ever came around to writing a library with the generic portions of the code. Well, "now" (like if this was something new) we don't need to, we can just use [BenchmarkDotNet](https://github.com/PerfDotNet/BenchmarkDotNet). This nice little library allows for just this simple benchmarks I talked about but also has a good amount of options to make some more advanced benchmarking. I'll just show a very simple sample, but the [documentation](https://perfdotnet.github.io/BenchmarkDotNet/) covers the stuff you can do with it.

The way the library works, as per the documentation:

> BenchmarkDotNet follows the following steps to run your benchmarks:
>
> 
> 
>	
>   1. `BenchmarkRunner` generates an isolated project per each benchmark method/job/params and builds it in Release mode.
> 
>	
>   2. Next, we take each method/job/params combination and try to measure its performance by launching benchmark process several times (`LaunchCount`).
> 
>
>   3. An invocation of the target method is an _operation_. A bunch of operation is an _iteration_. If you have a `Setup` method, it will be invoked before each iteration, but not between operations. We have the following type of iterations:
>
>	
>     * `Pilot`: The best operation count will be chosen.
> 
>	
>     * `IdleWarmup`, `IdleTarget`: BenchmarkDotNet overhead will be evaluated.
> 
>	
>     * `MainWarmup`: Warmup of the main method.
> 
>	
>     * `MainTarget`: Main measurements.
> 
>	
>     * `Result` = `MainTarget` - ``
> 
>
> 
>	
>   4. After all of the measurements, BenchmarkDotNet creates:
>
>	
>     * An instance of the `Summary` class that contains all information about benchmark runs.
> 
>	
>     * A set of files that contains summary in human-readable and machine-readable formats.
> 
>	
>     * A set of plots.


For my simple test (you can check it on [GitHub](https://github.com/joaofbantunes/BenchmarkDotNetSample)) I created a console application using .NET Core and added [BenchmarkDotNet](https://www.nuget.org/packages/BenchmarkDotNet/) and [BenchmarkDotNet.Core](https://www.nuget.org/packages/BenchmarkDotNet.Core/) Nuget packages. Then created a class with the code to benchmark.

{% highlight csharp linenos %}
using BenchmarkDotNet.Attributes;

namespace BenchmarkDotNetSample
{
    public class GetNameBenchmark
    {
        [Benchmark]
        public string Reflection()
        {
            return typeof(GetNameBenchmark).Name;
        }

        [Benchmark]
        public string NameOf()
        {
            return nameof(GetNameBenchmark);
        }
    }
}
{% endhighlight %}

Pretty simple right? (and the code to benchmark itself couldn't be more useless, you can see what'll happen from a mile away)

Now to run the benchmark, Program.cs looks like this:

{% highlight csharp linenos %}using BenchmarkDotNet.Configs;
using BenchmarkDotNet.Jobs;
using BenchmarkDotNet.Running;

namespace BenchmarkDotNetSample
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var summary = BenchmarkRunner.Run<GetNameBenchmark>();
        }
    }
}
{% endhighlight %}

This is just running the default configuration, we can play around with the parameters, depending on what we want.

{% highlight csharp linenos %}using BenchmarkDotNet.Configs;
using BenchmarkDotNet.Jobs;
using BenchmarkDotNet.Running;

namespace BenchmarkDotNetSample
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var summary = BenchmarkRunner.Run<GetNameBenchmark>(
                DefaultConfig.Instance.With(
                    new Job()
                        .With(Mode.Throughput)
                        .WithLaunchCount(2)
                        .WithWarmupCount(10)
                        .WithTargetCount(200)
                ));
        }
    }
}{% endhighlight %}

With this changes the output doesn't change a lot, I'm just messing a bit with the iterations used to perform the benchmark (the `Mode.Throughput` isn't doing anything as it's already the default).

The final console output from the above sample, disregarding all that's being written while performing the benchmarks, is something like this (besides some HTML, CSV and markdown files that are also generated):

    
    <code>// * Summary *
    
    Host Process Environment Information:
    BenchmarkDotNet.Core=v0.9.9.0
    OS=Windows
    Processor=?, ProcessorCount=8
    Frequency=2740584 ticks, Resolution=364.8857 ns, Timer=TSC
    CLR=CORE, Arch=64-bit ? [RyuJIT]
    GC=Concurrent Workstation
    dotnet cli version: 1.0.0-preview2-003133
    
    Type=GetNameBenchmark  Mode=Throughput  LaunchCount=2
    WarmupCount=10  TargetCount=200
    
         Method |     Median |    StdDev |
    ----------- |----------- |---------- |
     Reflection | 16.0348 ns | 0.7896 ns |
         NameOf |  0.0005 ns | 0.0147 ns |</code>


There a lot more options than the ones I used, like running the benchmarks in different framework versions, different runtimes, different jits, export in different formats and more. You can check all the options at the [docs](https://perfdotnet.github.io/BenchmarkDotNet/Configuration.htm).

This was just a very simple article (and sample) about the possibilities of the library. If you find it useful then you really should check out the more complete examples in the official project [repository](https://github.com/PerfDotNet/BenchmarkDotNet/tree/master/samples).

Cyaz
