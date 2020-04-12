---
author: João Antunes
date: 2019-10-25 18:00:00+01:00
layout: post
title: "Episode 031 - Some simple unit tests with xUnit - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we'll take a little break from the explorations we've been doing in the latest episodes, getting back to fundamentals and writing some really simple unit tests with xUnit."
images:
- '/assets/2019/10/25/aspnet-core-from-zero-to-overkill-e031.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
- xunit
slug: aspnet-031-from-zero-to-overkill-some-simple-unit-tests-with-xunit
---

In this episode, we'll take a little break from the explorations we've been doing in the latest episodes, getting back to fundamentals and writing some really simple unit tests with xUnit.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< yt ntNL2V45RtQ >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In the past few episodes we explored a bunch of more complex topics like IdentityServer, Docker, ProxyKit or BenchmarkDotNet, also ending up with some longer posts and videos, so I thought in this one we could take a little break and get back to some fundamentals that we've been ignoring until now.

On that note, we'll use [xUnit](https://xunit.net/) and write some unit tests for the `ProxiedApiRouteEndpointLookup` we've played around with in the last episode. We're not going into much complexity, if for nothing else, because the class we'll be testing is rather simple. In future episodes, with more complex code to test, we'll revisit the topic and look at some other testing possibilities/needs (e.g. mocking).

### What we'll be testing
Before starting, just as a quick refresher of what the `ProxiedApiRouteEndpointLookup` class does, to make it easier to understand the tests we're going to write.

The `ProxiedApiRouteEndpointLookup` class gets in the constructor a dictionary that maps the base path of an API our BFF exposes, to the base endpoint of a backing service it will forward to.

When we call the `TryGet` method with the path of the request we're processing, the `ProxiedApiRouteEndpointLookup` will try to match that path to a configured backing API. If it finds a match, it returns `true` and fills the `out` parameter with the endpoint to forward to, otherwise it'll return `false`.

## Creating the test project
To get started, we need to create a project to put our test in. Next to the `src` folder we can create a new `tests` folder, in which we'll then create a new project. We can use the IDE to do it, but as usual I'll do it from the command line:

```sh
# inside the newly created tests folder
dotnet new xunit -o CodingMilitia.PlayBall.WebFrontend.BackForFront.Web.Test
cd ..
dotnet sln add tests\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web.Test\CodingMilitia.PlayBall.WebFrontend.BackForFront.Web.Test.csproj
```

Now if we head to the IDE we should see the newly created project, so we can get to writing some tests.

## The first test
Let's write our first really simple test. For starters let's create a class to put it in. 

We'll start by creating a new folder `ApiRouting`, to match the structure of the Web project, then creating a `ProxiedApiRouteEndpointLookupTests` class.

Now that we have the class, we can write some test methods in there. Let's start with a really simple scenario: if we provide an empty route to endpoint map in the `ProxiedApiRouteEndpointLookup` constructor, then when we call `TryGet` we should not have a match, regardless of the route we pass it.

`ApiRouting\ProxiedApiRouteEndpointLookupTests.cs`
```csharp
public class ProxiedApiRouteEndpointLookupTests
{
    [Fact]
    public void WhenUsingAnEmptyLookupDictionaryThenNoRouteIsMatched()
    {
        var lookup = new ProxiedApiRouteEndpointLookup(new Dictionary<string, string>());

        var found = lookup.TryGet("/non-existent-route", out _);

        Assert.False(found);
    }
}
```

The test method is called `WhenUsingAnEmptyLookupDictionaryThenNoRouteIsMatched`, because I like to make it clear what the test does. Sometimes this causes the name to be massive, but being a test I don't really care because I'm not going to call the method directly anyway. Some people separate the words with underscores, I normally don't just because it looks weird, but I'll grant you that it makes it more readable in these cases with large names.

For xUnit to know that we want it to run that method as part of the test suite, we mark it with the `Fact` attribute.

Now for the test code itself, we start by creating a new instance of `ProxiedApiRouteEndpointLookup` with the empty dictionary parameter. Notwithstanding the fact that the class is simple, making the tests easy to implement, being able to pass in the dependency, in this case the route to endpoint map configuration, simplifies things further. If the class was auto-magically fetching the configuration from somewhere would make testing it tricker. That's one of the main reasons dependency injection is so useful.

With the sometimes called SUT (subject under test) instance in hand, we can execute the method we actually want to test, `TyeGet`, by passing it some route. As we can see, we can use the discard `_` on the `out` parameter, as we're not expecting it to be correctly hydrated. We store the result of `TryGet` so we can assert that it's the value we expect.

Finally we assert that `TryGet` didn't find any match, by ensuring that `found` is `false`.

Now we can run the test in the IDE or in the console by running `dotnet test`.

## Checking for thrown exceptions
Since we're testing behavior given an empty route map configuration, what about if we pass it a `null` dictionary? It should throw an exception, as we'd rather not deal with `null`s (and even allowing for an empty configuration is debatable but, let's get on with it).

We could write a test that called the constructor with `null` wrapped in a `try` `catch` block, then assert that the exception type is what we expected, but given this is a common type of test to implement, xUnit has specific assertion methods for that.

`ApiRouting\ProxiedApiRouteEndpointLookupTests.cs`
```csharp
[Fact]
public void WhenProvidingANullDictionaryThenTheConstructorThrowsArgumentNullException()
{
    Assert.Throws<ArgumentNullException>(() => new ProxiedApiRouteEndpointLookup(null));
}
```

As we can see, there's an `Assert.Throws<T>`, that gets an action as parameter (plus other overloads) that'll be executed expecting an exception of the indicated type to be thrown. If no exception (or the wrong type) is thrown, then the test will fail.

## Executing the same test with varying data
Sometimes the same test code is able to validate multiple similar scenarios, just by varying some parameters. This is something we could do using regular programming strategies, extracting the common code into auxiliary methods and then call them from multiple test methods, but being a common need, test frameworks like xUnit provide capabilities such cases.

Lets test some scenarios in which we provide routes that don't exist in our configuration, so we expect the result to be no match found.

`ApiRouting\ProxiedApiRouteEndpointLookupTests.cs`
```csharp
[Theory]
[InlineData("/non-existent-route")]
[InlineData("/test-route-almost")]
[InlineData("")]
[InlineData(null)]
public void WhenLookingUpANonExistentRouteThenNothingIsFound(string nonExistentRoute)
{
    var lookup = new ProxiedApiRouteEndpointLookup(new Dictionary<string, string>
    {
        ["test-route"] = "test-endpoint",
        ["another-test-route"]= "another-test-endpoint"
    });

    var found = lookup.TryGet(nonExistentRoute, out _);

    Assert.False(found);
}
```

Instead of decorating the method with the `Fact` attribute, we use the `Theory` attribute, which means that the test will be executed against varying data. One way to provide such test data to the test is by using the `InlineData` attribute, in which we can pass data that will be fed as parameters to the test method.

The `Fact` decorated test methods get no parameters, but in this case, we set a `nonExistentRoute` parameter that will come from the data we included in the `InlineData` attribute. Each `InlineData` instance will correspond to one test, and we'll see these multiple instances appear in the results after we run the tests.

Regarding the data we're providing, we're looking to test four scenarios:
- a non-existent route
- another non-existent route, but in which the start is the equal to an existing route
- an empty route
- a `null` route (we could throw in such case but, being a `TryGet`, we might as well be a little more permissive)

The test code itself is similar to the first test we wrote, but in this case we provide some actual routes and endpoints in the configuration dictionary, and then call `TryGet` with the parameter provided by the `InlineData`. As previously, we expect the result to be no match found.

Using the same strategy, we can create a test to check if an existing route is found.

`ApiRouting\ProxiedApiRouteEndpointLookupTests.cs`
```csharp
[Theory]
[InlineData("/test-route", "test-endpoint")]
[InlineData("/another-test-route/some/more/segments", "another-test-endpoint")]
public void WhenLookingUpExistingRouteThenItIsFound(string route, string expectedEndpoint)
{
    var lookup = new ProxiedApiRouteEndpointLookup(
        new Dictionary<string, string>
        {
            ["test-route"] = "test-endpoint",
            ["another-test-route"]= "another-test-endpoint"
        });

    var found = lookup.TryGet(route, out var endpoint);

    Assert.True(found);
    Assert.Equal(expectedEndpoint, endpoint);
}
```

The test is very similar to the previous one, with some important differences:
- `InlineData` now provides two parameters, one for the route to find and another with the expected resulting endpoint
- We assert that the route was found
- We assert that the resulting endpoint is the one expected

The data we provide tests two scenarios, the simplest one in which the match is exact (minus the prefixing `/`) and another to ensure that even if the request path has additional segments, the route is matched taking its base into consideration, not the full path.

## Outro
That does it for our quick look at xUnit and some simple test scenarios. I'm pretty sure we could break the `ProxiedApiRouteEndpointLookup` class if we tried a little harder, so it's probably a good idea to write some more tests, by thinking about plausible corner cases that we're not considering at this moment.

Anyway, for a quick intro, but most of all, for a reminder that tests are really important, it should be good enough for now. If this was production code, we should really work harder on the tests, but given this is an exploration project, tests end up not being as interesting to focus on.

Links in the post:
- [xUnit](https://xunit.net/)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode031).

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!