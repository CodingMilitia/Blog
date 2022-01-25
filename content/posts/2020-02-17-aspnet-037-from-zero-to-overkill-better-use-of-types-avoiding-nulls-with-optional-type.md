---
author: João Antunes
date: 2020-02-17 18:00:00+00:00
layout: post
title: "Better use of types - avoiding nulls with an Optional type [ASPF02O|E037]"
summary: "In this post, we'll make use of a concept most commonly associated with functional programming, the Optional type (aka Option or Maybe), in order to make our code safer and more explicit when expressing a lack of value, instead of leaning on the null reference, something that I'm sure has burned us many times in the past."
images:
- '/images/2020/02/17/e037.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-037-from-zero-to-overkill-better-use-of-types-avoiding-nulls-with-optional-type
---

In this post, we'll make use of a concept most commonly associated with functional programming, the Optional type (aka Option or Maybe), in order to make our code safer and more explicit when expressing a lack of value, instead of leaning on the null reference, something that I'm sure has burned us many times in the past.

**Note:** depending on your preference, you can check out the following video, otherwise, skip to the written version below.

{{< yt y-VmliKd2Ig >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

One of the things that more often catches us by surprise is the `NullReferenceException`, not only in C#, but also in other similar languages. Some other languages, particularly functional languages, use a different approach to represent the absence of a value.

A popular (and safer) approach to representing the possibility of a value not being present is to use a specific type to represent it, like an `Optional` type, also known as `Option` or `Maybe`, depending on the language.

The goal of using an `Optional` type is twofold: not only it makes the absence of value explicit, forcing us to deal with it, as in C# is far from impossible to forget the `null` check, but with some auxiliary methods, we can sometimes even bypass checking for the existence of value, instead writing more linear code, following some functional programming principles.

In this episode, we'll explore using such a type instead of relying on `null`. We'll implement a simple version of an `Optional` type, to understand what's going on, but there are some NuGet packages already available, so you might be interested in checking them out (e.g. [Optional](https://github.com/nlkl/Optional) or [Functional Extensions for C#](https://github.com/vkhorikov/CSharpFunctionalExtensions))

We'll continue using the [group management project](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode037) to play around with these concepts.

## Simple implementation

Let's begin by creating a simple implementation of an `Optional` type. We'll put it in a `Shared` folder inside the `Domain` project, just for simplicity, but it should probably be extracted to a different project, as it's a very generic concept, not tied to the domain logic.

I'll drop the code right now, so we can go through it afterwards.

`Domain\Shared.Optional.cs`
```csharp
public struct Optional<T>
{
    private readonly bool _hasValue;
    private readonly T _value;

    public bool HasValue => _hasValue;

    internal Optional(T value, bool hasValue)
    {
        _value = value;
        _hasValue = hasValue;
    }

    public bool TryGetValue(out T value)
    {
        value = _hasValue ? _value : default;
        return _hasValue;
    }
}
```

As you can see from the size alone, really not that much going on. Regardless of complexity though, I'd say just the fact that we use a type to represent the possible absence of value is the important part. Too many times we seem to avoid creating new types when it's a great way to make our code more expressive.

Starting with the fact that we're declaring `Optional` as a `struct` instead of a `class`, there are two main reasons. The first reason is that given `Optional` acts as a wrapper around another type (plus a boolean), we avoid always allocating another object on the heap, considering its small footprint is acceptable for a `struct` passed around by copy. The second, probably more interesting reason, is that by being a `struct`, it cannot be `null`, so we never even have to worry with `null` checks on it.

In the type, we keep two fields, a boolean indicating if there exists a value, and the actual value, which will contain the default for that type in situations where `_hasValue` is `false`.

We're exposing `HasValue` as a property, to simplify cases when checking for the presence of the value is enough.

The constructor is as basic as it could be, getting the parameters to hydrate the private read-only fields.

To wrap it up, we have a `TryGetValue` method, which provides the only way to access the value. This is a very purposeful decision, as if we exposed the value directly as a property, we could end up in the same situation as not having `Optional` in the first place, as we could do `.Value` and be greeted with an exception. To get the value, we must use this `TryGetValue` method, which follows a similar pattern to parsing methods we can find for instance in numeric types. This forces the client of this API to consider the two cases: presence and absence of value.

In addition to the `Optional` type itself, we can create some static methods to help with creating instances of it.

`Domain\Shared.Optional.cs`
```csharp
public static class Optional
{
    public static Optional<T> Some<T>(T value) => new Optional<T>(value, true);

    public static Optional<T> None<T>() => new Optional<T>(default(T), false);

    public static Optional<T> FromNullable<T>(T value) where T : class
        => value is null
            ? None<T>()
            : Some(value);

    public static Optional<T> FromNullable<T>(T? value) where T : struct
        => value.HasValue
            ? Some(value.Value)
            : None<T>();
}
```

The two first methods should be the most commonly used ones, to create an `Optional` instance when there is a value (`Some`) and when there isn't (`None`).

The other two methods, `FromNullable`, should be most useful on the edges of our system, when we integrate with code that doesn't use this approach. An example can be when integrating with EF Core, finding an item by id that might not exist. From our abstraction (e.g. a repository) we want to expose this operation as returning an `Optional`, but EF doesn't use this, so instead of manually doing the `null` check in every situation like this, we can just wrap the result from the EF query with a call to `Optional.FromNullable`.

## Exposing it in the APIs

Now that we have an `Optional` type to play around with, let's start by exposing it in our APIs.

The first place we can do this is in our query handlers (the things we're using as single operation repositories). For example, the `UserByIdQuery`, which might not find the user for the given id. We can change the class definition to `public class UserByIdQuery : IQuery<Optional<User>>`, signaling that a user might not be found. Then the handler implementation could be the following:

`Infrastructure\Data\Queries\UserByIdQueryHandler.cs`
```csharp
public class UserByIdQueryHandler : IQueryHandler<UserByIdQuery, Optional<User>>
{
    // ...

    public async Task<Optional<User>> HandleAsync(UserByIdQuery query, CancellationToken ct)
        => Optional.FromNullable(await _db.Set<User>().FindAsync(new object[] {query.UserId}, ct));
}
```

Other places we can adapt the exposed API, are the use case implementations. In a very similar example, the `UpdateGroupDetailsCommandHandler`, which could change the interface implementation as `IRequestHandler<UpdateGroupDetailsCommand, Optional<UpdateGroupDetailsCommandResult>>`, considering cases when we try to update a group that doesn't exist (we'll see the implementation in the next section).

## Imperative usage

Let's being implementing things in a more traditional way (for C# developers), imperatively.

Let's use `UpdateGroupDetailsCommandHandler` as an example.

`Domain\UseCases\UpdateGroupDetails\UpdateGroupDetailsCommandHandler.cs`
```csharp
public sealed class UpdateGroupDetailsCommandHandler
        : IRequestHandler<UpdateGroupDetailsCommand, Optional<UpdateGroupDetailsCommandResult>>
{
    // ...

    public async Task<Optional<UpdateGroupDetailsCommandResult>> Handle(
        UpdateGroupDetailsCommand request,
        CancellationToken cancellationToken)
    {
        var maybeGroup = await _userGroupQueryHandler.HandleAsync(
            new UserGroupQuery(request.UserId, request.GroupId),
            cancellationToken);

        if (!maybeGroup.TryGetValue(out var group))
        {
            return Optional.None<UpdateGroupDetailsCommandResult>();
        }

        var maybeUser = await _userByIdQueryHandler.HandleAsync(
            new UserByIdQuery(request.UserId),
            cancellationToken);

        if (!maybeUser.TryGetValue(out var currentUser))
        {
            // TODO: we'll get rid of these exceptions in the next episode
            throw new InvalidOperationException("Invalid user to create a group.");
        }

        group.Rename(currentUser, request.Name);

        await _groupsRepository.UpdateAsync(group, uint.Parse(request.RowVersion), cancellationToken);

        return Optional.Some(new UpdateGroupDetailsCommandResult(
            group.Id,
            group.Name,
            group.RowVersion.ToString()));
    }
}
```

Comparing with a version that uses `null`, the code is very much alike, having `if` statements in the same places. The difference is that we're forced by the compiler to handle these cases, as if we want to use the actual value we need to grab it. In a `null` driven approach, we could forget to handle these cases.

This way of using the `Optional` type is maybe not the one that takes the biggest advantage of it, considering functional approaches, but in my opinion, just the fact that we're forced to think about the value absence possibility makes a big difference.

## Functional usage

Now let's take a look at a more functional approach to using the `Optional` type.

When we think about a more functional approach, one easy way to think about it for C# developers is LINQ.

In LINQ, we chain together function calls, in a more declarative way. When we do, for example, `values.Where(v => v % 10 == 0).Select(v => v / 10 )`, we don't think about where to store the values that fulfill the condition, what happens to those who don't and so on. We just declare that for a given collection of values, we want to divide by 10 the ones that are divisible by 10.

We can use a similar approach in other situations, like the `Optional` type. Following on the footsteps of LINQ, we can create some extension methods to help us.

I created a bunch of extension methods, with common operations in the `OptionalExtensions.cs` file, but for this post, I'll just use a couple of them as examples.

Side note: removed the guard clauses from the methods below to keep the sample code simpler.

`Domain\Shared\OptionalExtensions.cs`
```csharp
public static class OptionalExtensions
{
    public static Optional<TOut> Map<TIn, TOut>(this Optional<TIn> maybeValue, Func<TIn, TOut> mapper)
    {
        return maybeValue.TryGetValue(out var value)
            ? Optional.Some(mapper(value))
            : Optional.None<TOut>();
    }

    public static TOut MapValueOr<TIn, TOut>(this Optional<TIn> maybeValue, Func<TIn, TOut> some, Func<TOut> none)
    {
        return maybeValue.TryGetValue(out var value)
            ? some(value)
            : none();
    }

    public static async Task MatchSomeAsync<T>(this Optional<T> maybeValue, Func<T, Task> some)
    {
        if (maybeValue.TryGetValue(out var value))
        {
            await some(value);
        }
    }

    // ...
}
```

Above we have 3 examples of helper methdods: `Map`, `MapValueOr` and `MatchSomeAsync`.

### Map

`Map` can be used in a situation where if we have a value, we want to map it to something else, but if we have no value, we want to propagate this fact. Let's see it in action in the `GetUserGroupQueryHandler`.

`Domain\UseCases\UpdateGroupDetails\GetUserGroupQueryHandler.cs`
```csharp
 public sealed class GetUserGroupQueryHandler : IRequestHandler<GetUserGroupQuery, Optional<GetUserGroupQueryResult>>
{
    // ...

    public async Task<Optional<GetUserGroupQueryResult>> Handle(
        GetUserGroupQuery request,
        CancellationToken cancellationToken)
    {
        var maybeGroup = await _userGroupQueryHandler.HandleAsync(
            new UserGroupQuery(request.UserId, request.GroupId),
            cancellationToken);

        return maybeGroup.Map(
            group => new GetUserGroupQueryResult(
                group.Id,
                group.Name,
                group.RowVersion.ToString(),
                new GetUserGroupQueryResult.User(group.Creator.Id, group.Creator.Name)));
    }
}
```

As we can see, when returning the group, instead of doing an `if` (or using a ternary operator) where the body would be mapping the value and returning it, then the else returning `Optional.None`, we can simply call `Map` on `maybeGroup`, passing it the mapping logic. If there is a value, it will be mapped and returned, otherwise, the absence of value is propagated.

It's maybe not a massive improvement, but it's a bit simpler and more declarative.

### MapValueOr

`MapValueOr` is used in situations when we want to map the value or, if it doesn't exist, return something else. It acts pretty much like an `if` `else`, but can be most useful when we're chaining method calls.

The example I have in this case isn't the greatest, as it's not in a chain of method calls, could easily use an `if` `else` (or ternary operator), but should be enough to understand its usage.

`Web\Features\Groups\GroupsController.cs`
```csharp
 [Route("groups")]
public class GroupsController : ControllerBase
{
    [HttpGet]
    [Route("{id}")]
    public async Task<ActionResult<GetUserGroupQueryResult>> GetByIdAsync(long id, CancellationToken ct)
    {
        var result = await _mediator.Send(new GetUserGroupQuery(_currentUserAccessor.Id, id));

        return result.MapValueOr<GetUserGroupQueryResult, ActionResult<GetUserGroupQueryResult>>(
            r => r,
            () => NotFound());
    }
}
```

Above, we have the implementation of the get group by id route. As we get the result, we're mapping it to the controller's action result. If the group is found, we return it, as it's implicitly converted to an `ActionResult<GetUserGroupQueryResult>`, otherwise we want to return a 404.

### MatchSomeAsync

`MatchSome` (or its async variant) can be used when we want to do something only when there's a value. We can see an example in `DeleteGroupCommandHandler`.

`Domain\UseCases\UpdateGroupDetails\DeleteGroupCommandHandler.cs`
```csharp
public class DeleteGroupCommandHandler : IRequestHandler<DeleteGroupCommand, Unit>
{
    // ...

    public async Task<Unit> Handle(DeleteGroupCommand request, CancellationToken cancellationToken)
    {
        var maybeGroup = await _userGroupQueryHandler.HandleAsync(
            new UserGroupQuery(request.UserId, request.GroupId),
            cancellationToken);

        await maybeGroup.MatchSomeAsync(async group =>
        {
            if (group.IsAdmin(request.UserId))
            {
                await _groupsRepository.DeleteAsync(group, cancellationToken);
            }
        });

        return Unit.Value;
    }
}
```

In this case, we want to delete a group, but only if it exists. We use `MatchSomeAsync`, passing in the deletion logic. If there is a group, we'll delete it, otherwise it's just a NOOP.

Again, this example is probably too simple and using an `if` statement wouldn't be more complex, but take it as an illustration of a possible
usage scenario, that could be more useful in a more complex situation.

## C# 8 nullable reference types

With C# 8, we've seen the introduction of [nullable reference types](https://docs.microsoft.com/en-us/dotnet/csharp/nullable-references) (also referred to as NRTs).

This new feature brings in to question if using types like the `Optional` we've been discussing are still worth it. My honest answer, no clue! 😅

The two main things I dislike about NRTs in C#, although I understand the reasoning behind them, are:

1. We only get warnings: instead of warnings, I'd prefer to actually have errors, forcing you to handle them (like Kotlin does). Maybe this is a moot point though, as we can configure the warnings to be treated as errors.
2. Requires changes to all the used libraries: all the dependencies must take into consideration this new reality. If a dependency doesn't update to support NRTs, the compiler is in the dark and assumes it's all good. A simple example, with EF Core, SingleOrDefaultAsync might return `null` if nothing is found, yet as it has not yet been adapted to NRTs, it appears as if it's all good, no warning is shown. In such a case, it would be safer if everything that's not yet adapted, would be assumed unsafe, instead of safe (Kotlin is again an example of such an implementation).

Again, I understand why it is the way it is, as it would be too disruptive, but it also seems as it ends up not fulfilling its potential.

It's still early days of the feature though, and we're in the "nullable rollout phase", as Mads Torgersen said [here](https://devblogs.microsoft.com/dotnet/embracing-nullable-reference-types/), so we'll have to wait and see how things pan out.

A final note on the `Optional` type, is that besides forcing us to think about the lack of value, as we've seen above it can also enable some more functional approaches to write code in those scenarios, like chaining method calls that "short-circuit" as soon as something missing. In these cases, it can still provide value even if we get to a point in which NRTs fulfill their potential.

## Outro

That does it for this episode. We've played around with the `Optional` type (aka `Option` or `Maybe`), creating a simple implementation, using it to make our logic safer against `NullReferenceException`s. We also played with more functional approaches to using this type, by enriching it with some extension methods, LINQ style, that reduce conditional statements.

Links in the post:

- [Understanding the Option (Maybe) Functional Type](http://codinghelmet.com/articles/understanding-the-option-maybe-functional-type)
- [Optional - A robust option type for C#](https://github.com/nlkl/Optional)
- [C# functional extensions NuGet library](https://enterprisecraftsmanship.com/posts/c-functional-extensions-nuget-library/)
- [C# 8 nullable reference types](https://docs.microsoft.com/en-us/dotnet/csharp/nullable-references)

The source code for this post is in the [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode037) repository, tagged as `episode037`.

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!