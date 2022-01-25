---
author: João Antunes
date: 2020-03-02 18:30:00+00:00
layout: post
title: "More explicit domain error handling and fewer exceptions with Either and Error types [ASPF02O|E038]"
summary: "Following up on the last episode about the Optional type, we continue taking inspiration from functional languages and introduce Either and Error types, as a way to make the possible business logic outcomes more explicit and minimize using exceptions in non-exceptional situations."
images:
- '/images/2020/03/02/e038.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-038-from-zero-to-overkill-more-explicit-error-handling-with-either
---

Following up on the last episode about the Optional type, we continue taking inspiration from functional languages and introduce Either and Error types, as a way to make the possible business logic outcomes more explicit and minimize using exceptions in non-exceptional situations.

**Note:** depending on your preference, you can check out the following video, otherwise, skip to the written version below.

{{< yt 3lAjAQ4J13Q >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

On the footsteps of the last episode, where, taking inspiration from functional languages, we introduced an `Optional` type to represent the absence of value, in this episode we'll introduce some new types, `Either` and `Error` (this one accompanied by a hierarchy of specific error types), to handle common business logic outcomes more explicitly. This will allow us to minimize the use of exceptions, as they should be used for exceptional cases, not for every error we come across, particularly business errors.

Alongside these new types, we'll introduce some extension methods to go along with them, to simplify their usage, otherwise, we would just end up in a world of `if` and `else` pain. If you used things like LINQ or RxJS (very common in Angular applications) in the past, the usage of these extension methods will look pretty familiar.

## Creating a robust Error type

For all of this to make sense, we take a look at all the concepts and then bring it all together, but we need to start somewhere, so let's begin with the `Error` type.

Before getting into how to represent errors, let's think about what we want to represent. Considering the business logic we've implemented so far, the kinds of errors we encountered are:

- Not found - we're trying to get a resource that doesn't exist/we don't have access
- Invalid - we're trying to do something that's not valid (e.g. change the name of the group to empty `string`)
- Unauthorized - we're trying to do something we don't have permissions to (e.g. renaming the group without being an administrator)

At first glance, our typical C# developer intuition would probably scream "create an `enum`!". So we could do something like:

```csharp
public enum Error { NotFound, Invalid, Unauthorized };
```

This would work, sure, but two things aren't great and could be improved:

1. We can't provide more information about the error
2. We need to always remember to find and adjust all the places we're handling errors when we add a new one

For the first one, we could kind of fix it by creating an `Error` class, containing, for example, a `string` message and an `ErrorType` `enum` value, like so:

```csharp
public enum ErrorType { NotFound, Invalid, Unauthorized };

public class Error
{
    public Error(ErrorType type, string message)
    {
        Type = type;
        Message = message;
    }

    public ErrorType Type { get; }
    public string Message { get; }
}
```

This partially solves the first mentioned problem but it's still not great:

1. The information we add is always general to all kinds of errors, as it's always present
2. Worse then 1., we add information to the `Error` class that only makes sense in some cases, so the caller needs to know what can be used depending on the `ErrorType`
3. The name `ErrorType` seems to want to tell us something... `Type`, as in, we can represent it as a type 🙂

Ok, let's give it another go and represent the different kinds of errors as types by themselves. It could be something like the following:

```csharp
public abstract class Error
{
}

public class NotFound : Error
{
    public NotFound(string message)
    {
        Message = message;
    }

    public string Message { get; }
}

public class Invalid : Error
{
    public Invalid(string message)
    {
        Message = message;
    }

    public string Message { get; }
}

public class Unauthorized : Error
{
    public Unauthorized(string message)
    {
        Message = message;
    }

    public string Message { get; }
}
```

This implementation should address the first issue we raised. We have a base error, just so we can assign all types of errors to a variable of said type, then have a `class` per error kind, which can hold the information that makes sense for each specific situation. The example isn't great, as all types of error have a single message, but imagine that, for example, the `Invalid` error instead of having a single message, had a collection of messages, one for each invalid condition detected (in fact, this is something we'll probably need to do in the future, but for now, let's keep it as is).

Now, how about the other issue mentioned previously: we need to always remember to find and adjust all the places we're handling errors when we add a new one.

This implementation does not address this problem. if we created a new error type, for instance `Unexpected`, inheriting from `Error` as the others, we would again need to remember to find and adjust every place we're handling errors. Having to always remember to do something, when we're probably thinking about other subjects isn't really a great strategy and can be error-prone.

Fortunately, even if with a bit of work, we can make the compiler help us with that.

### Error type as a tagged union

The concept of [tagged union](https://en.wikipedia.org/wiki/Tagged_union) can help us implement an `Error` type without the issues mentioned above. This concept is more prevalent in functional programming languages, but with a bit of work, we can also achieve it in C#.

From [Wikipedia](https://en.wikipedia.org/wiki/Tagged_union): "In computer science, a tagged union, also called a variant, variant record, choice type, discriminated union, disjoint union, sum type or coproduct, is a data structure used to hold a value that could take on several different, but fixed, types."

An example of a tagged union in C# is an `enum`. The problem with `enum`s for this specific case, like we saw above, is that they can't carry additional information and C# doesn't enforce the handling of all possibilities, so if we add a new entry, we'd need to remember to adjust all usages.

With some "trickery", we can achieve a similar behavior by implementing a closed type hierarchy and the [visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern), as blogged by Mark Seemann [here](https://blog.ploeh.dk/2018/06/25/visitor-as-a-sum-type/).

Let's look at the code:

`Domain\Shared\Error.cs`
```csharp
public abstract class Error
{
    private Error() { }

    public abstract TResult Accept<TVisitor, TResult>(TVisitor visitor)
        where TVisitor : IErrorVisitor<TResult>;

    public interface IErrorVisitor<out TVisitResult>
    {
        TVisitResult Visit(NotFound result);

        TVisitResult Visit(Invalid result);

        TVisitResult Visit(Unauthorized result);
    }

    public sealed class NotFound : Error
    {
        public NotFound(string message)
        {
            Message = message;
        }

        public string Message { get; }

        public override TResult Accept<TVisitor, TResult>(TVisitor visitor)
            => visitor.Visit(this);
    }

    public sealed class Invalid : Error
    {
        public Invalid(string message)
        {
            Message = message;
        }

        public string Message { get; }

        public override TResult Accept<TVisitor, TResult>(TVisitor visitor)
            => visitor.Visit(this);
    }

    public sealed class Unauthorized : Error
    {
        public Unauthorized(string message)
        {
            Message = message;
        }

        public string Message { get; }

        public override TResult Accept<TVisitor, TResult>(TVisitor visitor)
            => visitor.Visit(this);
    }
}
```

Looking at the code, we can see a lot of similarities with the first `Error` type hierarchy shown in the post, but also a bunch of differences. Let's go through these differences.

For starters, all the derived types moved into the `Error` class itself, became sealed and the base constructor is now private. The goal with this is to make the hierarchy closed, meaning we cannot create new error types outside of it. If we could inherit from `Error` (or one of its private subtypes) without such control, we wouldn't be able to ensure the handling of all error possibilities.

The second thing we have is the implementation of the visitor pattern, with the introduction of the `IErrorVisitor` interface and the `Accept` method, which we'll use to map each kind of error to something else.

The `IErrorVisitor` should be implemented by any client that needs to handle all the types of errors. The `Accept` method is the way to invoke the visitor code taking into consideration the actual type of error.

Side note: the `Accept` method receiving a parameter of a generic type that implements the `IErrorVisitor` interface, instead of using the interface directly, is a performance optimization, to avoid boxing visitor implementations that are `struct`s.

To understand the value of all of this, let's introduce an example, as it'll make it easier.

From the names of the error types introduced so far, we can picture they can map rather easily to HTTP status codes, which is at the end of the day one of the things we want to do with them, informing the group management API client about what went wrong. With that in mind, we can create an `IErrorVisitor` implementation that will map each type of error to an ASP.NET Core MVC `ActionResult`.

`Web\Features\ErrorMappingVisitor.cs`
```csharp
public readonly struct ErrorMappingVisitor<TModel> : Error.IErrorVisitor<ActionResult<TModel>>
{
    public ActionResult<TModel> Visit(Error.NotFound result)
        => new NotFoundObjectResult(result.Message);


    public ActionResult<TModel> Visit(Error.Invalid result)
        => new BadRequestObjectResult(result.Message);


    public ActionResult<TModel> Visit(Error.Unauthorized result)
        => new StatusCodeResult(StatusCodes.Status403Forbidden);
}
```

Nothing overly complex. We map `NotFound` to a 404, `Invalid` to a 400 and `Unauthorized` to 403.

Now the advantage of having implemented all of this the way that we did, is that the compiler will help us keep things tidy as we make changes.

Imagine we want to add a new type of error, for instance `Unexpected`. If we just copy-paste one of the other ones and correct the name, we're greeted with an error in the `Accept` implementation.

{{< embedded-image "/images/2020/03/02/new-error-type-missing-overload.jpg" "missing overload" >}}

This means we're missing a `Visit` method overload in the `IErrorVisitor` interface to cover for the newly added error type, so we go there and add it.

As we do this, now we get an error in our `ErrorMappingVisitor` struct, because it's not implementing all of the methods of the `IErrorVisitor` interface.

{{< embedded-image "/images/2020/03/02/new-error-type-incomplete-interface-implementation.jpg" "incomplete interface implementation" >}}

So now we need to also implement the remainder of the interface, for the project to even compile, ensuring that we always handle all types of errors.

It's not super hard, but it is some amount of work to get done in C#. Other languages, particularly functional languages, have this kind of thing solved by the language itself, removing a lot of boilerplate. For this reason, it's probably not something we do all the time, but in some cases, it's good to have this extra protection of ensuring correctness at compile time.

## The Either type

With an `Error` type in hand, now we need to figure out how to return it in our code. We could use them as the detail of an exception and throw it, but that wouldn't make sense given the title of this post 🙂.

Another possibility is to always return a pair of objects from our business logic methods, one being the error and the other the actual expected object when things go well. Using an `Either` type is one possible way to achieve this.

`Either` is a type commonly seen in functional languages (e.g. [Scala](https://www.scala-lang.org/api/current/scala/util/Either.html)), that can be used to return a value of two possible types. It's similar to the `Optional` type we discussed in the previous episode, but instead of being "some" or "none", it's "left" or "right". We could, for instance, implement the `Optional` type on top of `Either`, for instance using "left" for "none" and "right" for "some". When using `Either` to represent either an error or a success (as we'll do), the convention is for the "left" to contain the error and "right" the successful result.

Following the same approach as we took for the `Optional` type, we'll implement something ourselves in this episode, but there are already libraries out there that we can use.

Let's start by the `Either` type itself.

`Domain\Shared\Either.cs`
```csharp
public abstract class Either<TLeft, TRight>
{
    private Either()
    {
    }

    public sealed class Left : Either<TLeft, TRight>
    {
        public Left(TLeft value)
        {
            Value = value;
        }

        public TLeft Value { get; }
    }

    public sealed class Right : Either<TLeft, TRight>
    {
        public Right(TRight value)
        {
            Value = value;
        }

        public TRight Value { get; }
    }
}
```

There are more ways to implement it, but in this case, we're going with a similar approach to the one taken to implement the `Error` type, with a tagged union where `Either` can be `Left` or `Right`. Each "side" stores an instance of the type associated with it.

To simplify the creation of `Either` instances, namely avoid having to type in the generic type arguments every time, we can create a couple of helper methods.

`Domain\Shared\Either.cs`
```csharp
public static class Either
{
    public static Either<TLeft, TRight> Left<TLeft, TRight>(TLeft value)
        => new Either<TLeft, TRight>.Left(value);

    public static Either<TLeft, TRight> Right<TLeft, TRight>(TRight value)
        => new Either<TLeft, TRight>.Right(value);
}
```

A bit unrelated to the `Either` type itself, but to further simplify usage, specifically in the way we're expecting to use it in this application, we can create some extra helper methods to create results in our business logic.

`Domain\Shared\Result.cs`
```csharp
public static class Result
{
    public static Either<Error, TValue> Success<TValue>(TValue value)
        => Either.Right<Error, TValue>(value);

    public static Either<Error, TValue> Invalid<TValue>(string message)
        => Either.Left<Error, TValue>(new Error.Invalid(message));

    public static Either<Error, TValue> NotFound<TValue>(string message)
        => Either.Left<Error, TValue>(new Error.NotFound(message));

    public static Either<Error, TValue> Unauthorized<TValue>(string message)
        => Either.Left<Error, TValue>(new Error.Unauthorized(message));
}
```

When it's a success, we create an `Either.Right` with the value, when it's an error, we create the specified error and pass it into an `Either.Left`.

Now to see an example of it in action, we can make use of the `Group` entity from previous episodes, changing the `Rename` method's logic to remove the exception throwing.

`Domain\Entities\Group.cs`
```csharp
public class Group : IVersionedEntity<uint>
{
    // ...

    public string Name { get; private set; }

    public Either<Error, Unit> Rename(User editingUser, string newName)
    {
        if (!IsAdmin(editingUser.Id))
        {
            return Result.Unauthorized<Unit>("User is not authorized to edit this group");
        }

        if (string.IsNullOrWhiteSpace(newName))
        {
            return Result.Invalid<Unit>("The group's name cannot be empty.");
        }

        Name = newName;

        return Result.Success(Unit.Value);
    }

    // ...
}
```

`Rename` now returns `Either<Error, Unit>`, as it was void before. If it wasn't void, instead of `Unit` we'd put the actual type as the "right" argument.

Where we were throwing exceptions, we're now returning the `Either` with the error details, otherwise, if all goes well, we return a successful result, more specifically an `Either.Right`.

## Building on the Either type

Now how do we handle this result in the command handler?

As it is, we could handle the returned `Either` by pattern matching it, for example:

```csharp
result switch
{
    Either<Error, Unit>.Left error => /* do something with the error */
    Either<Error, Unit>.Right success => /* do something with the success */
};
```

This would work, but with more logic to do depending on the result, it could get a bit cumbersome. Another approach, that if you saw/read the last episode on `Optional` might be expecting, is to create extension methods to "LINQ" our way out of the mess 🙂.

### Fold

The first extension method we'll introduce is `Fold` (and an async version of it). The idea with this method is that regardless of the result of the operation, we want to map to a specific type.

Side note: removed the guard clauses from the methods below to keep the sample code simpler.

`Domain\Shared\EitherExtensions.cs`
```csharp
public static TOut Fold<TLeftIn, TRightIn, TOut>(
    this Either<TLeftIn, TRightIn> result,
    Func<TLeftIn, TOut> left,
    Func<TRightIn, TOut> right)
{
    return result switch
    {
        Either<TLeftIn, TRightIn>.Left error => left(error.Value),
        Either<TLeftIn, TRightIn>.Right success => right(success.Value),
        _ => throw CreateUnexpectedResultTypeException(nameof(result))
    };
}

public static async Task<TOut> FoldAsync<TLeftIn, TRightIn, TOut>(
    this Either<TLeftIn, TRightIn> result,
    Func<TLeftIn, Task<TOut>> left,
    Func<TRightIn, Task<TOut>> right)
{
    return result switch
    {
        Either<TLeftIn, TRightIn>.Left error => await left(error.Value),
        Either<TLeftIn, TRightIn>.Right success => await right(success.Value),
        _ => throw CreateUnexpectedResultTypeException(nameof(result))
    };
}
```

As we can see, that pattern matching mentioned previously lives here. We had to do it somewhere, so we abstracted it away. We get as input a couple of `Func`s, being only one invoked, depending on the result being `Left` or `Right`.

We'll see an example of `Fold` in action in a bit.

### Map

Another extension method we can create is `Map`. `Map` acts like its homonymous we saw in the past episode, in this case mapping `Right` if its the case, otherwise letting `Left` flow.

`Domain\Shared\EitherExtensions.cs`
```csharp
public static Either<TLeft, TRightOut> Map<TLeft, TRightIn, TRightOut>(
    this Either<TLeft, TRightIn> result,
    Func<TRightIn, TRightOut> right)
{
    return result switch
    {
        Either<TLeft, TRightIn>.Left error => Either.Left<TLeft, TRightOut>(error.Value),
        Either<TLeft, TRightIn>.Right success => Either.Right<TLeft, TRightOut>(right(success.Value)),
        _ => throw CreateUnexpectedResultTypeException(nameof(result))
    };
}

public static async Task<Either<TLeft, TRightOut>> MapAsync<TLeft, TRightIn, TRightOut>(
    this Either<TLeft, TRightIn> result,
    Func<TRightIn, Task<TRightOut>> right)
{
    return result switch
    {
        Either<TLeft, TRightIn>.Left error => Either.Left<TLeft, TRightOut>(error.Value),
        Either<TLeft, TRightIn>.Right success => Either.Right<TLeft, TRightOut>(await right(success.Value)),
        _ => throw CreateUnexpectedResultTypeException(nameof(result))
    };
}
```

The code is similar to `Fold`, differing in that it only needs one `Func`, to map the right side, and returns an `Either`, so it needs to create it to wrap around the result value (or error).

## Bringing it all together

Having all the components in hand, let's bring it all together.

As we changed the `Group` class `Rename` method, let's take as an example the `UpdateGroupDetailsCommandHandler` that uses it.

`Domain\UseCases\UpdateGroupDetails\UpdateGroupDetailsCommandHandler.cs`
```csharp
public sealed class UpdateGroupDetailsCommandHandler
    : IRequestHandler<UpdateGroupDetailsCommand, Either<Error, UpdateGroupDetailsCommandResult>>
{
    // ...

    public async Task<Either<Error, UpdateGroupDetailsCommandResult>> Handle(
        UpdateGroupDetailsCommand request,
        CancellationToken cancellationToken)
    {
        var maybeGroup = await _userGroupQueryHandler.HandleAsync(
            new UserGroupQuery(request.UserId, request.GroupId),
            cancellationToken);

        if (!maybeGroup.TryGetValue(out var group))
        {
            return Result.NotFound<UpdateGroupDetailsCommandResult>(
                $"Group with id {request.GroupId} not found.");
        }


        var maybeUser = await _userByIdQueryHandler.HandleAsync(
            new UserByIdQuery(request.UserId),
            cancellationToken);

        if (!maybeUser.TryGetValue(out var currentUser))
        {
            return Result.Invalid<UpdateGroupDetailsCommandResult>(
                "Invalid user to create a group.");
        }

        return await group
            .Rename(currentUser, request.Name)
            .MapAsync(async _ =>
            {
                await _groupsRepository.UpdateAsync(
                    group,
                    uint.Parse(request.RowVersion),
                    cancellationToken);

                return new UpdateGroupDetailsCommandResult(
                    group.Id,
                    group.Name,
                    group.RowVersion.ToString());
            });
    }
}
```

For the minor changes, we have:

- The command output was changed from just `UpdateGroupDetailsCommandResult` to `Either<Error, UpdateGroupDetailsCommandResult>`.
- When the group is not found, we now return an error with that information.
- Added an extra check that was missing, to ensure the user exists before trying to perform the renaming operation.

Finally, we make use of the `Map` extension method (in its async variant) to follow up on the result of `Rename`. If it failed, that error will flow, but if it was successful, the group will be updated through the repository and the successful result will be returned.

A final thing we need to change to tie it all together is to handle `Either` in the controller as well, to map the command/query result to an ASP.NET Core MVC `ActionResult`.

To keep the controller as clean as possible, and also be able to reuse the logic in other controllers, we can create some helper methods. Part of what we'll need we saw already, in the form of the visitor implementation (`ErrorMappingVisitor`) to map the errors into `ActionResult`s. What we're missing is also handling the success case.

To do this, we have a static class named `ResultExtensions`, where we put some `Either` extension methods, specifically tailored for mapping to `ActionResult`.

`Web\Features\ResultExtensions.cs`
```csharp
public static class ResultExtensions
{
    public static ActionResult<TValue> ToActionResult<TValue>(this Either<Error, TValue> result)
        => result.Fold(
            left: error => ToErrorResult<TValue>(error),
            right: success => ToSuccessResult(success, value => value));

    public static ActionResult<TModel> ToActionResult<TValue, TModel>(
        this Either<Error, TValue> result,
        Func<TValue, TModel> valueMapper)
        => result.Fold(
            left: error => ToErrorResult<TModel>(error),
            right: success => ToSuccessResult(success, valueMapper)
        );

    public static ActionResult ToUntypedActionResult<TValue>(
        this Either<Error, TValue> result,
        Func<TValue, ActionResult> successMapper)
        => result.Fold(
            left: error => ToErrorResult(error),
            right: successMapper);

    private static ActionResult<TModel> ToSuccessResult<TValue, TModel>(
        TValue result,
        Func<TValue, TModel> valueMapper)
        => result is Unit
            ? (ActionResult<TModel>) new NoContentResult()
            : valueMapper(result);

    private static ActionResult<TModel> ToErrorResult<TModel>(Error error)
        => error.Accept<ErrorMappingVisitor<TModel>, ActionResult<TModel>>(new ErrorMappingVisitor<TModel>());

    private static ActionResult ToErrorResult(Error error)
        => error.Accept<ErrorMappingVisitor<object>, ActionResult<object>>(new ErrorMappingVisitor<object>()).Result;
}
```

In this case, we're making use of `Fold` in all methods, as regardless of being a successful or error result, we want to get an `ActionResult`.

For the `Left` side, we rely on the `ErrorMappingVisitor` we saw earlier.

For the `Right` side, we have some variations:

- For the first overload of `ToActionResult`, when no extra parameter is provided, no value mapping is needed, just differs in that a `Unit` typed value will result in a 204, any other type results in a 200.
- The second overload of `ToActionResult` acts exactly in the same way as the first, but allows the value to be mapped.
- `ToUntypedActionResult` differs from the others in that it provides the caller with the responsibility of mapping the success case (i.e. maybe we want to return something other than a 200 or 204).

For a couple of examples of these extensions in use, let's take a look at the `GroupsController` `UpdateAsync` and `AddAsync` actions.

`Web\Features\Groups\GroupsController.cs`
```csharp
// ...

[HttpPut]
[Route("{id}")]
public async Task<ActionResult<UpdateGroupDetailsCommandResult>> UpdateAsync(
    long id,
    UpdateGroupDetailsCommandModel model,
    CancellationToken ct)
    => (await _mediator.Send(
            new UpdateGroupDetailsCommand(
                _currentUserAccessor.Id,
                id,
                model.Name,
                model.RowVersion),
            ct))
        .ToActionResult();

[HttpPut]
[HttpPost]
[Route("")]
public async Task<ActionResult<CreateGroupCommandResult>> AddAsync(
    CreateGroupCommandModel model,
    CancellationToken ct)
    => (await _mediator.Send(
            new CreateGroupCommand(
                _currentUserAccessor.Id,
                model.Name),
            ct))
        .ToUntypedActionResult(
            success =>
                CreatedAtAction(
                    GetByIdActionName,
                    new {id = success.Id},
                    success));

// ...
```

`UpdateAsync` uses one of the `ToActionResult` overloads, returning a 200 in the cases all goes well.

`AddAsync` uses `ToUntypedActionResult`, as in this case we want to return a 201, not one of the default status codes provided by the `ToActionResult` overloads.

In any case, we didn't worry about the error part, as it's abstracted by these helpers.

## Outro

Wrapping up, I hope I was able to make clear some of the advantages of these techniques:

- Making the domain errors explicit, avoiding surprise exceptions we didn't know could be thrown.
- Using exceptions for their true purpose, exceptional cases.
- Using the closed type hierarchies and the visitor pattern to have the compiler help us out when we make changes and don't consider all the impacts.
- Build extensions to help writing simpler code.

Even with these nice advantages, it's not all perfect, and we saw that C# doesn't exactly make it easy to employ some of these techniques, so we end up not using them as much as we probably should.

Links in the post:

- [Tagged union](https://en.wikipedia.org/wiki/Tagged_union)
- [Either type in Scala](https://www.scala-lang.org/api/current/scala/util/Either.html)
- [Visitor as a sum type](https://blog.ploeh.dk/2018/06/25/visitor-as-a-sum-type/)
- [Visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern)

The source code for this post is in the [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode038) repository, tagged as `episode038`.

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!