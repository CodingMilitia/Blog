---
author: João Antunes
date: 2024-04-13 17:00:00+01:00
layout: post
title: Primitive vs strongly typed dictionary keys (feat. frozen collections and BenchmarkDotNet)
summary: Quick post comparing the usage of different types as dictionary keys, including using custom types instead of only primitive types.
images:
  - /images/2024/04/13/primitive-vs-strongly-typed-dictionary-keys-feat-frozen-collections-and-benchmarkdotnet.png
categories:
  - csharp
  - dotnet
tags:
  - performance
  - benchmarkdotnet
  - frozencollections
slug: primitive-vs-strongly-typed-dictionary-keys-feat-frozen-collections-and-benchmarkdotnet
---
## Intro

As I messed around with the [frozen collections](https://learn.microsoft.com/en-us/dotnet/api/system.collections.frozen?view=net-8.0) recently added to .NET, namely `FrozenDictionary` and `FrozenSet`, I wondered: if the idea of these new collections is to optimize things at construction time, so subsequent reads are faster, does the key used influence their ability to optimize? In particular, would the usage of custom types, given I tend to try and avoid primitive obsession, have an impact on the potential for optimizations?

This very brief post goes into a quick benchmark I put up to try to get some answers to these questions.

The usual disclaimer applies: my typical focus isn't performance critical code or lower level details, so keep in mind I might mess something up (hopefully not 🤞). If you're into .NET performance topics, you should really look into content from [Stephen Toub](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/), [Alexandre Mutel](https://xoofx.github.io), [Antão Almada](https://aalmada.github.io), [Konrad Kokosa](https://tooslowexception.com) and other such legends.

## Setup

The setup isn't super complex, but it's also not the most straightforward thing, as I wanted to test multiple scenarios. As an example, I started focused on `string`s, as it's probably the type I see used most often as a dictionary key, but thought, given it's so common, maybe there are special optimizations specifically for it, and for a better understanding, maybe it's relevant to try things out with other types as well (PS: a quick look at the [source code](https://github.com/dotnet/runtime/blob/4e702bc40f15a5bae8ef458cc1e9c924c44c3a54/src/libraries/System.Collections.Immutable/src/System/Collections/Frozen/FrozenDictionary.cs#L113) confirms the suspicions that there are optimized special cases).

So, in summary, the variations tested:

- Different types of keys: `string`, `int` and `Guid`, as well as wrapped versions of them (i.e. custom types that in a real application would represent specific domain concepts)
  - The wrappers were implemented as readonly record structs
- Various amounts of entries - 1, 10, 100, 1000 and 10000
- Dictionaries and frozen dictionaries
- When using the type wrappers, test with the default equality comparer, as well as with a custom one

The wrapper looks like this:

```csharp
public readonly record struct StronglyTypedKey<T>(T Value);
```

The base benchmark implementation looks like this:

```csharp
public class BaseImplementation<T> where T : notnull  
{  
    private readonly T _toLookup;  
    private readonly StronglyTypedKey<T> _toLookupStronglyTyped;  
    private readonly Dictionary<T, string> _traditional;  
    private readonly Dictionary<StronglyTypedKey<T>, string> _traditionalWithStronglyTypedKey;  
    private readonly Dictionary<StronglyTypedKey<T>, string> _traditionalWithStronglyTypedKeyWithCustomComparer;  
    private readonly FrozenDictionary<T, string> _frozen;  
    private readonly FrozenDictionary<StronglyTypedKey<T>, string> _frozenWithStronglyTypedKey;  
    private readonly FrozenDictionary<StronglyTypedKey<T>, string> _frozenWithStronglyTypedKeyWithCustomComparer;  
  
    public BaseImplementation(
      int n,
      IEqualityComparer<StronglyTypedKey<T>> equalityComparer,
      Func<int, T> factory)  
    { 
      var contents = Enumerable.Range(0, n).Select(factory).ToArray();  
      _toLookup = contents.Last();  
      _toLookupStronglyTyped = new StronglyTypedKey<T>(_toLookup);  
      _traditional = contents.ToDictionary(x => x, x => x.ToString()!);  
      _traditionalWithStronglyTypedKey = contents.ToDictionary(x => new StronglyTypedKey<T>(x), x => x.ToString()!);  
      _traditionalWithStronglyTypedKeyWithCustomComparer
        = contents.ToDictionary(x => new StronglyTypedKey<T>(x), x => x.ToString()!, equalityComparer);  
      _frozen = _traditional.ToFrozenDictionary();  
      _frozenWithStronglyTypedKey = _traditionalWithStronglyTypedKey.ToFrozenDictionary();  
      _frozenWithStronglyTypedKeyWithCustomComparer 
        = _traditionalWithStronglyTypedKey.ToFrozenDictionary(equalityComparer);
    }
    
    public string LookupTraditional() => _traditional[_toLookup];  
  
    public string LookupTraditionalWithStronglyTypedKey() => _traditionalWithStronglyTypedKey[_toLookupStronglyTyped];  
  
    public string LookupTraditionalWithStronglyTypedKeyWithCustomComparer()  
        => _traditionalWithStronglyTypedKeyWithCustomComparer[_toLookupStronglyTyped];  
  
    public string LookupFrozen() => _frozen[_toLookup];  
  
    public string LookupFrozenWithStronglyTypedKey() => _frozenWithStronglyTypedKey[_toLookupStronglyTyped];  
  
    public string LookupFrozenWithStronglyTypedKeyWithCustomComparer()  
        => _frozenWithStronglyTypedKeyWithCustomComparer[_toLookupStronglyTyped];  
}
```

## Results

Onto the results!

Given the amount of scenarios I setup to benchmark, the results output isn't small, so I'll just summarize the findings and include a couple of extra highlights here. For the full results, you can check out the [GitHub repo](https://github.com/joaofbantunes/LooseBenchmarks/tree/main/src/PrimitiveVsStronglyTypedKeyLookup).

Even though there's a bit of variation depending on the amount of entries in the dictionary, let's use the 100 entries configuration as reference, as it does seem to represent results overall, starting with `string`s.

### string

| Method                                                        | N     | Mean       | Error     | StdDev    | Ratio | RatioSD | Rank |
|-------------------------------------------------------------- |------ |-----------:|----------:|----------:|------:|--------:|-----:|
| LookupTraditionalString                                       | 100   | 11.9771 ns | 0.0137 ns | 0.0122 ns |  1.00 |    0.00 |    2 |
| LookupTraditionalWithStronglyTypedKeyString                   | 100   | 30.8755 ns | 0.1421 ns | 0.1187 ns |  2.58 |    0.01 |    6 |
| LookupTraditionalWithStronglyTypedKeyStringWithCustomComparer | 100   | 27.1268 ns | 0.1055 ns | 0.0881 ns |  2.26 |    0.01 |    4 |
| LookupFrozenString                                            | 100   |  4.7020 ns | 0.0071 ns | 0.0059 ns |  0.39 |    0.00 |    1 |
| LookupFrozenWithStronglyTypedKeyString                        | 100   | 29.2874 ns | 0.2304 ns | 0.2155 ns |  2.44 |    0.02 |    5 |
| LookupFrozenWithStronglyTypedKeyStringWithCustomComparer      | 100   | 26.6226 ns | 0.2129 ns | 0.1992 ns |  2.22 |    0.02 |    3 |

As we can see clearly with these results, using a wrapper type around the `string` degrades the lookup performance, both for traditional and frozen dictionaries alike.

As expected, direct `string` lookup is faster when using the frozen dictionary, versus the traditional one. However, when the strongly typed key comes into play, there's virtually no difference between the two types of dictionaries, so it seems the optimizations done by frozen dictionaries don't translate well to custom types (maybe there's a way to tailor a custom type to play better with these optimizations? not sure, didn't investigate further).

Finally, a quick mention to the usage of the custom equality comparer. Across all benchmarks with number of entries larger than one, it was consistently faster than the default equality comparer for the custom key, though the difference isn't particularly impressive, as we can see.

Now let's take a brief look at the results for `int`.

### int

| Method                                                     | N   |      Mean |     Error |    StdDev | Ratio | RatioSD | Rank |
| ---------------------------------------------------------- | --- | --------: | --------: | --------: | ----: | ------: | ---: |
| LookupTraditionalInt                                       | 100 | 3.1269 ns | 0.1041 ns | 0.1157 ns |  1.00 |    0.00 |    4 |
| LookupTraditionalWithStronglyTypedKeyInt                   | 100 | 2.9621 ns | 0.0774 ns | 0.0724 ns |  0.95 |    0.03 |    3 |
| LookupTraditionalWithStronglyTypedKeyIntWithCustomComparer | 100 | 3.3904 ns | 0.0658 ns | 0.0616 ns |  1.09 |    0.04 |    5 |
| LookupFrozenInt                                            | 100 | 1.3530 ns | 0.0182 ns | 0.0171 ns |  0.44 |    0.02 |    1 |
| LookupFrozenWithStronglyTypedKeyInt                        | 100 | 2.2000 ns | 0.0214 ns | 0.0190 ns |  0.71 |    0.02 |    2 |
| LookupFrozenWithStronglyTypedKeyIntWithCustomComparer      | 100 | 3.1473 ns | 0.0303 ns | 0.0283 ns |  1.01 |    0.04 |    4 |

Looking at these results, there are some similarities when comparing with `string`, but also some differences.

For starters, in terms of differences, all the results are closer for `int` than for `string`. Also, it seems the usage of a custom type key isn't as impactful, with the results for the traditional dictionary being very close, and the results for the frozen dictionary showing a bit more distance, but still not as clear as the numbers for `string`s. Additionally, while the custom equality comparer marginally improved the numbers in the case of the `string`s, it worsens them for `int`s.

The only major similarity between the `string` and `int` seems to be that in both cases, for non-wrapped values, the frozen dictionary improved performance over the traditional dictionary.

Let's now see how `Guid` compares to `string` and `int`.

### Guid

| Method                                                      | N   |      Mean |     Error |    StdDev | Ratio | RatioSD | Rank |
| ----------------------------------------------------------- | --- | --------: | --------: | --------: | ----: | ------: | ---: |
| LookupTraditionalGuid                                       | 100 | 3.6592 ns | 0.0609 ns | 0.0570 ns |  1.00 |    0.00 |    1 |
| LookupTraditionalWithStronglyTypedKeyGuid                   | 100 | 4.1773 ns | 0.0143 ns | 0.0119 ns |  1.14 |    0.02 |    3 |
| LookupTraditionalWithStronglyTypedKeyGuidWithCustomComparer | 100 | 4.4013 ns | 0.0726 ns | 0.0679 ns |  1.20 |    0.01 |    4 |
| LookupFrozenGuid                                            | 100 | 3.8696 ns | 0.1064 ns | 0.0944 ns |  1.06 |    0.03 |    2 |
| LookupFrozenWithStronglyTypedKeyGuid                        | 100 | 3.6643 ns | 0.0985 ns | 0.0922 ns |  1.00 |    0.03 |    1 |
| LookupFrozenWithStronglyTypedKeyGuidWithCustomComparer      | 100 | 4.6254 ns | 0.0144 ns | 0.0120 ns |  1.26 |    0.02 |    5 |

Overall, the results are close to what we saw for `int`, with the results being very close, without too much of an impact of using a custom type key, though there's something in there, which gets worse when adding a custom equality comparer into the mix.

Now where `Guid` differs from both `string` and `int`, is that there doesn't seem to be an advantage of using a frozen dictionary over a traditional one, as the results are pretty much the same.

Before wrapping up, a couple of worthy mentions from the results.

### Other notes

- For 100, 1000 and 10000 entries, the results follow a similar pattern as described earlier
- For 10 entries, there are various situations where the frozen dictionary performs worse than the traditional one
- For 1 entry, a frozen dictionary, particularly for the custom type key scenario, has a more pronounced reduction in execution time when compared to the traditional dictionary. This improvement is likely due to the special case implemented for dictionaries with fewer than 10 entries

## Outro

That does it for this quick look at how some different types compare as dictionary keys in terms of lookup performance, namely `string`, `int` and `Guid`, as well as wrapped versions of them.

We saw some expected results, like `string` and `int` key lookup performing better with frozen vs traditional dictionaries, some more unexpected results, like `Guid` performance being mostly the same regardless of dictionary type, and also some neither expected or unexpected results, at least for me, regarding the impact of using custom types as dictionary keys (I had no idea what to expect, hence this whole investigation 😅).

What's always an interesting exercise, is that while creating these benchmarks and analyzing the results, also took the opportunity to look into the runtime's source code, particularly that of the frozen dictionary, to better understand why some things behaved the way they did. For example, looking into the code we can see the existence of special cases dependent on different factors, like dictionary size, known types, specialized versions for `string`s as well as for `int`s, and much more.

Another important thing to remember, is that, as you might notice from the resulting numbers, we're in the nanoseconds order of magnitude, which is very very fast. I imagine that in the vast majority of cases, application developers don't need to worry about this, but if there's a piece of code in the hot path, maybe it can be worth taking a look. Also, even if it isn't worth it, it's at least fun to learn about these things 🙂.

Relevant links:

- [Source code](https://github.com/joaofbantunes/LooseBenchmarks/tree/main/src/PrimitiveVsStronglyTypedKeyLookup)
- [Frozen Collections source code](https://github.com/dotnet/runtime/tree/4e702bc40f15a5bae8ef458cc1e9c924c44c3a54/src/libraries/System.Collections.Immutable/src/System/Collections/Frozen)
- [FrozenDictionary on .NET Source Browser](https://source.dot.net/#System.Collections.Immutable/System/Collections/Frozen/FrozenDictionary.cs) (easier to poke around than through GitHub web site)

Thanks for stopping by, cyaz! 👋
