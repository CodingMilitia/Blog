---
author: João Antunes
date: 2018-11-04 11:30:00+00:00
layout: post
title: "Episode 005 - Dependency Injection - ASP.NET Core: From 0 to overkill"
summary: "In this episode we go a little bit deeper into dependency injection, extracting our logic from the controller and into a service."
images:
- '/images/2018/11/04/aspnet-core-from-zero-to-overkill-e005.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-005-from-zero-to-overkill-dependency-injection
---

In this post/video we continue our look into base concepts of ASP.NET Core, in preparation to what's in our way for building the application. This time, dependency injection, which was built right into the core of ASP.NET Core (see what I did there? 😆).

For the walk-through you can check the next video, but if you prefer a quick read, skip to the written synthesis.

{{< yt Dj8CKHokDsQ >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In the last episode, we started looking into some important core concepts when developing ASP.NET Core applications, namely the `Program` and `Startup` classes, taking also a quick look into dependency injection and middlewares. In this episode, we go a little further with dependency injection.

## Extracting logic from the controller
### Why
Normally we don't want our controllers to have too much logic, particularly business logic. Ideally the controllers focus on MVC/HTTP specific logic - receiving client inputs, pass them along to something responsible for the business logic, return the results back (feeding views or in other forms, such as a JSON payload), maybe including different HTTP status codes depending on the output of the business logic.

This, of course, isn't mandatory, and I've seen articles and talks arguing that for instance, if we're developing really small microservices, we might as well bypass splitting the code this way, as if adjustments are needed we could just rewrite the service. Anyway, I'll go with the more classic approach of splitting things, just wanted to make the note that this isn't set in stone (as nothing is really, there are always multiple ways of doing things).

Some benefits of having the business logic separated from the web framework are the ability to test it without the complexity added by the web framework specificities, as well as allowing the use of the same logic components with different "gateways" (an alternative web framework, a desktop application, etc).

### New class libraries
To extract the business logic from the controller, we'll go along with the old school 3-tier architecture, where our MVC application will be the presentation layer and the extracted logic will make up the business layer. No data layer yet, we'll get to it in the future.

N-tier architectures may not be what's hot right now, but for what we're doing right now, it fits our needs and is simple enough.

Starting with the creation of couple of class libraries `CodingMilitia.PlayBall.GroupManagement.Business` and `CodingMilitia.PlayBall.GroupManagement.Business.Impl`. The first library will contain the contracts/API that the clients of the business logic need to know, while the latter will contain the implementation of said logic.

In the contracts library, we need the models (`Group` class only in this case) and the `IGroupService` interface.

```csharp
public class Group
{
    public long Id { get; set; }
    public string Name { get; set; }
}
```

```csharp
public interface IGroupsService
{
    IReadOnlyCollection<Group> GetAll();

    Group GetById(long id);

    Group Update(Group group);

    Group Add(Group group);
}
```

Going to the service implementation library, we create a `InMemoryGroupsService` class that implements the `IGroupsService` interface - there isn't much business logic right now, it's mostly CRUD, but let's pretend there is 😛

```csharp
public class InMemoryGroupsService : IGroupsService
{
    private readonly List<Group> _groups = new List<Group>();
    private long _currentId = 0;
    
    public IReadOnlyCollection<Group> GetAll()
    {
        return _groups.AsReadOnly();
    }

    public Group GetById(long id)
    {
        return _groups.SingleOrDefault(g => g.Id == id);
    }

    public Group Update(Group group)
    {
        var toUpdate = _groups.SingleOrDefault(g => g.Id == group.Id);

        if (toUpdate == null)
        {
            return null;
        }

        toUpdate.Name = group.Name;
        return toUpdate;
    }

    public Group Add(Group group)
    {
        group.Id = ++_currentId;
        _groups.Add(group);
        return group;
    }
}
```

As the name implies, the data is stored in memory (it's not even thread safe) so to say it's far from production ready is an understatement. It's exactly the same logic we had in the controller, just pulled into a class of its own.

Heading back to the web application project, we add the references to the newly created libraries and rework the `GroupsController` to use the `IGroupsService` instead of having the logic implemented in it.

```csharp
[Route("groups")]
public class GroupsController : Controller
{
    private readonly IGroupsService _groupsService;

    public GroupsController(IGroupsService groupsService)
    {
        _groupsService = groupsService;
    }
    
    [HttpGet]
    [Route("")]
    public IActionResult Index()
    {
        return View(_groupsService.GetAll().ToViewModel());
    }
    
    [HttpGet]
    [Route("{id}")]
    public IActionResult Details(long id)
    {
        var group = _groupsService.GetById(id);

        if (group == null)
        {
            return NotFound();
        }

        return View(group.ToViewModel());
    }

    [HttpPost]
    [Route("{id}")]
    [ValidateAntiForgeryToken]
    public IActionResult Edit(long id, GroupViewModel model)
    {
        var group = _groupsService.Update(model.ToServiceModel());

        if (group == null)
        {
            return NotFound();
        }

        return RedirectToAction("Index");
    }

    [HttpGet]
    [Route("create")]
    public IActionResult Create()
    {
        return View();
    }

    [HttpPost]
    [Route("")]
    [ValidateAntiForgeryToken]
    public IActionResult CreateReally(GroupViewModel model)
    {
        _groupsService.Add(model.ToServiceModel());

        return RedirectToAction("Index");
    }
}
```

A couple of things to notice:
- We're now receiving the `IGroupsService` in the constructor, so we can use it in the actions
- `ToViewModel` and `ToServiceModel` extension methods - as the presentation layer and the business layer use a different set of models, we need to convert them when passing them along

```csharp
public static class GroupMappings
{
    public static GroupViewModel ToViewModel(this Group model)
    {
        return model != null ? new GroupViewModel { Id = model.Id, Name = model.Name } : null;
    }
    
    public static Group ToServiceModel(this GroupViewModel model)
    {
        return model != null ? new Group { Id = model.Id, Name = model.Name } : null;
    }

    public static IReadOnlyCollection<GroupViewModel> ToViewModel(this IReadOnlyCollection<Group> models)
    {
        if (models.Count == 0)
        {
            return Array.Empty<GroupViewModel>();
        }
        
        var groups = new GroupViewModel[models.Count];
        var i = 0;
        foreach (var model in models)
        {
            groups[i] = model.ToViewModel();
            ++i;
        }

        return new ReadOnlyCollection<GroupViewModel>(groups);
    }
}
```

We could do this mapping directly in the controller, but it would end up being polluted by this boilerplate code. We could also use [AutoMapper](https://github.com/AutoMapper/AutoMapper) to do this auto-magically for us, but it can sometimes hide problems in our code (for instance, we can't be sure of the existence of references to properties when using it). All in all, I like this extension method idea by a colleague of mine, as it provides nice readability in the controller, so I'll go with it.

If we try to run this now we'll get an error, because we're now expecting to get an `IGroupsService` in the controller, but we haven't told the framework how to get it. To do this we need to register the service implementation in the dependency injection container.

## Using the builtin container
### Registering the service
In the previous episode we saw how to register services in the builtin dependency injection container, and this will be no different. We want to register the `InMemoryGroupsService` and like we also saw previously, we want it to have lifestyle of type singleton, so the data it keeps in memory sticks around for the lifetime of the application.

To do this, we need a single new line added to the `Startup` class: `services.AddSingleton<IGroupsService, InMemoryGroupsService>();`.

Now we can run the application and it behaves has before, just with a different internal organization.

### Improve organization
For now, we don't have too much to worry, as we only have a couple of lines in the `ConfigureServices` method of the `Startup` class, but what about when we add more and more?

One good way of keeping the `ConfigureServices` tidy is to create extension methods on `IServiceCollection` to register groups of services that go along together. With this in mind, in a newly created `IoC` folder, we add a new class `ServiceCollectionExtensions`.

```csharp
namespace Microsoft.Extensions.DependencyInjection
{
    public static class ServiceCollectionExtensions
    {
        public static IServiceCollection AddBusiness(this IServiceCollection services)
        {
            services.AddSingleton<IGroupsService, InMemoryGroupsService>();
            
            //more business services...
    
            return services;
        }
    }
}
```

Notice the namespace "trick" in there, using the same namespace as `IServiceCollection`. This is something usually done to ease discoverability of 
extension methods, as it allows for us to go to the `ConfigureServices` method, type `services.` and get immediately the methods in intellisense without the need to add another `using`.

Now in the `ConfigureServices` we can replace the line we added previously with `services.AddBusiness();`.

## Using third party containers
It's good that ASP.NET Core comes with dependency injection built right into it, but we need to keep in mind that the out of the box container was built to answer ASP.NET Core internal needs, not every need of the applications built on top of it.

For more advanced scenarios we might have, we can use third party DI containers that can provide those features.

Just to check out this possibility, we'll add [Autofac](https://github.com/autofac/Autofac) to the project and play a little with it. The objective is not to go into much detail on Autofac itself, but see how to use it with ASP.NET Core and an example of extra features it adds.

### Replacing the container
First thing is to add a couple of NuGet packages - `Autofac` and `Autofac.Extensions.DependencyInjection`.

To register dependencies with Autofac, we create a new class `AutofacModule` (in the `IoC` folder) that inherits from `Module` and we override the `Load` method.

```csharp
public class AutofacModule : Module
{
    protected override void Load(ContainerBuilder builder)
    {
        builder
            .RegisterType<InMemoryGroupsService>()
            .As<IGroupsService>()
            .SingleInstance();
    }
}
```

The code seen above is the equivalent of what we had in `AddBusiness`, but adapted to Autofac's API. Now we need to configure Autofac as the container.

```csharp
//...
public IServiceProvider ConfigureServices(IServiceCollection services)
{
    services.AddMvc();
    
    //if using default DI container, uncomment
    //services.AddBusiness();
    
    // Add Autofac
    var containerBuilder = new ContainerBuilder();
    containerBuilder.RegisterModule<AutofacModule>();
    containerBuilder.Populate(services);
    var container = containerBuilder.Build();
    return new AutofacServiceProvider(container);
}
//...
```

The first change to notice is the `ConfigureServices` method is no longer `void`, it now returns an `IServiceProvider`. This is needed so ASP.NET Core uses the returned service provider instead of building its own out of the `IServiceCollection` that's received as an argument.

The Autofac configuration code by the end of the method is a plain copy paste from Microsoft's dependency injection docs 😇

Basically, what we're doing here is:
- Creating an Autofac container builder
- Registering the module we created with our dependency configurations
- Populating Autofac's container with the registrations present in the `IServiceCollection`, put there by ASP.NET Core's components
- Building the container
- Return a new service provider, implemented using the Autofac container

Again, rerunning the application, we maintain the same functionality.

### Using other container features
So far we haven't used any specific feature of Autofac that justifies the change, unless it was a matter of performance (which wouldn't be the case, at least at the time of writing, the builtin container seems faster, see [here](https://github.com/danielpalme/IocPerformance)).

Just for arguments sake, let's use an extra feature Autofac provides: decorators.

```csharp
public class AutofacModule : Module
{
    protected override void Load(ContainerBuilder builder)
    {
        builder
            .RegisterType<InMemoryGroupsService>()
            .Named<IGroupsService>("groupsService")
            .SingleInstance();
        builder
            .RegisterDecorator<IGroupsService>(
            (context, service) => new GroupsServiceDecorator(service), 
            "groupsService");
    }

    private class GroupsServiceDecorator : IGroupsService
    {
        private readonly IGroupsService _inner;

        public GroupsServiceDecorator(IGroupsService inner)
        {
            _inner = inner;
        }
        
        public IReadOnlyCollection<Group> GetAll()
        {
            Console.WriteLine($"######### Helloooooo from {nameof(GetAll)} #########");
            return _inner.GetAll();
        }

        public Group GetById(long id)
        {
            Console.WriteLine($"######### Helloooooo from {nameof(GetById)} #########");
            return _inner.GetById(id);
        }

        public Group Update(Group group)
        {
            Console.WriteLine($"######### Helloooooo from {nameof(Update)} #########");
            return _inner.Update(group);
        }

        public Group Add(Group group)
        {
            Console.WriteLine($"######### Helloooooo from {nameof(Add)} #########");
            return _inner.Add(group);
        }
    }
}
```

The decorator implements the `IGroupsService` interface, writing to the console on every method call, then delegating the real work to another `IGroupsService` implementation it gets in the constructor.

To register the decorator, we changed the `InMemoryGroupsService` registration to a named one, and then call `RegisterDecorator` with a lambda to build the decorator and the name we added to the real implementation's registration. Can't say I'm the biggest fan of this API, but that's what we have right now.

Now if we rerun the application, we keep the same functionality as before, but additionally if we look at the console, intermixed with the logs we see the decorators messages.

## Outro
This is it for a quick look at dependency injection in ASP.NET Core. Even though not going very deep it's good enough to prepare for what's coming, and we'll introduce more related concepts as needed along the way.

Using Autofac was good for an introduction on replacing the builtin container, but I'm not entirely sold on its API, I'll probably replace it in the future.

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode005).

Please send any feedback you have, so the next posts/videos can be better and even adjusted to more interesting topics.

Thanks for stopping by, cyaz!