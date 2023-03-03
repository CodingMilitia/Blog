---
author: João Antunes
date: 2020-02-06 19:30:00+00:00
layout: post
title: "E036 - Making things more object oriented with rich domain entities - ASPF02O"
summary: "In this episode, we'll make things more object oriented, by moving some logic that's present in the request handlers, which in fact should be present in the domain entities, that currently are just bags of data with public getters and setters."
images:
- '/images/2020/02/06/aspnet-core-from-zero-to-overkill-e036.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-036-from-zero-to-overkill-making-things-more-object-oriented-with-rich-domain-entities
---

In this episode, we'll make things more object oriented, by moving some logic that's present in the request handlers, which in fact should be present in the domain entities, that currently are just bags of data with public getters and setters.

**Note:** depending on your preference, you can check out the following video or skip it to the written version below.

{{< youtube laDbyrHpVSA >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro

One thing I've been noticing for a while now, is our tendency to put all logic inside services (or like we've seen recently, request handlers), leaving the classes that represent our domain entities as simple bags of properties that we pass around. I've written about it [here](https://blog.codingmilitia.com/2019/12/10/use-pocos-to-group-data-and-logic), but now it's time to apply that to this series code.

Moving the logic to the domain entities is something we read about when looking at DDD (domain driven design), resulting in what's called a rich domain model, but I'd argue it's not specific to this context. DDD literature commonly uses object oriented programming languages, but it can be implemented in other paradigms (e.g. functional programming). When using OOP languages however, it makes sense to use this paradigm's best practices, and probably the most important ones are to use the objects to represent specific concepts, encapsulate data and expose behavior through a public API.

When we put all the logic inside services or similar concepts, the code ends up being more procedural than object oriented. There's nothing wrong with procedural, but for the typical line of business applications (and others), modeling concepts using object oriented practices works well.

## Moving domain logic into entities

So far, in the group management API, all the domain logic is part of the use case handlers. The goal for this episode is to reduce the logic contained in the handlers to the entities, making the handlers focus on orchestrating the interactions with the entities, repositories and so on.

### Entities

As the application is far from complex at this point, the focus is on the `Group` entity. I'll drop the code below and then go through it.

`Group.cs`
```csharp
public class Group : IVersionedEntity
{
    private readonly List<GroupUser> _groupUsers = new List<GroupUser>();

    public Group(string name, User creator)
    {
        Name = !string.IsNullOrWhiteSpace(name) ? name : throw new ArgumentNullException(nameof(name));
        Creator = creator ?? throw new ArgumentNullException(nameof(creator));
        _groupUsers.Add(GroupUser.NewAdministrator(Id, creator.Id));
    }

    public long Id { get; private set; }
    public string Name { get; private set; }
    public uint RowVersion { get; private set; }
    public User Creator { get; private set; }
    public IReadOnlyCollection<GroupUser> GroupUsers => _groupUsers.AsReadOnly();

    public void Rename(User editingUser, string newName)
    {
        ThrowIfNotAdmin(editingUser.Id);

        Name = !string.IsNullOrWhiteSpace(newName) ? newName : throw new ArgumentNullException(nameof(newName));
    }

    public bool IsAdmin(string userId)
        => GroupUsers.Any(gu => gu.UserId == userId && gu.Role == GroupUserRole.Admin);

    // TODO: temporary we'll get rid of all these exceptions eventually
    private void ThrowIfNotAdmin(string userId)
    {
        if (!IsAdmin(userId))
        {
            throw new UnauthorizedAccessException("User is not authorized to edit this group");
        }
    }
}
```

Let's begin with the properties, as those were the only things that were there already. All the properties are still there, but all of them now have private setters (minus `GroupUsers`, which doesn't even have that). The goal is to make all the changes to the entity data go through explicitly exposed APIs, that take into consideration the domain rules, rather than everything being modifiable through setters. On that note, the `GroupUsers` property not only doesn't have a setter, but the type was changed to be an `IReadOnlyCollection<GroupUser>`, so the collection can't be modified from outside the entity.

We now have a constructor that receives the name of the group and the user that is creating it. Not only is this necessary as we've disabled access to the properties, but it allows us to abstract the caller from the need to set the creator and add it to the list of group users, the constructor also encapsulates that piece of domain logic.

Continuing onwards, we have a couple more public methods, `Rename` and `IsAdmin`. `Rename` not only has a name that clearly states its intent (for example `SetName` would be a much more generic name), it will enforce the rules regarding renaming the entity, like requiring the editor to be an administrator, as well as the new name not being empty. With a public setter we wouldn't be able to achieve all of these characteristics. As for the `IsAdmin` method, it simply centralizes the logic to check if a user is an administrator, avoiding its repetition in the handlers that require such information.

The rest of the entities are not as interesting as the group entity, but they follow the same logic: encapsulate data, expose a public API with the available capabilities.

One final note for the `GroupUser` class (code below), that exposes a couple of helper factory methods, `NewAdministrator` and `NewParticipant`, which again serve to make the capabilities explicit. We could easily call the constructor passing in the role, but using these kinds of methods can simplify the calling code and make it more readable. Naming is important (and hard 😛).

`GroupUser.cs`
```csharp
public class GroupUser
{
    public GroupUser(long groupId, string userId, GroupUserRole role)
    {
        GroupId = groupId;
        UserId = !string.IsNullOrWhiteSpace(userId) ? userId : throw new ArgumentNullException(nameof(userId));
        Role = role;
    }

    public long GroupId { get; private set; }
    public string UserId { get; private set; }
    public GroupUserRole Role { get; private set; }

    public static GroupUser NewAdministrator(long groupId, string userId)
        => new GroupUser(groupId, userId, GroupUserRole.Admin);

    public static GroupUser NewParticipant(long groupId, string userId)
        => new GroupUser(groupId, userId, GroupUserRole.Participant);
}
```

### Use case handlers

Now that we moved a bunch of logic to the entities where it belongs, we can adjust the handlers to make use of it. We'll use `UpdateGroupDetailsCommandHandler` as an example, as the others follow the same approach.

`UpdateGroupDetailsCommandHandler.cs`
```csharp
public sealed class UpdateGroupDetailsCommandHandler
        : IRequestHandler<UpdateGroupDetailsCommand, UpdateGroupDetailsCommandResult>
{
    public UpdateGroupDetailsCommandHandler(
        IQueryHandler<UserByIdQuery, User> userByIdQueryHandler,
        IQueryHandler<UserGroupQuery, Group> userGroupQueryHandler,
        IVersionedRepository<Group, uint> groupsRepository)
    {
        // ...
    }

    public async Task<UpdateGroupDetailsCommandResult> Handle(
        UpdateGroupDetailsCommand request,
        CancellationToken cancellationToken)
    {
        var group = await _userGroupQueryHandler.HandleAsync(
            new UserGroupQuery(request.UserId, request.GroupId),
            cancellationToken);

        if (group is null)
        {
            return null;
        }

        var currentUser = await _userByIdQueryHandler.HandleAsync(
            new UserByIdQuery(request.UserId),
            cancellationToken);

        group.Rename(currentUser, request.Name);

        await _groupsRepository.UpdateAsync(
            group,
            uint.Parse(request.RowVersion),
            cancellationToken);

        return new UpdateGroupDetailsCommandResult(
            group.Id,
            group.Name,
            group.RowVersion.ToString());
    }
}
```

In general, the code is the same as it was before, with a small change. We're fetching the group and current user information, then we call `group.Rename`, without needing any additional logic, as it's present in the `Rename` method. The rest, again, the same as before, update the data in the database and then return the updated group.

## Playing nicely with EF Core

Until now, we were focusing on the domain logic part of our application. Now we need to be able to persist said entities to the database.

In a perfect world, we could completely design the entities (and all the domain logic for that matter) without concerning ourselves with what goes on in the infrastructure side of things. In reality, there are decisions to make and tradeoffs to consider. If we want a completely "pure" domain, we'll have more work in the infrastructure, as we'd probably need to replicate all the entity classes to use for persistence (usually referred to as the persistence model). Even then, we'd need to have a way to create the domain entities given the persistence model, as in a "pure" implementation, it wouldn't normally expose all of its internal data for others to set.

Fortunately, EF Core (and other ORMs) lets us get away with just a few tweaks, so we won't have a complete infrastructure agnostic domain model, but it won't be too far off.

Going back to the group entity we saw above, one thing I omitted is the presence of a private parameterless constructor. Let me drop here the relevant code for this section.

`Group.cs`
```csharp
public class Group : IVersionedEntity
{
    private readonly List<GroupUser> _groupUsers = new List<GroupUser>();

    public Group(string name, User creator)
    {
        // ...
    }

    private Group()
    {
    }

    public long Id { get; private set; }
    public string Name { get; private set; }
    public uint RowVersion { get; private set; }
    public User Creator { get; private set; }
    public IReadOnlyCollection<GroupUser> GroupUsers => _groupUsers.AsReadOnly();

    // ...
}
```

EF Core can use constructors with parameters, [as long as they're not navigation properties](https://docs.microsoft.com/en-us/ef/core/modeling/constructors) (maybe this changes in the future). In this case, the `creator` parameter is a navigation property, so we're out of luck with it. Alternatively, by having a parameterless constructor, EF can create an instance of the class (even if it is private, reflection FTW 😛).

With an instance of the object in hand, EF sets the various properties. That's the reason (almost) all properties have a private setter, even things that don't change, like the id, that could get away with having a getter only. By having a private setter, much like the private constructor, EF can use it to hydrate the entity with the data from the database.

As for the `GroupUsers` property, EF can't set it directly, but it can figure out it has a backing field and work with it, being automatic as long as we [follow some conventions](https://docs.microsoft.com/en-us/ef/core/modeling/backing-field), in this case, the backing field name is the camel case version of the property, prefixed with an underscore. If we didn't follow such conventions, we could configure it in the `IEntityTypeConfiguration` implementations we have in the infrastructure project, but as we are, no need for that.

As the entities we have so far aren't very complex, that's all there is to it, but EF Core has more features like these to ease the work with rich domain models, so it's a matter of investigating the possibilities as needs arise.

## Outro

That's about it for this episode, as we made a code a bit more object oriented, by moving the logic to the place it makes sense to be, instead of shoving everything in services (or request handlers).

Although the focus was on better separation of concerns, through object oriented practices, with it some common DDD related topics emerged. If you're looking for specific DDD resources, [Julie Lerman](http://thedatafarm.com/blog/), [Steve Smith](https://ardalis.com/blog) and [Vladimir Khorikov](https://enterprisecraftsmanship.com/posts) blogs/courses are a good bet.

Links in the post:

- [EF Core - Entity types with constructors](https://docs.microsoft.com/en-us/ef/core/modeling/constructors)
- [EF Core - Backing Fields](https://docs.microsoft.com/en-us/ef/core/modeling/backing-field)
- [Julie Lerman's blog](http://thedatafarm.com/blog/)
- [Steve Smith's blog](https://ardalis.com/blog)
- [Vladimir Khorikov's blog](https://enterprisecraftsmanship.com/posts)

The source code for this post is in the [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode036) repository, tagged as `episode036`.

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!