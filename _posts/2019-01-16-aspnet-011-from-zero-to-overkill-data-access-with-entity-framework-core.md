---
author: João Antunes
date: 2019-01-16 20:45:00+00:00
layout: post
title: "Episode 011 - Data access with Entity Framework Core - ASP.NET Core: From 0 to overkill"
summary: 'In this episode, we replace our current groups "persistence" with an actual database, accessing it with Entity Framework Core.'
image: '/assets/2019/01/16/aspnet-core-from-zero-to-overkill-e011.jpg'
categories:
- fromzerotooverkill
tags:
- dotnet
- aspnetcore
---

In this episode, we replace our current groups "persistence" with an actual database, accessing it with Entity Framework Core.

For the walk-through you can check the next video, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/gb2SOufcbQg" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In this post we'll replace the in-memory "persistence" we've been using until now. We'll go with a PostgreSQL database, using Entity Framework Core to implement the data access code.

### Repository pattern
We won't implement the repository pattern right now, instead using EF directly in the business layer. Some say implementing the repository pattern isn't really needed when using EF because it does it already - a `DbContext` acts like a unit of work and a `DbSet` as a repository. 

Even though this is true, and EF has some facilities to allow for testing, I think implementing the repository pattern simplifies testing a bit more, and, of course, keeps the code decoupled from the concrete ORM used. Check out Steve Smith's tip on ["024: Do I Need a Repository?"](http://www.weeklydevtips.com/024).

This, as most of the times in software development, comes down to compromises, and if you feel implementing the pattern isn't worth the extra work, and being more coupled to the ORM isn't worrisome, by all means, roll with it.

### Running PostgreSQL
I'm using PostgreSQL in a Docker container, but of course this isn't needed, I just do it to keep the computer clean, instead of installing everything we want to test, we can install only Docker and just run what we want to test in containers. If you want to try it, the command I used to start the PostgreSQL container was the following.

```docker run -p 5432:5432 --restart unless-stopped --name postgres -e POSTGRES_USER=user -e POSTGRES_PASSWORD=pass -d postgres```

I won't go into more details because we'll play around with Docker later.

## The data access project
To keep all the data access code for the group management service, we create a new class library project (targeting .NET Standard 2.0), called `CodingMilitia.PlayBall.GroupManagement.Data`. To this new project we need to add a couple of dependencies:
- `Microsoft.EntityFrameworkCore` - which is easily identifiable as Entity Framework Core´s package
- `Npgsql.EntityFrameworkCore.PostgreSQL` - which allows us to use EF Core with a PostgreSQL database

## Creating the entities and DbContext
So far we only have one entity, which is the group. So we create a folder named `Entities` in the new project and add a `GroupEntity` class to it.

`GroupEntity.cs`
```csharp
public class GroupEntity
{
    public long Id { get; set; }
    public string Name { get; set; }
}
```

You probably recognize this code, as it's basically the same as we have in the API and business layer.

Now to create the `DbContext` which will allow us to "talk" to the database, in the root of the project we add a new class called `GroupManagementDbContext`, that inherits from `DbContext`.

`GroupManagementDbContext.cs`
```csharp
public class GroupManagementDbContext : DbContext
{
    public DbSet<GroupEntity> Groups { get; set; }

    public GroupManagementDbContext(DbContextOptions<GroupManagementDbContext> options) : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasDefaultSchema("public");
        
        // Configure entities ...
    }
}
```

In this class, we create a property of type `DbSet<T>` for each entity we want available in the context, which right now is only `GroupEntity`. 

We also create a constructor that receives a `DbContextOptions<GroupManagementDbContext>`, which will allow us to setup the `DbContext` in the dependency injection container later, using the provided extension methods.

Finally we have the `OnModelCreating`, which allows us to configure the aspects of the entities, like specific column mappings, keys, indexes and more, if the conventions aren't enough. 

Note that in the first line of `OnModelCreating` we're calling `HasDefaultSchema` with `"public"`. I'll be honest about this one, I'm not really sure this is still needed, maybe I should test it and make sure 😛, but the reason I've become accustomed to using it is because by default, in SQL Server, the schema is `dbo`, while in Postgres it is `public`, and I remember having issues with this in the past (before even .NET Core existed).

Now for configuring the entities, we have a couple of options - data annotations and fluent API. I prefer the fluent API, as it allows for the entities to be clean, without EF specific "pollution". To use the fluent API, all the code was normally put directly in the `OnModelCreating` method, which got ugly fast. But recently, I think in EF Core 2.0, there is the `IEntityTypeConfiguration<T>` interface, which we can implement for each entity we want to configure. In a new folder named `Configurations` we add a new class `GroupEntityConfiguration`.

`GroupEntityConfiguration.cs`
```csharp
internal class GroupEntityConfiguration : IEntityTypeConfiguration<GroupEntity>
{
    public void Configure(EntityTypeBuilder<GroupEntity> builder)
    {
        builder
            .HasKey(e => e.Id);

        builder
            .Property(e => e.Id)
            .UseNpgsqlSerialColumn();
    }
}
```

For the first line of the `Configure` method we're saying that the `Id` property, or more exactly, the `Id` column that it will map to, will be the primary key of the table. This isn't really needed, as by convention a property named `Id` would be assumed as such by EF, but it doesn't hurt 🙃

Right afterwards, we're saying that the `Id` column should be a `SERIAL` column, which is automatically assigned, and incremented on each insertion - spoiler alert, later I found out I should have been using `UseNpgsqlIdentityAlwaysColumn` instead, that acts as we're used to in `IDENTITY` columns of SQL Server.

To wrap up model configuration, we need to use this `GroupEntityConfiguration` class. I've done the simplest thing, by adding it manually in `OnModelCreating`, but if there were many entities, this would get tedious fast, so maybe some reflection shenanigans to auto-discover these classes would be nice, but for now, keeping it simple.

`GroupManagementDbContext.cs`
```csharp
public class GroupManagementDbContext : DbContext
{
    //...
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasDefaultSchema("public");
        modelBuilder.ApplyConfiguration(new GroupEntityConfiguration());
    }
}
```
## Adapting the business layer
Now that we have our (massively complex) entities and `DbContext` prepared, we can adapt the business layer to use the database instead of the "in-memory persistence".

Nothing really changes for the interface, but we can now delete the `InMemoryGroupsService` class and create a new one called `GroupsService` (also thought of `EfGroupsService`, but I think there's no need for the extras).

First of all, we create a constructor that receives an instance of `GroupManagementDbContext`.

`GroupsService.cs`
```csharp
public class GroupsService : IGroupsService
{
    private readonly GroupManagementDbContext _context;

    public GroupsService(GroupManagementDbContext context)
    {
        _context = context;
    }

    // ...
}
```

Then we can start with the read methods, `GetAllAsync` and `GetByIdAsync`.

`GroupsService.cs`
```csharp
public class GroupsService : IGroupsService
{
    // ...

    public async Task<IReadOnlyCollection<Group>> GetAllAsync(CancellationToken ct)
    {
        var groups = await _context.Groups.AsNoTracking().OrderBy(g => g.Id).ToListAsync(ct);
        return groups.ToService();
    }

    public async Task<Group> GetByIdAsync(long id, CancellationToken ct)
    {
        var group = await _context.Groups.AsNoTracking().SingleOrDefaultAsync(g => g.Id == id, ct);
        return group.ToService();
    }

    // ...
}
```

Some notes on these:
- We use the `_context` to access the `Groups` `DbSet<Group>` which acts as the repository to query the database.
- On `GetAllAsync` we're ordering by `Id` just to have a consistent order, as if we didn't order it, it would be whatever the database engine would please, and could end up not being consistent.
- Notice the `AsNoTracking` method call. This tells EF that we don't need it to track the fetched entities, as we are going to use them for readonly purposes. This is an optimization, as we avoid EF doing more work than we need.
- The `ToService` method is pretty much the same as `ToViewModel` we used in previous episodes, in this case converting the database entity to its service representation.

`UpdateAsync` and `AddAsync` are also pretty simple, but particularly `UpdateAsync` has some things to pay special attention.

`GroupsService.cs`
```csharp
public class GroupsService : IGroupsService
{
    // ...

    public async Task<Group> UpdateAsync(Group group, CancellationToken ct)
    {
        var updatedGroupEntry = _context.Groups.Update(group.ToEntity());
        await _context.SaveChangesAsync(ct);
        return updatedGroupEntry.Entity.ToService();
    }

    // ...
}
```

I think this is the simplest and most efficient way to implement the update operation, but there are some things to keep in mind:
- We could fetch the entity first and then update its properties before calling `SaveChangesAsync`, but this would mean that for one update, we would do two round trips to the database. If we can do it in one round trip, better.
- We can do `_context.Groups.Update` with the `group` we got as argument because it has all the entity data. If we only wanted a partial update, we would need to fetch the existing entity from the database before updating it.
- Doing `_context.Groups.Update` is equivalent to doing `_context.Entry(entity).State = EntityState.Modified`, which is to say, add the entity to EF tracker and mark it as modified. If we already had the entity in the context (because of a fetch without `AsNoTracking` for instance) we would get an `InvalidOperationException`.

Finally, we have `AddAsync`, which looks just like `UpdateAsync`, only using the `DbSet`s `Add` method instead of `Update`.

`GroupsService.cs`
```csharp
public class GroupsService : IGroupsService
{
    // ...

    public async Task<Group> AddAsync(Group group, CancellationToken ct)
    {
        var addedGroupEntry = _context.Groups.Add(group.ToEntity());
        await _context.SaveChangesAsync(ct);
        return addedGroupEntry.Entity.ToService();
    }
}
```

## Configuring EF in the web application
With the entities prepared and the business layer using them, we need to configure the web application to use it all, namely registering the `DbContext` in the DI container and adding the connection string to configuration.

The changes here are small. Let's start with DI registration where we simply need to make use the `AddDbContext` extension method.

`Startup.cs`
```csharp
public class Startup
{
    // ...
    
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddMvc();

        services.AddDbContext<GroupManagementDbContext>(options =>
        {
            options.UseNpgsql(_config.GetConnectionString("GroupManagementDbContext"));
        });

        services.AddBusiness();
    }

    // ...
}
```

We provide `AddDbContext` with the the concrete type of the context we created, and use the options builder to make it use PostgreSQL with the connection string we still need to add to the configurations. The `GetConnectionString` extension method assumes there is a configuration section named `ConnectionStrings`, and then fetches the value for the name we provided.

Ideally, a connection string should be treated as a secret, so a good candidate to use the secret manager tool, but since we're just using a local dev database, with the simplest of credentials, it's just easier to put the connection string in the `appsettings.development.json` file.

`appsettings.development.json`
```json
{
  "OtherSections": "...",

  "ConnectionStrings": {
    "GroupManagementDbContext": "server=localhost;port=5432;user id=user;password=pass;database=GroupManagement"
  }
}
```
Now, on startup, the application will be able to read the configuration and pass it along to the `DbContext` setup.

## Migrations
### What are migrations
Before we run the application, we need to create migrations that will allow us to create/update the database schema. 

Being more precise, we don't need migrations to get started, we could just do `dbContext.Database.EnsureCreated` at the start of the application, and it would just create the database taking into account the state of our entities configuration at that moment. We could even do `EnsureDeleted` (also from `dbContext.Database`) before `EnsureCreated`, so we always have a clean database, but this is only really useful for a dev/test environment.

Migrations, on the other hand, allow us to keep the various changes we do to the database schema, which is important when we have the database in production and want to update it to match new data requirements. Each time we apply migrations to the database, the information about the applied migrations is kept, so the next time we need to change the schema, only the new migrations are applied. In the database, you'll see a table named `__EFMigrationsHistory`, which contains the history of applied migrations.

### Creating migrations
If we weren't using Entity Framework Core (or another ORM/migration utility) we would probably need to implement the migrations manually, i.e. write SQL scripts to make changes to the database schema. But when using EF migrations, we just need to make use of the command line tool to create the migrations, and it will generate the scripts - C# scripts in this case, which are translated to SQL when the migrations are applied - inferring the changes by comparing the current status of the entities with the status at the time of the last created migration (which doesn't exist when we create the first migration).

To create the migration, we head on to the database project folder in the command line and type in `dotnet ef migrations add InitialCreate`.

Supposedly this would create a migration, but instead we get an error:
```
Startup project 'CodingMilitia.PlayBall.GroupManagement.Data.csproj' targets framework '.NETStandard'. There is no runtime associated with this framework, and projects targeting it cannot be executed directly. To use the Entity Framework Core .NET Command-line Tools with this project, add an executable project targeting .NET Core or .NET Framework that references this project, and set it as the startup project using --startup-project; or, update this project to cross-target .NET Core or .NET Framework. For more information on using the EF Core Tools with .NET Standard projects, see https://go.microsoft.com/fwlink/?linkid=2034781
```

So, to summarize this wall of text, to run the migrations we need a project that targets .NET Framework or .NET Core, as .NET Standard can't be run directly. There are different ways to solve this, but I just went with the proposed solution and used the web project (`CodingMilitia.PlayBall.GroupManagement.Web`) for startup like follows:

```dotnet ef migrations add InitialCreate -s ../CodingMilitia.PlayBall.GroupManagement.Web/CodingMilitia.PlayBall.GroupManagement.Web.csproj```

Using the main project as startup for migrations, has the benefit of being able to reuse the configurations already in place in it to run the migrations as well, namely the dependency injection `DbContext` configuration, as this or another supported way to create a `GroupManagementDbContext` is required to create the migrations (more on this [here](https://docs.microsoft.com/en-us/ef/core/miscellaneous/cli/dbcontext-creation)).

## Ensuring database up to date
We have done quite a bit so far and are almost ready to start the application. Summarizing, we created the entities, created the `DbContext`, prepared the business logic to use EF and created migrations to keep the database in sync with the code. Now just before starting the application, we need to apply the migrations we created.

I can think of 3 possibilities to go around doing this (there are probably more though):
1. Run the migrations with EF's command line tools
2. Creating an helper application to run the migrations
3. Run the migrations when the web application starts

### Run the migrations with EF's command line tools
The first option is the easiest and when we're creating the migrations and want to test them, it's the best. In this case we would simply need to execute the following command to apply the migrations:

```
dotnet ef database update -s ..\CodingMilitia.PlayBall.GroupManagement.Web\CodingMilitia.PlayBall.GroupManagement.Web.csproj
```

It's pretty similar to the command to create the migrations, in regards to needing a runnable startup project specified.

This approach is good, but because I'm running the database in Docker and am always deleting and creating the container/database, it gets tiresome to always be running this manually, so I want a more automated workflow.

### Creating an helper application to run the migrations
Although I'm not going to create a sample of this option, I wanted to mention it as it's a nice idea from a colleague of mine.

The gist of it is to create a simple console application that all it has to do, is applying the migrations. This is nice because we can just provide the binaries to someone and it's just a matter of running the executable file.

Although I'm not following this approach in this project, the code to achieve it is pretty similar to the third option, with the main difference this one would be in a standalone migrations application and the other is contained in the web application itself.

### Run the migrations when the web application starts
The best way to make sure we have the database up to date - at least in development, in production this might not be the best idea - is to run the migrations when the application starts. This way, whenever we make changes to the database, when we start the application to test them, they will be automatically applied.

In the web application project, we create a new folder called `StartupHelpers`, then created a `DatabaseExtensions` static class, with the method `EnsureDbUpToDate` (it's missing the `Async` suffix, I'll fix it when I remember 😛).

`DatabaseExtensions.cs`
```csharp
namespace Microsoft.AspNetCore.Hosting
{
    internal static class DatabaseExtensions
    {
        internal static async Task EnsureDbUpToDate(this IWebHost host)
        {
            using (var scope = host.Services.CreateScope())
            {
                var context = scope.ServiceProvider.GetRequiredService<GroupManagementDbContext>();
                await context.Database.MigrateAsync();
            }
        }
    }
}
```

The method is an extension on `IWebHost` because we need to resolve dependencies, and prior to the application start, this is the way to do it, as we have access to the `IServiceProvider`, containing the dependencies configured in `Startup.ConfigureServices`.

Before we get the `DbContext` from the provider, we need to create a scope. If we don't create a scope, the resolved `DbContext` will live on as a singleton in the DI container. By creating a scope, we are basically doing the same as the framework when processing a request, creating a scope that starts when the request arrives and is disposed when the response is sent back.

Now with a scope, we can get another `IServiceProvider` in its context that'll provide us with the correctly scoped dependencies. We can use 
`GetRequiredService<T>` or `GetService<T>`, being the difference that the former throws if what we requested can't be resolved, and the latter doesn't.

After all of this, just to use the DI container the best way possible, we get the `DbContext` and can call `context.Database.MigrateAsync` to apply any pending migrations.

As a side note, which I think I already pointed out in a past post of this series, take a look at the namespace. This is not really needed, it just helps with discoverability, as in the piece of code where we have the `IWebHost` we don't need any additional `using`, we will immediately see our newly created method.

To wrap it up, we need to call the new method, so we change the `Program` class `Main` method.

`Program.cs`
```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var host = CreateWebHostBuilder(args).Build();
        await host.EnsureDbUpToDate();
        host.Run();
    }

    //...
}
```

For starters the method signature changed and it is now an async method. To be able to use this alternative main method signature, we need to be using C# 7.1 or greater. To have it I simply added `<LangVersion>7.3</LangVersion>` to the `csproj` file, just below the framework version (`<TargetFramework>netcoreapp2.2</TargetFramework>`).

Then instead of running the host immediately, we call (awaiting it) the new `EnsureDbUpToDate` method.

Now we should finally be able to run the application, having it behave as before introducing EF Core, except for having real persistence now 🙂.

## Optimistic concurrency
One final subject I would like to touch before wrapping up, is the possibility of using optimistic concurrency mechanisms to minimize concurrent database change issues, while avoiding the use of locks.

To achieve this, we'll add a property to the `GroupEntity` to act as a concurrency token. With it in place, each time we do an update, EF will check if the token has changed, and if it has, the update won't go through and a `DbUpdateConcurrencyException` exception will be thrown.

Heading to the `GroupEntity` class, we add a new property called `RowVersion`.

`GroupEntity.cs`
```csharp
public class GroupEntity
{
    public long Id { get; set; }
    public string Name { get; set; }
    public uint RowVersion { get; set; }
}
```

Note that the type I'm using (`uint`) is very specific to the fact I'm using PostgreSQL. In SQL Server the usual type is a `byte[]` (for more information on optimistic concurrency, particularly with SQL Server examples, take a look at [these docs](https://docs.microsoft.com/en-us/ef/core/modeling/concurrency)).

Then we must tell EF to use the new property as concurrency token. This in done in the `GroupEntityConfiguration` class.

`GroupEntityConfiguration.cs`
```csharp
internal class GroupEntityConfiguration : IEntityTypeConfiguration<GroupEntity>
{
    public void Configure(EntityTypeBuilder<GroupEntity> builder)
    {
        //...

        builder
            .Property(e => e.RowVersion)
            .HasColumnName("xmin")
            .HasColumnType("xid")
            .ValueGeneratedOnAddOrUpdate()
            .IsConcurrencyToken();
    }
}
```

The code to configure the property as concurrency token with PostgreSQL is a bit more cumbersome than it would be with SQL Server (in which a call to the `IsRowVersion` method would suffice). From what I understand, in PostgreSQL there aren't auto-updating properties, we would need to use triggers to achieve it. There is however a column (`xmin`) that's present in all tables (even though we don't see it with a simple `select *`), and it auto updates every time the row changes, so we can use it for checking concurrent accesses.

There is a helper method from Npgsql, `ForNpgsqlUseXminAsConcurrencyToken`, that would configure all I did above in one line, with one caveat, it would use a [shadow property](https://docs.microsoft.com/en-us/ef/core/modeling/shadow-properties) - one that is mapped to the database and used by the EF model, but not mapped in the entity class, hence not visible to the calling code - instead, and we don't want that in this case.

With a shadow property, we could only detect concurrency issues in the scope of one request, which is good, as it can avoid the need for transactions, but we might want to take it a little further. By using a normal property and letting it flow all the way to the frontend, when we get an update request for that entity later, we can check if the entity was changed since the client originally read the values. This can be useful if we want to avoid multiple users to edit the same entity concurrently, making sure they see the latest values before making changes.

The rest of the code, namely the business layer that uses the data layer, can remain mostly the same, we just need to add the `RowVersion` property to all the `Group` representations, map it in the layer transitions (from database to service and then to view model) and add it to the edit view, so it is included in the update request (so, adding `<input asp-for="RowVersion" type="hidden" />` to `Details.cshtml`).

Now if we, for instance, open the same group detail in a couple of tabs, update one and then try to update the other, we're "greeted" with an error.

A final note on this subject, regarding the update code. Because `UpdateAsync` is attaching the complete entity that came from the frontend, it works as expected. If, on the other hand, our update code fetched the entity from the database first and then updated the values, we would need to do something, that at least for me wasn't obvious. Imagine that `UpdateAsync`'s code is as follows:

```csharp
public async Task<Group> UpdateAsync(Group group, CancellationToken ct)
{
    var entityToUpdate = await _context.Groups.SingleAsync(g => g.Id == group.Id, ct);
    entityToUpdate.Name = group.Name;
    entityToUpdate.RowVersion = group.RowVersion;
    await _context.SaveChangesAsync(ct);
    return entityToUpdate.ToService();
}
```

The code as presented above, would only detect concurrency issues between the time the entity is fetched and then updated, and not take into consideration the `RowVersion` that came with the request. For it to be considered, we would need to do something like the following:

```csharp
public async Task<Group> UpdateAsync(Group group, CancellationToken ct)
{
    // ...
    _context.Entry(entityToUpdate).OriginalValues.SetValues(new Dictionary<string, object> { { "RowVersion", group.RowVersion} });
    await _context.SaveChangesAsync(ct);
    // ...
}
```
This isn't the prettiest way to do it, but it illustrates the point. The value used for concurrency check is the one kept in EF's change tracker as original value, not the one that's accessible through the entity. Therefore, if we want the update code fetching the entity first, we need to do this change directly. In the original implementation we don't have this problem because we're attaching the object, so the original values are the ones we provide.

## Outro
As usual in this series, there's a lot more to it than what we talked in this post, but it's more than enough to get started (and I probably went on to more complex things than needed). We'll get back to Entity Framework Core in future episodes, as we'll certainly need to add more things to our database for the application to get useful

Links in the post:
- Steve Smith's tip on ["024: Do I Need a Repository?"](http://www.weeklydevtips.com/024)
- [Entity Framework Core](https://docs.microsoft.com/en-us/ef/core/)
    - [Migrations](https://docs.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
    - [Design-time DbContext Creation](https://docs.microsoft.com/en-us/ef/core/miscellaneous/cli/dbcontext-creation)
    - [Concurrency Tokens](https://docs.microsoft.com/en-us/ef/core/modeling/concurrency)
    - [Shadow Properties](https://docs.microsoft.com/en-us/ef/core/modeling/shadow-properties)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode011).

Any questions and feedback are welcomed and appreciated.

Thanks for stopping by, cyaz!