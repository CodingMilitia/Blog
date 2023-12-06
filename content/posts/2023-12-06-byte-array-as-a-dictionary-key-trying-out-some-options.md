---
author: João Antunes
date: 2023-12-06 13:30:00+00:00
layout: post
title: Byte array as a dictionary key? Trying out some options
summary: Using arrays as a dictionary key is not really something very common, but I had the need for it, so did some investigation to try and figure out a good way to do it.
images:
  - '/images/2023/12/06/byte-array-as-a-dictionary-key-trying-out-some-options.png'
categories:
  - csharp
  - dotnet
tags:
  - performance
  - benchmarkdotnet
slug: byte-array-as-a-dictionary-key-trying-out-some-options
---
## Intro

Ever needed to use a byte array, or any collection for that matter, as the key of a dictionary? I imagine it's not super common, at least it isn't in the type of software I usually write, but I recently had a need for it (more on the why in a bit).

This post goes through some experiments to try and understand what approaches would be good to implement something like this, making use of benchmarks to guide the process (using the awesome [BenchmarkDotNet](https://benchmarkdotnet.org) library).

A word of warning: I'm no expert in these kinds of performance matters, so remember that, and, as you should always do anyway, do your due diligence if you need to use some of the approaches I'll describe in this post, as I may say something stupid. If this ends up being a train wreck, I blame [Khalid](https://mastodon.social/@khalidabuhakmeh/111518228367336212) for pushing me into writing it 😅.

PS: the benchmark result tables might not be super easy to read on the blog, so I added a screenshot of the whole thing at the end, which hopefully is easier to view as a whole.

## Context: why a byte array as a dictionary key?

Before getting into the interesting stuff, just a bit of context on why could I possibly need a byte array as a dictionary key.

So, I've been messing around with some ideas to create a library that implements parallel message consumption from Apache Kafka topics, as, out of the box, Confluent's .NET Kafka client library gives us a simple consumer to handle messages one by one.

In Kafka, each message (or, more precisely named, record) is comprised of a key and a value (plus some other things not relevant right now). The key is used to decide into which partition to write a message. Each partition can only be read from one consumer at a time, and messages are read sequentially from a partition, thus achieving ordered reading of messages from a partition.

Implementing parallelism when consuming messages from Kafka, I don't want to lose completely the ordering guarantees provided, so one of the parallelism strategies I want to implement, is parallelism per key, which means all messages can be processed in parallel unless their key is the same. To implement this parallelism control, I'm using a dictionary where the key is the message key, and the value is a queue of messages for that key.

This explains why I need something as a key, now why is it a byte array? When consuming messages from Kafka, we can either configure a deserializer, and the consumer will immediately give us deserialized keys and values, or we can ignore configuring a deserializer, and get the data as a byte array. For simplicity, instead of filling the code unnecessarily with generics to support any types of keys and values, as well as to defer the deserialization work to a later point where it's more relevant, I decided to work directly with the byte arrays, hence needing to make use of the message key as a byte array (or something derived from it) to help in implementing the parallelism algorithm.

## Setup

Let's start with a very brief look at the benchmarking setup. As I mentioned earlier, I'm using BenchmarkDotNet, and created the class `KeyComparisonBenchmark` to contain all the benchmarking code.

In the benchmark class constructor, I'm initializing a bunch of stuff that is not relevant to the measurements, like creating a collection of keys, instantiating some supporting types, and so on.

```csharp
[RankColumn, MinColumn, MaxColumn]
[MemoryDiagnoser]
public class KeyComparisonBenchmark
{
    private readonly DefaultObjectPool<byte[]> _byteArrayPool;
    private readonly IEqualityComparer<byte[]> _comparerWithLinq;
    private readonly IEqualityComparer<byte[]> _comparerWithoutLinqInHashCode;
    private readonly IEqualityComparer<byte[]> _comparerAlternative;
    private readonly IEqualityComparer<byte[]> _comparerAlternativeWithSpan;
    private readonly SHA256 _hashAlgorithm;
    private readonly byte[][] _possibleKeys;

    public KeyComparisonBenchmark()
    {
        _possibleKeys = Enumerable.Range(0, 1_000)
            .Select(_ => Encoding.UTF8.GetBytes(SampleKey.Random().ToString()))
            .ToArray();
        
        // initializing supporting types...
    }
```

To understand if/how performance is impacted by the amount of different keys, as well as the number of times they are added to the dictionary, I included a couple of parameters to test this potential variation:

```csharp
[Params(100, 1_000)] public int TryAddCount { get; set; }  
  
[Params(100, 1_000)] public int PossibleKeysCount { get; set; }
```

Then we have the benchmark methods themselves. All of them follow the same recipe, with slight tweaks for each of the options being tested. Let's take on of the methods to briefly look at as an example (as they're all so similar, I won't paste the rest of them in the post, but you can check the sample code linked at the end):

```csharp
[Benchmark(Baseline = true)]
public void ByteArray()
{
    var possibleKeys = _possibleKeys.AsSpan(0, PossibleKeysCount);
    var map = new Dictionary<byte[], object>(_comparerWithLinq);
    for (var i = 0; i < TryAddCount; ++i)
    {
        var key = possibleKeys[i % possibleKeys.Count];
        _ = map.TryAdd(key, key);
    }
}
```

As we can see, the method starts by grabbing a subset of the possible keys, based on the parameter mentioned earlier. Then we create the dictionary and, in this case, pass in an `IEqualityComparer` implementation for `byte[]`. Finally, we iterate the number of times defined in the `TryAddCount` parameter, calling `TryAdd` on the dictionary with a key fetched from the possible keys.

In the next sections, we'll go through the various steps I went through, while exploring the various options. I won't include all the benchmark numbers, just a summary. If you're interested in everything, it's included in the gist linked at the end of the post, along with the rest of the code, if you feel like exploring.

## Initial approach: byte array with custom equality comparer

My initial approach was what seemed more obvious: if I want to use something that isn't, by default, prepared to be a key in a dictionary, implement a custom `IEqualityComparer<byte[]>`. So that was exactly what I did, and it worked as expected, but got me thinking if it had good performance (pushing me into a rabbit hole, culminating in this post).

The originally implemented `IEqualityComparer<byte[]>` looked like the following (the original name wasn't this though, it was renamed in the context of the benchmarks):

```csharp
private sealed class ByteArrayEqualityComparerWithLinq
    : IEqualityComparer<byte[]>  
{  
    public bool Equals(byte[]? x, byte[]? y)  
        => x is not null && y is not null  
            ? x.SequenceEqual(y)  
            : x is null && y is null;  
  
    public int GetHashCode(byte[] obj)   
        => obj.Aggregate(0, static (acc, value) => HashCode.Combine(acc, value));  
}
```

Before looking at the benchmark results, a brief high-level refresher on the relationship between the dictionary and the equality comparer. A dictionary is implemented as a hash map, so, when figuring out where an entry with a given key should be stored in its internal structure, the dictionary uses the `GetHashCode` method. Because using hashes means we may have collisions (i.e. different keys with the same hash code), the `Equals` method is used to ensure we're looking at the right entry.

Now onto benchmarks! Running things with this comparer, gives us these numbers:

| Method                            | TryAddCount | PossibleKeysCount | Mean         | Error      | StdDev     | Median       | Min          | Max          | Ratio | RatioSD | Rank | Gen0    | Gen1    | Allocated | Alloc Ratio |  
|---------------------------------- |------------ |------------------ |-------------:|-----------:|-----------:|-------------:|-------------:|-------------:|------:|--------:|-----:|--------:|--------:|----------:|------------:|
| ByteArray                         | 1000        | 1000              | 1,101.866 μs |  9.5614 μs |  8.4759 μs | 1,100.523 μs | 1,089.881 μs | 1,119.614 μs |  1.00 |    0.00 |   11 | 19.5313 |       - | 131.07 KB |        1.00 |

These numbers by themselves don't say much, we need to compare them with something, so let's get going to the next approach.

## Hash the byte array, but keep it in a byte array

Thinking about what other approaches I could look into, the next thing that came to mind was to hash the byte array, instead of using it directly. Originally I thought about hashing into a string, where the custom comparer wouldn't be needed, but while implementing it, I remembered that hashing with .NET's built-in hashing algorithms, like `SHA256`, returns a byte array as well, which we then need to turn into a string, so I decided to test first with the byte array hash, and later with the string (next section).

> Note: remember that as soon as we hash something, there's some likelihood of collisions, which depending on the context, might not be acceptable. In my particular use case, it's perfectly fine.

Even before implementing this approach, one thing was (almost) certain, this would cause more allocations, because we would be allocating a new byte array for the hash. The hope was that being a smaller byte array could have a positive impact, offsetting the extra heap allocations.

So, the implementation benchmark only differed from the first one, in that before adding the pair to the dictionary, we'd call the following method to hash the key:

```csharp
private byte[] HashKey(byte[] key) => _hashAlgorithm.ComputeHash(key);
```

The `_hashAlgorithm`, as you might have noticed earlier, is a `SHA256`.

Running this benchmark resulted in the following numbers:

| Method                            | TryAddCount | PossibleKeysCount | Mean         | Error      | StdDev     | Median       | Min          | Max          | Ratio | RatioSD | Rank | Gen0    | Gen1    | Allocated | Alloc Ratio |  
|---------------------------------- |------------ |------------------ |-------------:|-----------:|-----------:|-------------:|-------------:|-------------:|------:|--------:|-----:|--------:|--------:|----------:|------------:|
| Hash                              | 1000        | 1000              |   466.534 μs |  6.5156 μs |  6.0947 μs |   464.461 μs |   460.510 μs |   479.007 μs |  0.42 |    0.01 |    9 | 39.0625 |       - | 240.45 KB |        1.83 |

As we can see by comparing both, even though allocating more, almost twice as much, this approach ended up being more than twice as fast. Could the move to a string hash improve things?

## Hash the byte array into a string

If in the previous approach we were expecting more allocations, transforming the hash into a string would certainly worsen things in that department, as we'll add the string allocation on top of the byte array from the previous section. But could it still somehow payoff?

Comparing to the other benchmarks, the dictionary changes from a byte array key, to a string, and the hash is calculated as follows:

```csharp
private string HashKeyAsString(byte[] key)  
    => Convert.ToBase64String(_hashAlgorithm.ComputeHash(key));
```

These changes translate into the following numbers:

| Method                            | TryAddCount | PossibleKeysCount | Mean         | Error      | StdDev     | Median       | Min          | Max          | Ratio | RatioSD | Rank | Gen0    | Gen1    | Allocated | Alloc Ratio |  
|---------------------------------- |------------ |------------------ |-------------:|-----------:|-----------:|-------------:|-------------:|-------------:|------:|--------:|-----:|--------:|--------:|----------:|------------:|
| HashAsString                      | 1000        | 1000              |   179.712 μs |  0.8524 μs |  0.7974 μs |   179.688 μs |   178.793 μs |   181.397 μs |  0.16 |    0.00 |    7 | 51.7578 | 12.6953 | 318.57 KB |        2.43 |

As expected, the allocations gotten worse, more than doubling those of the original approach, but it still paid off, and it's even faster than the byte array hash.

## Trying to improve the hash string approach

We're getting better, but could we improve further in this hash string approach, at least in terms of allocations? If we could bypass that intermediate allocation into the byte array hash, which is immediately discarded as we transform it into a string, it'd be sweet!

With this in mind, the next idea that popped into my head was to pool the intermediate array. The overhead of pooling could possibly make it not really worthwhile, but at least it should reduce allocations, so I tried it.

The code for this looks like the following (abbreviated):

```csharp
public class KeyComparisonBenchmark  
{  
    private readonly DefaultObjectPool<byte[]> _byteArrayPool;

    public KeyComparisonBenchmark()  
    {  
        _byteArrayPool = new(new ByteArrayPoolPolicy());
    }

    private string HashKeyAsStringWithSpanAndPool(byte[] key)  
    {  
        var destination = _byteArrayPool.Get();  
        try  
        {  
            return _hashAlgorithm.TryComputeHash(key, destination, out _)  
                ? Convert.ToBase64String(destination)  
                : Throw();  
        }  
        finally  
        {  
            _byteArrayPool.Return(destination);  
        }  
        
        static string Throw() => throw new InvalidOperationException();  
    }

    private sealed class ByteArrayPoolPolicy : PooledObjectPolicy<byte[]>  
    {  
        public override byte[] Create() => new byte[SHA256.HashSizeInBytes];  
        
        // not worth clearing the array  
        public override bool Return(byte[] obj) => true;  
    }
}
```

As we can see, we create an object pool, with a policy to create byte arrays the size of a `SHA256`. This pool is then used when computing the hash, storing the intermediate step result in a pooled array.

The numbers for a benchmark run look like the following:

| Method                            | TryAddCount | PossibleKeysCount | Mean         | Error      | StdDev     | Median       | Min          | Max          | Ratio | RatioSD | Rank | Gen0    | Gen1    | Allocated | Alloc Ratio |  
|---------------------------------- |------------ |------------------ |-------------:|-----------:|-----------:|-------------:|-------------:|-------------:|------:|--------:|-----:|--------:|--------:|----------:|------------:|
| HashAsStringWithSpanAndPool       | 1000        | 1000              |   148.005 μs |  0.5539 μs |  0.5181 μs |   147.776 μs |   147.440 μs |   149.034 μs |  0.13 |    0.00 |    5 | 33.9355 | 11.2305 |  209.2 KB |        1.60 |

As we can see, we get improvements in terms of speed, though not as much, but it's not nothing 😁. Proving our expectation, allocations also reduced.

After implementing this, I remembered, we could maybe bypass the pool, trying to remove some overhead it might add, by using a `stackalloc`, to allocate the intermediate array on the stack, storing it in a `Span` (a [Span](https://learn.microsoft.com/en-us/dotnet/api/system.span-1?view=net-8.0) can reference a contiguous region of memory, which can be, for example, an array on the heap, or, in this case, some memory allocated on the stack). By allocating on the stack, the memory is automatically reclaimed when we return from the function, so we spare the GC some trouble. We should be careful allocating arrays on the stack, but being a small one, just to fit the `SHA256` shouldn't be an issue.

The code to implement this, looks like the following:

```csharp
private string HashKeyAsStringWithSpanAndStackAlloc(byte[] key)  
{  
    Span<byte> destination = stackalloc byte[SHA256.HashSizeInBytes];  
    return _hashAlgorithm.TryComputeHash(key, destination, out _)  
        ? Convert.ToBase64String(destination)  
        : Throw();  
  
    static string Throw() => throw new InvalidOperationException();  
}
```

It's similar to the previous one, but ignoring the pool and just allocating the array on the stack.

The benchmark results look like this:

| Method                            | TryAddCount | PossibleKeysCount | Mean         | Error      | StdDev     | Median       | Min          | Max          | Ratio | RatioSD | Rank | Gen0    | Gen1    | Allocated | Alloc Ratio |  
|---------------------------------- |------------ |------------------ |-------------:|-----------:|-----------:|-------------:|-------------:|-------------:|------:|--------:|-----:|--------:|--------:|----------:|------------:|
| HashAsStringWithSpanAndStackAlloc | 1000        | 1000              |   140.880 μs |  0.8028 μs |  0.7509 μs |   140.872 μs |   139.931 μs |   142.154 μs |  0.13 |    0.00 |    4 | 33.9355 | 11.2305 |  209.2 KB |        1.60 |

In terms of allocations, it's the same, as expected, but we do see an ever so slight improvement in speed. In a real application, I think (but didn't validate, so I may be thinking wrong) that there could be more contention when using the pool, due to concurrent access, which doesn't exist here.
## Back to the initial approach: trying to optimize things

So, I went through all this trouble, exploring alternatives to just shoving the original array as the key to the dictionary directly, but there was something bugging me. It felt weird that with the custom equality comparer, and with less allocations, we couldn't get at least closer to the numbers of the other approaches. So I decided to take another look at the custom equality comparer, to see if I could come up with some ways to improve it.

If you noticed in the original implementation, `GetHashCode` is using LINQ's `Aggregate` method. LINQ is typically one of the first things that is thrown out the window when we're aiming for performance, so I replaced it with a good old `for` loop, to check for any differences.

```csharp
private sealed class ByteArrayEqualityComparerWithoutLinqInHashCode
    : IEqualityComparer<byte[]>  
{  
    public bool Equals(byte[]? x, byte[]? y)  
        => x is not null && y is not null  
            ? x.SequenceEqual(y)  
            : x is null && y is null;  
  
    public int GetHashCode(byte[] obj)  
    {  
        var hash = 0;  
        for (var i = 0; i < obj.Length; ++i)  
        {  
            hash = HashCode.Combine(hash, obj[i]);  
        }  
        return hash;  
    }  
}
```

The change above resulted in these benchmark numbers:

| Method                            | TryAddCount | PossibleKeysCount | Mean         | Error     | StdDev    | Min          | Max          | Ratio | Rank | Gen0    | Gen1    | Allocated | Alloc Ratio |  
|---------------------------------- |------------ |------------------ |-------------:|----------:|----------:|-------------:|-------------:|------:|-----:|--------:|--------:|----------:|------------:|
| ByteArrayWithoutLinqInHashCode    | 1000        | 1000              |   903.790 μs | 4.3657 μs | 4.0837 μs |   897.684 μs |   912.625 μs |  0.82 |    9 | 15.6250 |  0.9766 |  99.82 KB |        0.76 |

There's some improvement, both in terms of faster execution speed and less allocations, but it's still far from the other approaches.

The next idea I had, was to poke around in the `HashCode` struct, to see if there was some alternative method that was more optimized to handle collections. While exploring `HashCode`, I noticed there was an instance method named `AddBytes`, that receives a `ReadOnlySpan` of bytes, which is basically the perfect fit for our use case. So I tweaked the code once again to check for some differences.

```csharp
private sealed class ByteArrayEqualityComparerAlternative
    : IEqualityComparer<byte[]>  
{  
    public bool Equals(byte[]? x, byte[]? y)  
        => x is not null && y is not null  
            ? x.SequenceEqual(y)  
            : x is null && y is null;  
  
    public int GetHashCode(byte[] obj)  
    {  
        var hashCode = new HashCode();  
        hashCode.AddBytes(obj);  
        return hashCode.ToHashCode();  
    }  
}
```

Back to our benchmark results, we got:

| Method                            | TryAddCount | PossibleKeysCount | Mean         | Error     | StdDev    | Min          | Max          | Ratio | Rank | Gen0    | Gen1    | Allocated | Alloc Ratio |  
|---------------------------------- |------------ |------------------ |-------------:|----------:|----------:|-------------:|-------------:|------:|-----:|--------:|--------:|----------:|------------:|
| ByteArrayAlternative              | 1000        | 1000              |    52.339 μs | 0.2579 μs | 0.2287 μs |    51.998 μs |    52.715 μs |  0.05 |    2 | 16.2354 |  1.3428 |  99.82 KB |        0.76 |

😱 Now this is an improvement! Allocations remain in line with the previous attempt, but the execution time decreased dramatically, even better than the other approaches. Don't know what kind of voodoo is being done in `HashCode` (I mean, it's open source, we can see it, I just didn't 😅), but it's fast!
## Community input: XxHash128 into an UInt128

As I was hacking away on this, I shared my progress on [Mastodon](https://mastodon.social/@joaofbantunes/111517843879194369), where [Alexandre Mutel chimed in](https://mastodon.social/@xoofx/111517880052348254), mentioning that a recommended approach to this problem is indeed to hash the byte array, but into a numeric value, like `UInt128`. This recommendation made perfect sense, as we're hashing like in other approaches with which we got some good results, but using a struct (`UInt128`) instead of a reference type (as is the case of a byte array or string), so hopefully minimizing allocations.

Following Alexandre's recommendation, I added a reference to the `System.IO.Hashing` package, to use the `XxHash128` type to generate an `UInt128` hash of the byte array.

```csharp
private static UInt128 HashKeyAsUInt128(byte[] key)
    => XxHash128.HashToUInt128(key);
```

This implementation got me the following results:

| Method                            | TryAddCount | PossibleKeysCount | Mean         | Error     | StdDev    | Min          | Max          | Ratio | Rank | Gen0    | Gen1    | Allocated | Alloc Ratio |  
|---------------------------------- |------------ |------------------ |-------------:|----------:|----------:|-------------:|-------------:|------:|-----:|--------:|--------:|----------:|------------:|
| HashAsUInt128                     | 1000        | 1000              |    37.738 μs | 0.2449 μs | 0.2291 μs |    37.422 μs |    38.169 μs |  0.03 |    1 | 20.8130 |  2.2583 | 128.19 KB |        0.98 |

As we can see, this is the fastest option of them all. It allocates more than the last version of the custom equality comparer, but still manages to reduce the execution time, pretty cool.

## Outro

That's a wrap for this post. Even though these performance topics aren't a particular focus of mine, I hope I did a good enough job of explaining my thought process regarding this particular adventure.

We took a look at:

- How to use a byte array (or something derived from it) as a dictionary key
- Implementing custom equality comparers to allow any type to be used as a dictionary key
- Using object pools to minimize heap allocations
- Using `stackalloc` to allocate memory on the stack, minimizing heap allocations
- How LINQ can often have some impact on performance
- The `HashCode` struct to help in calculating traditional .NET hash codes
- `XxHash128` to calculate an `UInt128` hash code

Before finishing, as promised in the intro, here's a screenshot with the full results table (almost didn't fit in a single screen 😅):

{{< embedded-image "/images/2023/12/06/full-benchmark-results.png" "Full benchmarks results" >}}

The numbers might be ever so slightly different, because I reran the benchmarks again to capture the screenshot.

Relevant links:

- [Sample code](https://gist.github.com/joaofbantunes/be5fac1adfb21d126f2836f356800acf)
- [BenchmarkDotNet](https://benchmarkdotnet.org)
- [IEqualityComparer](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iequalitycomparer-1?view=net-8.0)
- [stackalloc](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/stackalloc)
- [Span](https://learn.microsoft.com/en-us/dotnet/api/system.span-1?view=net-8.0)
- [HashCode](https://learn.microsoft.com/en-us/dotnet/api/system.hashcode?view=net-8.0)

Thanks for stopping by, cyaz! 👋
