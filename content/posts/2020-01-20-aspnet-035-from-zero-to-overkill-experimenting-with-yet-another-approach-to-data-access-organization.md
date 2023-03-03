---
author: João Antunes
date: 2020-01-20 19:00:00+00:00
layout: post
title: "E035 - Experimenting with (yet) another approach to data access organization - ASPF02O"
summary: "In this episode, we'll take a look at (yet) another approach to organizing data access code, very likely overkill."
images:
- '/images/2020/01/20/aspnet-core-from-zero-to-overkill-e035.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-035-from-zero-to-overkill-experimenting-with-yet-another-approach-to-data-access-organization
---

In this episode, we'll take a look at (yet) another approach to organizing data access code, very likely overkill.

**Note:** depending on your preference, you can check out the following video or skip it to the written version below.

{{< youtube xSmZpZiSBgc >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

As in the past couple of episodes we've been playing around with the redesign of our group management API internal architecture, in this one we'll take a look at an alternative approach to data access code organization.

This alternative is very inspired by the same idea of the individual request handlers per use case, having an individual query handler for each type of query we want to make.

## The idea

To put it simply, the idea is based on a "classic" repository approach, which would expose all the methods required to interact with the database, decomposing it, having a dedicated query handler for each type of query.

This decomposition, at least right now, is only on the read side. On the write side (create, update, delete) we'll still be using the typical repository implementation.

In the previous episode, while implementing the create group use case, we used this approach to get information about the user (and to write the newly created group):

`CreateGroupCommandHandler.cs`
```csharp
public sealed class CreateGroupCommandHandler : IRequestHandler<CreateGroupCommand, CreateGroupCommandResult>
{
    public CreateGroupCommandHandler(
        IRepository<Group> groupsRepository, // the repository is used for the write operations
        IQueryHandler<UserByIdQuery, User> userByIdQueryHandler) // the handler for the query we'll need to make
    {
        // ...
    }

    public async Task<CreateGroupCommandResult> Handle(
        CreateGroupCommand request,
        CancellationToken cancellationToken)
    {
        var currentUser = await _userByIdQueryHandler.HandleAsync(
            new UserByIdQuery(request.UserId), // passing in the query parameter
            cancellationToken);

        var group = //...

        //using the repository for the add operation
        var addedGroup = await _groupsRepository.AddAsync(group, cancellationToken);

        // ...
    }
}
```

## Implementation

Now let's take a look at the implementation. If you already implemented the repository pattern in the past, it should all feel familiar, minus splitting up all the queries.

Let's start with the repository/write side, which is the most traditional part of this implementation. In the `Domain` project, where we put the domain logic plus the interfaces for it to interact with the infrastructure, we create an `IRepository` interface.

### Writing to the database

`Domain\Data\IRepository.cs`
```csharp
public interface IRepository<T>
{
    Task<T> AddAsync(T entity, CancellationToken ct);
    Task UpdateAsync(T entity, CancellationToken ct);
    Task DeleteAsync(T entity, CancellationToken ct);
}
```

Pretty classic stuff, a method for each letter of CRUD, except read 🙂.

Now the implementation, goes into the `Infrastructure` project, along with the `DbContext`.

`Infrastructure\Data\EfRepository.cs`
```csharp
public class EfRepository<T> : IRepository<T> where T : class
{
    private readonly GroupManagementDbContext _dbContext;

    public EfRepository(GroupManagementDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<T> AddAsync(T entity, CancellationToken ct)
    {
        await _dbContext.Set<T>().AddAsync(entity, ct);
        await _dbContext.SaveChangesAsync(ct);
        return entity;
    }

    public async Task UpdateAsync(T entity, CancellationToken ct)
    {
        var entry = _dbContext.Entry(entity);
        if (entity is IVersionedEntity versionedEntity)
        {
            entry.OriginalValues[nameof(IVersionedEntity.RowVersion)] = versionedEntity.RowVersion;
        }
        entry.State = EntityState.Modified;
        await _dbContext.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(T entity, CancellationToken ct)
    {
        _dbContext.Set<T>().Remove(entity);
        await _dbContext.SaveChangesAsync(ct);
    }
}
```

Again, with some knowledge of EF Core (for instance by looking at [episode 011](https://blog.codingmilitia.com/2019/01/16/aspnet-011-from-zero-to-overkill-data-access-with-entity-framework-core)), most of this should be familiar.

The only slightly different thing going on here, is the code to manipulate the entity version, as EF uses the version present in the original values, if we're flowing it to frontend, we must ensure that value is up to date. I'm still not completely happy with this approach (with the interface), so this may change in the future.

### Querying the database

Now for the read side. Like I mentioned, this is inspired by the individual request handlers for the use cases, so we could again make use of MediatR, but in this instance I prefer not to mix everything, so we'll keep MediatR only for the use cases.

Let's begin with the things on the `Domain` side. We'll have a couple of interfaces, `IQuery` and `IQueryHandler`, plus one class for each type of query we want to make.

`Domain\Data`
```csharp
// IQuery.cs
public interface IQuery<out TResult>
{
}

// IQueryHandler.cs
public interface IQueryHandler<in TQuery, TQueryResult> where TQuery : IQuery<TQueryResult>
{
    Task<TQueryResult> HandleAsync(TQuery query, CancellationToken ct);
}
```

`IQuery` is basically a marker interface, that we'll use on the classes that represent each type of query.

`IQueryHandler` will be implemented in the infrastructure layer, using whatever data access technology we desire.

Using the create command example we saw previously, still on the `Domain` project, we have a class representing the get user by id query.

`Domain\Data\UserByIdQuery.cs`
```csharp
public class UserByIdQuery : IQuery<User>
{
    public UserByIdQuery(string userId)
    {
        UserId = userId;
    }

    public string UserId { get; }
}
```

This class is just a DTO, with the required information to actually perform the query.

For the implementation, in the `Infrastructure` project, we have a class with a pretty straightforward call to EF's `DbSet.FindAsync`.

`Infrastructure\Data\Queries\UserByIdQuery.cs`
```csharp
public class UserByIdQueryHandler : IQueryHandler<UserByIdQuery, User>
{
    private readonly GroupManagementDbContext _db;

    public UserByIdQueryHandler(GroupManagementDbContext db)
    {
        _db = db;
    }

    public async Task<User> HandleAsync(UserByIdQuery query, CancellationToken ct)
        => await _db.Set<User>().FindAsync(new object[] {query.UserId}, ct);
}
```

For a slightly more complex example, we can look at getting a group for a given user.

`Domain\Data\UserByIdQuery.cs`
```csharp
public class UserGroupQuery : IQuery<Group>
{
    public UserGroupQuery(string userId, long groupId)
    {
        UserId = userId;
        GroupId = groupId;
    }

    public string UserId { get; }

    public long GroupId { get; }
}
```

`Infrastructure\Data\Queries\UserGroupQueryHandler.cs`
```csharp
public class UserGroupQueryHandler : IQueryHandler<UserGroupQuery, Group>
{
    private readonly GroupManagementDbContext _db;

    public UserGroupQueryHandler(GroupManagementDbContext db)
    {
        _db = db;
    }

    public async Task<Group> HandleAsync(UserGroupQuery query, CancellationToken ct)
        => await _db
            .Groups
            .Include(g => g.Creator)
            .Include(g => g.GroupUsers)
            .ThenInclude(g => g.User)
            .SingleOrDefaultAsync(g => g.Id == query.GroupId && g.GroupUsers.Any(gu => gu.User.Id == query.UserId), ct);
}
```

The idea is the same, a DTO with the query parameters and a query handler. In this case, the query is slightly more complex, including navigation properties of eager loading, as well as a some more conditions in the where clause (part of the `SingleOrDefaultAsync` call).

## Pros and cons

![but why](/assets/2020/01/20/but-why.webp)

I can imagine some looking at this and thinking, why? Overengineering! Again!

Well, yeah, I wrote that at the beginning 😛. It's very likely overkill, but I see some advantages (as well as disadvantages).

For the main pros, I'd say:

- Similarly to the use case segregation, no big repository class with lots of methods.
- Clearer dependencies - in the use case handler, we get dependencies that clearly represent specific queries, instead of repositories with a bunch of methods. This is interesting, for instance, for unit testing the use case handler, as instead of mocking everything or having to figure out which method(s) will be called, we know exactly what can be called from the dependency injected in the constructor.

As the main con, I think it's **very verbose**. The discussion around the need to implement an abstraction on top of Entity Framework is really common, as some consider it to be unneeded. With the approach used in this post, the verbosity is even greater, having to create two classes per query, one of which requires an interface implementation and dependencies injected.

## Saner approaches

The group management API will go forward with this strategy, for the other services, we'll see what comes to mind when it's time to implement them 🙂.

Before wrapping up, just wanted to leave here some approaches that are saner than this one.

The simplest one is to just use the ORM directly. Even though I like me some abstractions, it's true that ORMs like EF, EF Core, NHibernate already implement the repository and unit of work patterns, so it's acceptable to use them directly. For an example implementation, take a look at Jimmy Bogard's [ContosoUniversity project](https://github.com/jbogard/ContosoUniversityDotNetCore/tree/master/ContosoUniversity/Features).

If you like abstractions and want to implement something like repository pattern, you could go with it old-school, or you could try to include the [specification pattern](https://en.wikipedia.org/wiki/Specification_pattern) for the query bits. You can see examples of this by [Vladimir Khorikov](https://enterprisecraftsmanship.com/posts/specification-pattern-c-implementation/) and [Steve Smith](https://deviq.com/specification-pattern/) (Steve's is included in the [Microsoft eShopOnWeb reference application](https://github.com/dotnet-architecture/eShopOnWeb)).

Before going with the strategy presented in this post, I was thinking about following the example provided by Steve Smith in the eShopOnWeb application, but ended up avoiding it as I wanted to not be tied to using an ORM (as the way the specifications are implemented in the example end up being).

## Outro

That does it for this episode. We've seen an alternative approach to organizing the data access code, which is probably overkill, particularly when using an ORM.

The main reason for this approach is to explore alternatives, particularly trying to avoid as much as possible being tied to an ORM, as I'm imagining for future services to not use EF Core, using something like [Dapper](https://github.com/StackExchange/Dapper) instead, as well as not even using a SQL database.

I'm still not completely happy with this strategy though, so I'll continue thinking about tweaks I can do to improve upon it, particularly to reduce the verbosity as much as possible.

Links in the post:

- [ContosoUniversity on ASP.NET Core with .NET Core](https://github.com/jbogard/ContosoUniversityDotNetCore/)
- [Specification pattern](https://en.wikipedia.org/wiki/Specification_pattern)
- [Specification pattern: C# implementation](https://enterprisecraftsmanship.com/posts/specification-pattern-c-implementation/)
- [Specification Pattern](https://deviq.com/specification-pattern/)
- [Microsoft eShopOnWeb ASP.NET Core Reference Application](https://github.com/dotnet-architecture/eShopOnWeb)
- [Dapper](https://github.com/StackExchange/Dapper)
- [Episode 011 - Data access with Entity Framework Core - ASP.NET Core: From 0 to overkill](https://blog.codingmilitia.com/2019/01/16/aspnet-011-from-zero-to-overkill-data-access-with-entity-framework-core)

The source code for this post is in the [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode033) repository, tagged as `episode033` (didn't really change anything for this post).

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!