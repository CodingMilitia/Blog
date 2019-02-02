---
author: João Antunes
date: 2018-12-17 20:00:00+00:00
layout: post
title: "Episode 010 - Async all the things - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we take a look at using async await in ASP.NET Core, why it's important, the happy path and some sad paths, along with some interesting bits."
image: '/assets/2018/12/17/aspnet-core-from-zero-to-overkill-e010.jpg'
categories:
- fromzerotooverkill
tags:
- dotnet
- aspnetcore
---

In this episode, we take a look at using async await in ASP.NET Core, why it's important, the happy path and some sad paths, along with some interesting bits.

For the walk-through you can check the next couple of videos, but if you prefer a quick read, skip to the written synthesis.

Main video:
{% youtube "https://youtu.be/CGi1bQgaqwg" %}

A short addendum added later:
{% youtube "https://youtu.be/bWyB0VFKeKA" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
Let's start this post on a different note from the others. Instead of going right to doing async/await stuff, why do we do it?
When we didn't bother with all this `async`, `await`, `Task` and so on, life was easier 😛

From a web application point of view, all of this matters to improve the scalability, by not having lots of resources (threads) blocked for no reason (other types of applications may have different reasons, like in desktop applications not blocking the UI thread).

Before even the introduction of `Task`s with the TPL ([Task Parallel Library](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-parallel-library-tpl])) and the TAP ([Task-based Asynchronous Pattern](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)) that came after it, there were other ways to achieve this, namely with the APM ([Asynchronous Programming Model](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/asynchronous-programming-model-apm?view=netframework-4.7.2)), but it was not as simple to do. So much like JS has recently introduced async/await to simplify the usage of promises, .NET did the same a while back.

I'll drop here a couple of diagrams for a very high level overview of the request handling behavior in a non-async scenario vs an async scenario. If you want a more thorough explanation of theses diagrams, please check out the first video I linked/embedded at the start of the post.

**Without using async/await (or similar approaches)**

[![pre-async](/assets/2018/12/17/pre-async.jpg)](/assets/2018/12/17/pre-async.jpg)


**Using async/await (or similar approaches)**

[![async](/assets/2018/12/17/async.jpg)](/assets/2018/12/17/async.jpg)

In summary, using async/await in ASP.NET Core allows us to simplify the writing of more scalable code, particularly in regards to IO, by freeing resources to handle other tasks while, for instance, a database access or external service call is being done.

## Making the service async
Now that we have a better idea why we care about all of this (probably you already had anyway), let's start changing our existing code to be async where it makes sense, in preparation for the replacement of the in-memory groups "persistence" with an actual database. 

### Adapting the interface
Revisiting the current synchronous interface for the groups service, we have:

`IGroupsService.cs`
```csharp
public interface IGroupsService
{
    IReadOnlyCollection<Group> GetAll();

    Group GetById(long id);

    Group Update(Group group);

    Group Add(Group group);
}
```

We'll change all of the methods signature to the async counterpart, as we expect them all to do IO. If a method is not required to be async, then, don't make it 🙂

Rule of thumb is every method should return a `Task<T>` (or `Task` if void) and have `Async` as suffix (although some don't like to use this suffix).

**Note:** there are a couple of other options that we're not going to look at in this post
- Have `async void` methods, but this is maybe useful in other kinds of applications, like WinForms or Xamarin maybe, but not really in ASP.NET Core, [where it probably just brings problems](https://msdn.microsoft.com/en-us/magazine/jj991977.aspx)
- Return `ValueTask<T>` (or `ValueTask`) instead of `Task`, which is a recent addition that's particularly useful for scenarios where high performance is really important, and avoiding allocations can make a big difference

The async version of the interface ends up as follows:

`IGroupsService.cs`
```csharp
public interface IGroupsService
{
    Task<IReadOnlyCollection<Group>> GetAllAsync();

    Task<Group> GetByIdAsync(long id);

    Task<Group> UpdateAsync(Group group);

    Task<Group> AddAsync(Group group);
}
```

Of course, by changing the interface we must also change the calling code, which in this case is in the `GroupsController`. We'll take a look only at the (previously named) `Index` method, as the others follow the same recipe (but you can check out all the code on [GitHub](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode010)).

`GroupsController.cs`
```csharp
public class GroupsController : Controller
{
    private readonly IGroupsService _groupsService;

    public GroupsController(IGroupsService groupsService)
    {
        _groupsService = groupsService;
    }

    [HttpGet]
    [Route("")]
    public async Task<IActionResult> IndexAsync()
    {
        var result = await _groupsService.GetAllAsync();
        return View("Index", result.ToViewModel());
    }

    //...
    
}
```

So, considering what we already discussed, `Index` became `IndexAsync`, although in this case, I think leaving it at `Index` could have been a good option to simplify some things, as this forced the view name (to map to `Index.cshtml`) to be passed as an argument when calling the `View` method, as by default te action and view are expected to have the same name (maybe this will change in [ASP.NET Core 3.0](https://github.com/aspnet/Mvc/issues/6723)). 

Besides that, the method signature now has an `async` modifier, signifying there will be an `await` in the implementation and the return type is now `Task<IActionResult>`.

In the method implementation, now when invoking `_groupsService.GetAllAsync()`, we `await` its result.

### Implementing the service
With the interface and client code updated, we need to update our `InMemoryGroupsService`. Because we don't really have a async things to do, namely we don't go to a database yet, as we have the groups data stored in memory, the implementation of the service will be faking the async part. In the next post, when we add Entity Framework Core to the mix, we'll adapt the service again.

Let's revisit the non-async version:

`InMemoryGroupsService.cs`
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

Moving this to the (fake) async version, all we do is change the signature of all the methods, to match the interface, and instead of returning the result directly, we wrap it in a completed task by using the `Task.FromResult` method.

`InMemoryGroupsService.cs`
```csharp
public class InMemoryGroupsService : IGroupsService
{
    private static readonly Random RandomGenerator = new Random();
    private readonly List<Group> _groups = new List<Group>();
    private long _currentId = 0;

    public Task<IReadOnlyCollection<Group>> GetAllAsync(CancellationToken ct)
    {
        return Task.FromResult<IReadOnlyCollection<Group>>(_groups.AsReadOnly());
    }

    public Task<Group> GetByIdAsync(long id, CancellationToken ct)
    {
        return Task.FromResult(_groups.SingleOrDefault(g => g.Id == id));
    }

    public Task<Group> UpdateAsync(Group group, CancellationToken ct)
    {
        var toUpdate = _groups.SingleOrDefault(g => g.Id == group.Id);

        if (toUpdate == null)
        {
            return Task.FromResult(null);
        }

        toUpdate.Name = group.Name;
        return Task.FromResult(toUpdate);
    }

    public Task<Group> AddAsync(Group group, CancellationToken ct)
    {
        group.Id = ++_currentId;
        _groups.Add(group);
        return Task.FromResult(group);
    }
}
```

When implementing an async interface, but there isn't anything async to do in there, `Task.FromResult` is the way to go for `Task<T>` returning methods, `Task.CompletedTask` for the ones that return `Task`.

### No need for Task.Run
Looking at the `InMemoryGroupsService` implementation above, one might ask, why not do something like the following:

`InMemoryGroupsService.cs`
```csharp
public class InMemoryGroupsService : IGroupsService
{
    //...
    
    public async Task<IReadOnlyCollection<Group>> GetAllAsync(CancellationToken ct)
    {
        return await Task.Run(() => _groups.AsReadOnly());
    }

    //...
}
```

This also implements the async interface, but instead of returning the completed task, it asks the thread pool the run the code which is passed in to `Task.Run` in another thread, then we await its result. Well, I would say this is just wasting resources, we already have ready, running that in another thread just to make it async is wasteful.

Let's take a look at a less obvious scenario. Imagine we are already making a database call, but we're using a provider that for some reason doesn't support async. We could do something like:

`InMemoryGroupsService.cs`
```csharp
public class InMemoryGroupsService : IGroupsService
{
    //...
    
    public async Task<IReadOnlyCollection<Group>> GetAllAsync(CancellationToken ct)
    {
        return await Task.Run(() => GetAllUsingSyncDbProvider());
    }

    private IReadOnlyCollection<Group> GetAllUsingSyncDbProvider()
    {
        //...
    }

    //...
}
```

At first glance this may look like a nice idea, we're making the sync database access async(ish), by making it run in another thread. I would argue that, again, this is waste of resources. Instead of blocking the thread that's handling the request, we ask another thread to block. Blocking is never good, but since there's no other way, I would say just do it in the request handling thread, no need for the context switch overhead.

`Task.Run` should be used to do CPU bound work and eventually do some things in parallel. Keep in mind I'm talking about ASP.NET Core here, in desktop and Xamarin applications for instance, it might be important to use `Task.Run` to make sure the UI thread is not blocked (even on async methods, as even the asynchronous methods may have parts that run synchronously).

If I'm mistaken about this, and you feel there's an advantage to use `Task.Run` in these scenarios that I'm missing, please reach out, I appreciate it 🙂

You can check out more about `Task.Run` [here](https://blog.stephencleary.com/2013/11/taskrun-etiquette-examples-dont-use.html).

PS: `Task.Run` can also be used to do some shenanigans with synchronization contexts, but that's not really useful in ASP.NET Core (and aside from that, is basically a hack).

### Using cancellation tokens
One nice thing you can add to our async methods is the possibility of cancellation of an ongoing operation. This is done using `CancellationToken`s.

Cancellation tokens should be passed as arguments to async functions (by convention, the last argument). That would make the group service interface look like this:

`IGroupsService.cs`
```csharp
public interface IGroupsService
{
    Task<IReadOnlyCollection<Group>> GetAllAsync(CancellationToken ct);

    Task<Group> GetByIdAsync(long id, CancellationToken ct);

    Task<Group> UpdateAsync(Group group, CancellationToken ct);

    Task<Group> AddAsync(Group group, CancellationToken ct);
}
```

Then in the implementation we can pass it along to async method calls. Because the current implementation is faking the async part, I'll add a delay to fake an IO operation that uses the cancellation token (for brevity I included just one method as example).

`InMemoryGroupsService.cs`
```csharp
public class InMemoryGroupsService : IGroupsService
{
    //...
    
    public async Task<IReadOnlyCollection<Group>> GetAllAsync(CancellationToken ct)
    {
        await Task.Delay(5000, ct);
        return _groups.AsReadOnly();
    }

    //...
}
```

With this in place, if the request is cancelled during those 5 seconds, an `OperationCancelledException` is thrown, stopping the current executing code. Another way to cause this exception is calling `ct.ThrowIfCancellationRequested()`. This is useful if we're in a long running operation scenario and we want to check along the way if we should continue or not.

Now that we have the code prepared to be cancelled, how does it happen? In ASP.NET Core is really easy to get this to good use, we head onto our controller and add a cancellation token to the action methods, and the framework will pass it in.

`GroupsController.cs`
```csharp
public class GroupsController : Controller
{
    private readonly IGroupsService _groupsService;

    public GroupsController(IGroupsService groupsService)
    {
        _groupsService = groupsService;
    }

    [HttpGet]
    [Route("")]
    public async Task<IActionResult> IndexAsync(CancellationToken ct)
    {
        var result = await _groupsService.GetAllAsync(ct);
        return View("Index", result.ToViewModel());
    }

    //...
    
}
```

To see this working, we can head to the browser, make a request and before it finishes hit the cancel button. Now if we look at the logs we see the exception that was thrown. Of course some things may not be cancelled, for instance if an operation with side effects is done, maybe it shouldn't be just abruptly stopped, but in the other cases, this pattern works well (even if it's probably a bit annoying to have to be always passing the `CancellationToken` argument).

## Calling it synchronously (but don't)
We should avoid calling async methods synchronously as much as we can, but we know that sometimes, for some weird reason, we just can't. In those cases, we can at least try to make the best out of a bad situation.

The usual way one would get the `Task<T>` result synchronously is by accessing the `Result` property. One thing about this is that if an exception occurs, instead of the expected one we get an `AggregateException`, that wraps any exception thrown in an async method. This doesn't happen when we await the task, because it unwraps the exception. Alternatively to `Result`, we can do `DoSomethingAsync().GetAwaiter().GetResult()`, as this will handle unwrapping the exception. It won't solve the problem that we're blocking the thread waiting for the result and that we might get a deadlock in certain scenarios (for instance in ASP.NET pre-Core), but at least we get the right exception.

## ConfigureAwait(false)
Continuing on the subject (but not exclusively), the reason why blocking on an async method can cause a deadlock (in some scenarios, but not in ASP.NET Core) is that there is a `SynchronizationContext` that's hold on by the blocked thread, and when the async method completes, tries to get a hold of that context but can't, as another thread owns it.

This context is relevant, for instance, in desktop and Xamarin applications, where it's important for the async continuation to run on the UI thread to make some visible change. Because ASP.NET Core doesn't have these kinds of needs, `SynchronizationContext` was removed altogether.

Even knowing that this isn't a problem in ASP.NET Core, if we write a library that we may eventually want to share, we don't know what kind of application will use it. So a common practice when developing such libraries, when the code doesn't rely on this context, is to use `ConfigureAwait(false)` when awaiting on tasks. This means that the code after the `await` doesn't need the context, so no need to try and acquire, meaning no deadlocks for badly behaved clients (and possibly other benefits, like small performance gains, read more about it [here](https://msdn.microsoft.com/en-us/magazine/jj991977.aspx)).

## Returning Task without awaiting
If you worked with async/await in .NET, you've probably came across situations where you have an async method in which the only async thing it does is the last line, and the output of that line is what it will return. Something like:

```csharp
public async Task<bool> CheckSomethingAsync()
{
    return await InnerCheckSomethingAsync();
}

private async Task<bool> InnerCheckSomethingAsync()
{
    //...
}
```

In such scenarios, can't we ignore the async/await part and just return the result of the other method?

```csharp
public Task<bool> CheckSomethingAsync()
{
    return InnerCheckSomethingAsync();
}

//...
```

As in many other situations, it depends. In a scenario like the above, it might be acceptable, but we have to keep in mind that in case an exceptions occurs, the stack trace will be different, as if `CheckSomethingAsync` wasn't even called. Let's take a look at a couple of sample stack traces.

With async/await:
```
   at Program.<InnerCheckSomethingAsync>d__3.MoveNext() in d:\Windows\Temp\kw03shrz.0.cs:line 28
--- End of stack trace from previous location where exception was thrown ---
   at System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
   at System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
   at System.Runtime.CompilerServices.TaskAwaiter`1.GetResult()
   at Program.<CheckSomethingAsync>d__0.MoveNext() in d:\Windows\Temp\kw03shrz.0.cs:line 22
--- End of stack trace from previous location where exception was thrown ---
   at System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
   at System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
   at System.Runtime.CompilerServices.TaskAwaiter`1.GetResult()
   at Program.Main() in d:\Windows\Temp\kw03shrz.0.cs:line 12
```

Returning the task immediately:
```
   at Program.<InnerCheckSomethingAsync>d__0.MoveNext() in d:\Windows\Temp\uh2ndf5e.0.cs:line 28
--- End of stack trace from previous location where exception was thrown ---
   at System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
   at System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
   at System.Runtime.CompilerServices.TaskAwaiter`1.GetResult()
   at Program.Main() in d:\Windows\Temp\uh2ndf5e.0.cs:line 12
```

It depends on the situation whether this is acceptable or not. By returning the task immediately, we lose a bit of information, but gain a bit of performance, because we're bypassing the creation of an async state machine in `CheckSomethingAsync`. Depending on the situation, one might be more important than the other.

Besides this, there are at least a couple other things to take care. If the call to `InnerCheckSomethingAsync` was actually inside a try/catch or using block (and maybe others I don't remember right now), we couldn't do the immediate return, as some code could run after the blocks were no longer in scope.

An example:
```csharp
public static Task<bool> CheckSomethingAsync()
{
    try 
    {
        return InnerCheckSomethingAsync();
    } 
    catch(Exception ex)
    {
        //act on exception
    }
}

private static async Task<bool> InnerCheckSomethingAsync()
{
    await Task.Delay(1000);
    throw new Exception("Sample error");
}
```
In this case, the exception wouldn't be caught, as `CheckSomethingAsync` would've already returned the `InnerCheckSomethingAsync` task to its caller.

## Making requests in parallel
### How to
The nature of async, more specificaly the usage of async when making requests makes it easier to parallelize them, without needing to mess up with threads (explicitly at least). 

Let's take as an example a method `CallExternalServiceAsync`. Imagining we want to make a couple of unrelated requests to it, instead of awaiting them both immediatly, we can invoke the method and store the returned `Task`s in variables, and awaiting them later, allowing for them to go in parallel. If we await them immediately, the requests will be serialized, not parallel. Sample code:

`InMemoryGroupsService.cs`
```csharp
public class InMemoryGroupsService : IGroupsService
{
    private static readonly Random RandomGenerator = new Random();
    
    //...

    public async Task<Group> GetByIdAsync(long id, CancellationToken ct)
    {
        var extResult1Task = CallExternalServiceAsync(ct);
        var extResult2Task = CallExternalServiceAsync(ct);

        var result1 = await extResult1Task;
        var result2 = await extResult2Task;
        
        return _groups.SingleOrDefault(g => g.Id == id);
    }

    //...

    private async Task<int> CallExternalServiceAsync(int multiplier, CancellationToken ct)
    {
        await Task.Delay(1000);
        return RandomGenerator.Next();
    }
}
```

By doing the "requests" this way, instead of the code taking about 2 seconds to run, as it would be the case if we did `await CallExternalServiceAsync(ct)`, serializing the code execution, it takes about 1 second, because we're starting both asynchronous operations as soon as the synchronous part of the code allows us.

Be aware that we can only do this if the target code allows us, for instance, if using the same service class instance is thread safe. A couple of examples:
- When using an `HttpClient` to make requests, we can do this without problems, as long as we only use the thread safe methods (at the time of writing: `DeleteAsync`, `GetAsync`, `GetByteArrayAsync`, `GetStreamAsync`, `GetStringAsync`, `PostAsync`, `PutAsync` and `SendAsync` - see [here](https://docs.microsoft.com/en-us/dotnet/api/system.net.http.httpclient?view=netcore-2.2))
- When using Entity Framework Core, the `DbContext` is not thread safe, so invoking operations in parallel on the same instance is not a good idea

Another thing to remember, when invoking another service,is to be careful not to abuse it, as we could basically cause a denial of service without meaning to (although there are some protections in place to avoid too many requests to the same host, read more [here](https://docs.microsoft.com/en-us/dotnet/api/system.net.servicepointmanager?view=netcore-2.2)).

### Waiting for all the requests to complete

Regarding waiting for the tasks completion, the approach above is the simplest one, but we have other choices, like `Task.WhenAll` and `Task.WhenAny`. 

`Task.WhenAll` will do basically the same as the code above, but can get a collection of `Task`s to await, instead of having to `await` each one individually. It can also be useful if for instance we wanted to group the returned tasks in a collection. Just by adapting the above example, we get:

`InMemoryGroupsService.cs`
```csharp
public class InMemoryGroupsService : IGroupsService
{
    //...

    public async Task<Group> GetByIdAsync(long id, CancellationToken ct)
    {
        await Task.Delay(1000, ct);
        var extResult1Task = CallExternalServiceAsync(1, ct);
        var extResult2Task = CallExternalServiceAsync(2, ct);

        await Task.WhenAll(extResult1Task, extResult2Task);
        
        return _groups.SingleOrDefault(g => g.Id == id);
    }

    //...
}
```

Another option, although with a different goal, is `Task.WhenAny`. Like the name suggests, it will wait for the tasks, like `WhenAll`, but in this case, as soon as one of them completes, the code execution proceeds.

```csharp
public class InMemoryGroupsService : IGroupsService
{
    //...

    public async Task<Group> GetByIdAsync(long id, CancellationToken ct)
    {
        var extResult1Task = CallExternalServiceAsync(1, ct);
        var extResult2Task = CallExternalServiceAsync(2, ct);

        await Task.WhenAny(extResult1Task, extResult2Task);
            
        return _groups.SingleOrDefault(g => g.Id == id);
    }
    
    //...
    
    private async Task<int> CallExternalServiceAsync(int multiplier, CancellationToken ct)
    {
        await Task.Delay(1000 * multiplier);
        return RandomGenerator.Next();
    }
}
```

In this example, `Task.WhenAll` would wait for about 2 seconds, as it's the duration of the longest running call. `Task.WhenAny` on the other hand, would take about 1 second, because it's the duration of the quickest call.

## TaskCompletionSource
Although I'm leaving the "basics" realm of async/await, I think the final couple of subjects we're going to talk about are interesting to know about, even if not needed on a regular basis (depending on what we're developing of course).

Starting with `TaskCompletionSource`, it allows us to create a way to fulfill a `Task` "manually", as we're used to simply await on them, provided by some already existing implementation, like an `HttpClient` or a `DbContext`. To show how this work, let's create a sort of queue with a controller.

### Creating a TaskCompletionSource and setting the result

`DemoNotRecommendedQueueController.cs`
```csharp
[Route("queue")]
public class DemoNotRecommendedQueueController : Controller
{
    private static readonly ConcurrentQueue<TaskCompletionSource<int>> TaskCompletionSourceQueue 
        = new ConcurrentQueue<TaskCompletionSource<int>>();

    [Route("ask")]
    public async Task<IActionResult> AskAsync()
    {
        var tcs = new TaskCompletionSource<int>();
        TaskCompletionSourceQueue.Enqueue(tcs);
        var result = await tcs.Task;
        return Content(result.ToString());
    }

    [Route("tell/{value}")]
    public IActionResult Tell(int value)
    {
        if (TaskCompletionSourceQueue.TryDequeue(out var tcs))
        {
            if (!tcs.TrySetResult(value))
            {
                return StatusCode(500);        
            }
                
            return NoContent();
        }

        return NotFound();
    }
```

So, we're using a `ConcurrentQueue` to store `TaskCompletionSource<int>` instances. When a request arrives at `/queue/ask`, a `TaskCompletionSource<int>` is created and stored in the queue, then we can `await` on the provided `Task`. Then we can make a request, `/queue/tell/1`, which will try to set the result of the first `TaskCompletionSource` in the queue, effectively causing `AskAsync` action to resume execution.

Something important to keep in mind, and David Fowler [tweeted](https://twitter.com/davidfowl/status/1026736063836876800) alerting for the fact, is that when we set the result on `TaskCompletionSource`, by default the continuation of the awaiting `Task` executes immediately on the thread that sets it, and this can bring problems if we're not aware of it. A good rule of thumb in these cases is to make the continuation run on another thread, by specifying `TaskCreationOptions.RunContinuationsAsynchronously` when creating the `TaskCompletionSource`.

`DemoNotRecommendedQueueController.cs`
```csharp
[Route("ask")]
public async Task<IActionResult> AskAsync()
{
    var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
    //...
}
```

An interesting article for the types of issues we should be aware when using `TaskCompletionSource` is Sergey Teplyakov's ["The danger of TaskCompletionSource&lt;T&gt; class"](https://blogs.msdn.microsoft.com/seteplia/2018/10/01/the-danger-of-taskcompletionsourcet-class/).

### Making the task fail
Of course we may want not to set the result of the task, but instead cancel it or throw an exception, and we can do that. Besides `SetResult` (and `TrySetResult`), we have `SetCanceled` (and `TrySetCanceled`) and `SetException` (along with `TrySetException`). Let's see an example of `TrySetCanceled`.

`DemoNotRecommendedQueueController.cs`
```csharp
[Route("queue")]
public class DemoNotRecommendedQueueController : Controller
{
    //...

    [Route("cancel")]
    public IActionResult Cancel()
    {
        if (TaskCompletionSourceQueue.TryDequeue(out var tcs))
        {
            if (!tcs.TrySetCanceled())
            {
                return StatusCode(500);        
            }
                    
            return NoContent();
        }

        return NotFound();
    }
}
```

This will have the same behavior as we've seen earlier on `CancellationToken`s, ending up with a `TaskCanceledException`.

## CancellationTokenSource
Speaking about `CancellationToken`, like for `TaskCompletionSource`, we can also have a `CancellationTokenSource`, so we can signal the cancellation of an operation (like MVC does with the `CancellationToken` injected in the actions).

Continuing with an example similar to the above, we have:

```csharp
[Route("queue")]
public class DemoNotRecommendedQueueController : Controller
{
    //...

    private static readonly ConcurrentQueue<CancellationTokenSource> CancellationTokenSourceQueue =
        new ConcurrentQueue<CancellationTokenSource>();

    //...

    [Route("delay/{value}")]
    public async Task<IActionResult> DelayAsync(int value)
    {
        using (var cts = new CancellationTokenSource())
        {
            CancellationTokenSourceQueue.Enqueue(cts);
            await Task.Delay(value, cts.Token);
            CancellationTokenSourceQueue.TryDequeue(out _);
            return Content("Done waiting");
        }
    }

    [Route("delay/cancel")]
    public IActionResult CancelDelay()
    {
        if (CancellationTokenSourceQueue.TryDequeue(out var cts))
        {
            cts.Cancel();
            return Content("Delay cancelled!");
        }

        return NotFound();
    }
}
```

The code is pretty similar to the previous example (ignore that I'm not using the `ConcurrentQueue` correctly and there are concurrency issues because of that 😛), but instead of awaiting on the `TaskCompletionSource`'s task, we're awaiting on something that can be cancelled by the created `CancellationTokenSource`. So to test, we can go to the browser and invoke `/queue/delay/10000`, and before the time expires, invoke `/queue/delay/cancel`, causing the `DelayAsync` action to end abruptly, with the usual `TaskCanceledException`.

## Outro
There's a lot more to async and await, and concurrent programming in .NET, but I think I covered some of the most regularly used parts of it. I'll leave an extra couple of resources if you want to dig deeper:
- Stephen Cleary's blog - [https://blog.stephencleary.com/](https://blog.stephencleary.com/) - which has lots of .NET concurrency related content, and I used a lot as reference for this episode
  - ["Task.Run Etiquette Examples: Don't Use Task.Run in the Implementation"](https://blog.stephencleary.com/2013/11/taskrun-etiquette-examples-dont-use.html)
- Stephen Cleary MSDN post on async/await best practices - ["Async/Await - Best Practices in Asynchronous Programming"](https://msdn.microsoft.com/en-us/magazine/jj991977.aspx) - which is most likely more thorough then mine (although from before ASP.NET Core was a thing)
- David Fowler's [async guidance article](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md), with a lot of examples of common scenarios in ASP.NET Core
- Sergey Teplyakov's ["The danger of TaskCompletionSource&lt;T&gt; class"](https://blogs.msdn.microsoft.com/seteplia/2018/10/01/the-danger-of-taskcompletionsourcet-class/)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement/tree/episode010).

Please send any feedback so I can improve and adjust the next episodes.

Thanks for stopping by, cyaz!