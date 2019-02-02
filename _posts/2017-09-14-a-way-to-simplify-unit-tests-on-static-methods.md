---
author: João Antunes
comments: true
date: 2017-09-14 18:08:49+00:00
layout: post
slug: a-way-to-simplify-unit-tests-on-static-methods
title: A way to simplify unit tests on static methods
categories:
- dotnet
- testing
tags:
- dotnet
- mock
- static
- testing
---

## Intro


This post will be simple and try to act as a discussion starter on unit testing static methods.

I'm not the greatest fan of making static stuff, mainly because I've been burned by it in the past (the fact is that static was being used badly in those cases, but even so, if I can, I'll avoid it) but I understand it's the best way to do things sometimes.

One of the problems that might arise when using static methods is how to test them (and test the code that uses them, but this is even a bigger problem that I've not seen a great solution for).

When our methods are simple and, mainly, don't use any shared state, we're good, we can just test them as any other method. But a problem arises when we use shared state in a static method: when we have multiple unit tests, given test frameworks may execute them in parallel, how do we guarantee one test doesn't mess with another?

The idea I use (not my idea, I've read about it some years ago, and recently saw a Twitter thread talking about such issues and thought of writing this post) is to implement the required logic in a "normal" non static class, that we can design as usual, with DI as we .NET developers love so much :P Then we implement a static class that acts as a wrapper over the class that contains the logic, having a private instance of the latter and forwarding any calls to it.


## Sample


Let's go for an example (way too simple, but it think it's enough): I want to implement an extension method on `int` that returns a `string` with the length equal to the `int` - `string GetStringWithNLength(this int n)`. Now for some reason I don't want to create a new `string` every time, and want to cache the results. Here we have our shared state.

So, as I mentioned, I'd start by creating a non-static class to hold the logic.

{% highlight csharp linenos %}
using System;

namespace CodingMilitia.UnitTestingStaticsSample.Library
{
    public class CacheableStuffCalculator
    {
        private readonly ICache<int,string> _cache;

        public CacheableStuffCalculator(ICache<int,string> cache)
        {
            _cache = cache;
        }

        public string GetStringWithNLength(int n)
        {
            return _cache.GetOrAdd(n, nAgain => new string('n', nAgain));
        }
    }
}
{% endhighlight %}

Then we implement the static wrapper, that knows how to construct the `CacheableStuffCalculator` and provide its dependencies, and then forward any calls to it.

{% highlight csharp linenos %}
namespace CodingMilitia.UnitTestingStaticsSample.Library
{
    public static class StaticCacheableStuffCalculatorWrapper
    {
        private static readonly CacheableStuffCalculator CalculatorInstance;

        static StaticCacheableStuffCalculatorWrapper()
        {
            CalculatorInstance = new CacheableStuffCalculator(new SampleCache<int,string>());
        }

        public static string GetStringWithNLength(this int n)
        {
            return CalculatorInstance.GetStringWithNLength(n);
        }
    }
}
{% endhighlight %}

With this in place, it's pretty easy to make our unit tests on `CacheableStuffCalculator` as we usually do, avoiding problems with shared state.

{% highlight csharp linenos %}
using System;
using Xunit;
using Moq;

namespace CodingMilitia.UnitTestingStaticsSample.Library.Tests
{
    public class GetStringWithNLengthTest
    {
        [Fact]
        public void GivenLength1ReturnsStringWithLength1()
        {
            //prepare
            var cacheMock = new Mock<ICache<int, string>>();

            cacheMock.Setup(cache => cache.GetOrAdd(It.IsAny<int>(), It.IsAny<Func<int, string>>()))
                .Returns((int key, Func<int, string> valueProvider) => valueProvider(key));

            var calculator = new CacheableStuffCalculator(cacheMock.Object);

            //execute
            var result = calculator.GetStringWithNLength(1);

            //assert
            Assert.Equal(1, result.Length);
        }
    }
}
{% endhighlight %}


## Wrapping up


And that's about it. I think it's pretty simple and a nice solution. So much so, I thought it was the usual way people did this kind of thing. But as I mentioned, I saw a thread on Twitter recently debating on how to do unit tests in these kinds of scenarios, and thought of putting this on a post.

So, how would you do it? Does this seem like a nice approach or you know of better ways? Share your thoughts!

Cyaz

PS: sample code [here](https://github.com/joaofbantunes/UnitTestingStaticsSample)
