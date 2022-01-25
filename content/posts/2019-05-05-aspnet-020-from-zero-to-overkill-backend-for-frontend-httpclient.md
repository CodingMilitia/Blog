---
author: João Antunes
date: 2019-05-05 09:30:00+01:00
layout: post
title: "Episode 020 - The backend for frontend and the HttpClient - ASP.NET Core: From 0 to overkill"
summary: "In this episode, we'll start building our backend for frontend, which will bridge the interaction between the Vue.js frontend application and the backend APIs we'll develop. For now this will be mostly an excuse for playing with the HttpClient in ASP.NET Core, as we'll improve the BFF in the future, with some proxying capabilities for requests that don't need additional logic."
images:
- '/images/2019/05/05/aspnet-core-from-zero-to-overkill-e020.jpg'
categories:
- fromzerotooverkill
- dotnet
tags:
- dotnet
- aspnetcore
slug: aspnet-020-from-zero-to-overkill-backend-for-frontend-httpclient
---

In this episode, we'll start building our backend for frontend, which will bridge the interaction between the Vue.js frontend application and the backend APIs we'll develop. For now this will be mostly an excuse for playing with the `HttpClient` in ASP.NET Core, as we'll improve the BFF in the future, with some proxying capabilities for requests that don't need additional logic.

For the walk-through you can check out the next video, but if you prefer a quick read, skip to the written synthesis.

{{< yt A8ZCVzeqFtA >}}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />

## Intro
In this episode, we'll start building our backend for frontend, which will bridge the interaction between the Vue.js frontend application and the backend APIs we'll develop.

A backend for frontend (to which we'll refer to as BFF) isn't a required piece, but can be a useful one when we're working with an application which is composed by multiple services, as is our goal with the PlayBall application.

The main goal of having a BFF is to create a simpler interface between a frontend and the backend. During this project, the Vue.js application will mostly likely be the only frontend we'll develop, but imagining that we would have more options (Android, iPhone, smart watches, TVs, ...), a BFF would allow us to create a tailored API for the specific frontend, simplifying the work in the client's device, having it done instead in the server, closer to the supporting services.

In this episode, we'll basically just create the project and use it as an excuse to play around with the `HttpClient` in ASP.NET Core, to see some relevant usage patterns that emerged recently. In the future we'll improve the BFF, so simple calls to the services can be proxied automatically, allowing us to only write code when we want to tailor the API to the frontend.

To improve the BFF in the future, we'll explore [ProxyKit](https://github.com/damianh/ProxyKit) and [Ocelot](https://github.com/ThreeMammals/Ocelot) (or other projects that show up until then), but that's a subject for another time 🙂.

## New project
As the backend for frontend is specific to a... frontend (🙃), we'll put it next to the Vue.js application, in the same [GitHub repository](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend).

It will be an ASP.NET Core 2.2 application, like the [GroupManagement](https://github.com/AspNetCoreFromZeroToOverkill/GroupManagement) service we developed in past episodes. In the repo, we'll have the following structure:

```
.
├── client
|   ├── Vue.js application files
|   └── ...
├── server
|   ├── src
|   |   └── CodingMilitia.PlayBall.WebFrontend.BackForFront.Web
|   |       ├── CodingMilitia.PlayBall.WebFrontend.BackForFront.Web.csproj
|   |       ├── BFF application files
|   |       └── ...
|   ├── CodingMilitia.PlayBall.WebFrontend.BackForFront.sln
|   └── .gitignore
├── LICENSE
└── README
```

Before starting the with the main objective of this episode (playing with the `HttpClient`), let's do some initial setup. Beginning with the `Startup` class, we want to get MVC working, as we'll implement the "proxying" logic in controllers. We already saw this in the past, but as a refresher:

`Startup.cs`
```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services
            .AddMvc()
            .SetCompatibilityVersion(CompatibilityVersion.Version_2_2);

            // ...
    }

    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseMvc();
    }
}
```

Regarding the organization of the application, this time instead of the "classic" `Controllers`, `Models` and `Views` folders, we'll create a `Features` folder, and split things by feature in there. Right now the `GroupManagement` service only has groups logic, so the BFF will only have a `Groups` folder, with a `GroupsController` and a `GroupModel`. We'll see the controller in a bit, but regarding the `GroupModel`, it's an exact copy of the one in the group management service.

`Features\Groups\GroupModel.cs`
```csharp
public class GroupModel
{
    public long Id { get; set; }
    public string Name { get; set; }
    public string RowVersion { get; set; }
}
```

## Implementing the requests
Let's take a look at our `GroupsController` implementation to see how we fullfil the requests. It's really boiler plate code, that's why we'll try to get rid of it in a later episode, but I think it's worth the quick look.

Just before diving into the code, a couple of notes:
- We'll get the `HttpClient` as a parameter in the constructor, we'll see how to configure it in a moment. The `HttpClient` comes with the base address of the group management service pre-configured.
- We won't worry about handling errors right now, but we definitely have to do it in the future.

Let's start with the action to get all groups.

`Features\Groups\GroupsController.cs`
```csharp
[Route("groups")]
public class GroupsController : ControllerBase
{
    private const string BaseAddress = "/groups";
    private readonly HttpClient _httpClient;

    public GroupsController(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    [HttpGet]
    [Route("")]
    public async Task<ActionResult<IReadOnlyCollection<GroupModel>>> GetAllAsync(CancellationToken ct)
    {
        var response = await _httpClient.GetAsync(BaseAddress, ct);
        var result = await response.Content.ReadAsAsync<IReadOnlyCollection<GroupModel>>(ct);
        return Ok(result);
    }

    // ...
```

The first thing I'd like to point out, is the return type of the `GetAllAsync` method, as it's something I haven't used yet in this series. Normally when implementing an API action method, we would indicate as the return type the model we'll return, or if we need to return a different status code (like a 500, 404, etc) we would declare it as an `IActionResult`. 

`ActionResult<T>` is an interesting little class that allows us to have the best of both worlds. If we to return a `NotFound()`, it'll work fine, if we want to return a model directly, it'll also work fine. That's because it has implicit operators to make those conversions. The corner case is exactly the one you see above. The implicit operators don't work with interfaces, hence the `return Ok(result)`. If, for example, I had used `GroupModel[]` instead, it would have worked just fine.

Regarding the HTTP request, nothing too strange going on I think. We make a GET request with the base address, as we want to list the available groups, then read the JSON response so we can return it. The models are an exact match, so no need for any mapping logic.

Onwards to the get by id action.

`Features\Groups\GroupsController.cs`
```csharp
// ...
[HttpGet]
[Route("{id}")]
public async Task<ActionResult<GroupModel>> GetByIdAsync(long id, CancellationToken ct)
{
    var response = await _httpClient.GetAsync($"{BaseAddress}/{id}", ct);
    if(response.StatusCode == HttpStatusCode.NotFound)
    {
        return NotFound();
    }
    var result = await response.Content.ReadAsAsync<GroupModel>(ct);
    return result;
}
// ...
```

Again, more of the same. We do some extra url composition before making the request, then we check if we got a 404 so we propagate that status if needed, otherwise we parse the response and return it, as you can see, taking advantage of the implicit conversion of the model.

The rest of the actions are more of the same, so I'll just drop them here for reference.

`Features\Groups\GroupsController.cs`
```csharp
// ...
[HttpPut]
[Route("{id}")]
public async Task<ActionResult<GroupModel>> UpdateAsync(long id, GroupModel model, CancellationToken ct)
{
    var response = await _httpClient.PutAsJsonAsync($"{BaseAddress}/{id}", model, ct);
    if (response.StatusCode == HttpStatusCode.NotFound)
    {
        return NotFound();
    }
    var result = await response.Content.ReadAsAsync<GroupModel>(ct);
    return result;
}

[HttpPut]
[HttpPost]
[Route("")]
public async Task<ActionResult<GroupModel>> AddAsync(GroupModel model, CancellationToken ct)
{
    var response = await _httpClient.PostAsJsonAsync(BaseAddress, model, ct);
    var result = await response.Content.ReadAsAsync<GroupModel>(ct);
    return CreatedAtAction(nameof(GetByIdAsync), new { id = result.Id }, result);
}

[HttpDelete]
[Route("{id}")]
public async Task<IActionResult> RemoveAsync(long id, CancellationToken ct)
{
    await _httpClient.DeleteAsync($"{BaseAddress}/{id}", ct);
    return NoContent();
}
// ...
```

## Using the HttpClient (before ASP.NET Core 2.1)
Before looking at how we'll use the `HttpClient`, let's take a quick aside to talk about how it was normally used prior to ASP.NET Core 2.1.

I've mentioned this in a past [post](https://blog.codingmilitia.com/2016/10/27/redirect-how-you-should-not-use-httpclient), but using the `HttpClient` isn't as straightforward as one would expect in the first place.

The `HttpClient` implements the `IDisposable` interface, so it's easy to assume that the best way to use it is just wrap it in a `using` block, making the required requests in there, having it disposed afterwards. Unfortunately, this isn't the best way, as even when we dispose of it, the socket it used to make the requests isn't let go immediately, it remains open for a while. This may not have an impact on a not too loaded application, but if there is sufficient load, we can start to see port exhaustion due to this behavior (more info [here](https://aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/)).

Knowing this, what's the solution? Well, just create a single `HttpClient` and share it (or a constrained number of instances at least). The methods used to make the requests (`GetAsync`, `PostAsync` and so on) are thread safe, so no problem there, as long as we're careful with the other stuff (e.g. messing with `BaseAddress` and `DefaultRequestHeaders`). It isn't ideal, but knowing the issues, we can go around the problem right?

When it seems we've found a good'ish solution, another problem bites us! If we keep the client for the lifetime of the application, eventual DNS changes are not detected (more info [here](http://byterot.blogspot.com/2016/07/singleton-httpclient-dns.html))! Again, we find another solution by playing around with the `ServicePointManager` and we're good to go (again, go [here](http://byterot.blogspot.com/2016/07/singleton-httpclient-dns.html) for more details).

But these are a lot of weird hoops to go through, and there are probably other issues I don't even know. Fortunately, with ASP.NET Core 2.1, a new set of features emerged to ease this pain, mainly the new `HttpClientFactory`.

## Using the HttpClient
We have the controller ready to make the requests, and we've seen some of the problems we faced with the `HttpClient` in the past, so now it's time to see how to use it in ASP.NET Core >= 2.1.

> In this post I'm not going in a lot of details about the new `HttpClient` and `HttpClientFactory` usage patterns, I'll just use them to implement what we need in our application, but for a more detailed read, Steve Gordon has a great series of posts you can check out, starting with [this one](https://www.stevejgordon.co.uk/introduction-to-httpclientfactory-aspnetcore).

As I briefly mentioned, there is a new type called `HttpClientFactory`, which like the name suggests, provides us with instances of the `HttpClient`. The problems mentioned earlier about the `HttpClient` don't stem from it directly, but from the `HttpClientHandler` it uses internally to actually make the requests. With the new features, this handlers are pooled and used by the `HttpClient`s, so we don't really need to worry about disposing the clients (again, for a much more detailed explanation, check [this post out](https://www.stevejgordon.co.uk/introduction-to-httpclientfactory-aspnetcore)).

We can inject the factory directly into the classes that need to make requests, or we can configure the class as I typed client, getting the `HttpClient` in the constructor immediately without needing the factory. In this case, we'll go with the typed client, but of course there may be cases where the factory is preferred.

In the `Startup` class, we'll add the configuration for the typed client.

`Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services
        .AddHttpClient<GroupsController>((serviceProvider, client) => 
        {
            // TODO: use serviceProvider to fetch the base address from configuration
            client.BaseAddress = new Uri("http://localhost:5002");
        });
    //...
```

Even though this code is only setting the base address, we can configure various aspects of the client. We could also delegate these configurations to the actual class that'll be using, I'd say it's a matter of preference, or maybe in some situations one makes more sense than the other.

This should be it to have the thing running, but it's not, because we aren't injecting the `HttpClient` into a "normal" class, we're injecting it into a controller. Controllers aren't registered into the dependency injection container as services, like the other classes we registered, so if we try to run the application like this and make a request to `http://localhost:5000/groups`, we'll get an error like:

```
An unhandled exception occurred while processing the request.
InvalidOperationException: Unable to resolve service for type 'System.Net.Http.HttpClient' while attempting to activate 'CodingMilitia.PlayBall.WebFrontend.BackForFront.Web.Features.Groups.GroupsController'.
```

To be able to make use of the typed client with the controller, we need to go back to MVC configuration in `ConfigureServices` and add an extra line to register the controllers as services, so all of this can wire up nicely.

`Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services
        .AddMvc()
        .SetCompatibilityVersion(CompatibilityVersion.Version_2_2)
        .AddControllersAsServices();
    //...
```

There are other similar methods, like `AddTagHelpersAsServices`, but we only need controllers right now.

Now we can test again and everything should work as expected.

## Using Polly to add retry logic
The final thing we'll take a quick look in this post is using [Polly](https://github.com/App-vNext/Polly) to have the `HttpClient` retry upon a failed request.

> I have a an older post ([here](https://blog.codingmilitia.com/2016/11/08/simpler-error-handling-in-net-applications-using-polly)) and video ([here](https://youtu.be/wBZmdx-5-Cs)) about Polly in general. In this post we'll just take a quick look at the more recently added integration between Polly and the HttpClient in ASP.NET Core. Polly has a lot more feature than just the retries we'll briefly look at in this post.

To have the Polly `HttpClient` integration, we need to install the `Microsoft.Extensions.Http.Polly` NuGet package. Then with a couple of lines of code added to the `Startup` class, we have everything ready.

`Startup.cs`
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services
        .AddHttpClient<GroupsController>((serviceProvider, client) => {/*...*/})
        .AddPolicyHandler(
            HttpPolicyExtensions
            .HandleTransientHttpError()
            .WaitAndRetryAsync(5, attempt => TimeSpan.FromSeconds(attempt)));
```

`AddPolicyHandler` is an extension method from the NuGet package we installed. It allows us to configure policies to apply to the `HttpClient`. Policies are a Polly concept, and are something in which we can wrap an action execution, adding some additional behavior to it, namely implementing some resilience patterns. Some examples of policies are retries, circuit-breakers or timeouts.

In the above code we're configuring a retry policy for transient HTTP errors - from the docs: in the 500 range (server errors) and 408 (request timeout). We have a bunch of options for configuring retries, like retrying immediately, retrying after some time, always retrying on error, retrying for a number of times, and some more variations of these.

The approach taken here is to wait after each failure before retrying, for a maximum of 5 retries. Each time the request fails, the retry is attempted the number of the retry seconds later, or to be clearer, the first retry will wait for 1 second and the last one will wait for 5 seconds.

To see this in action, we can go to the group management service and mess with the code a bit, adding some random exceptions and stuff 🙂.

## Outro
That does it for this first look at the backend for frontend, or better yet, an excuse to use the `HttpClient` 😛. If you think what we implemented here is a bit silly, well... it probably is, as we have tools to do this kind of proxying for us, without writing loads of boiler plate code. As mentioned, we'll tackle that subject in the future, but for now I thought it would be relevant to point out the new usage patterns of the `HttpClient`, alerting again for the troubles that using it incorrectly may bring.

Also, we could bypass the backend for frontend altogether and leave to the frontend the responsibility of calling all the required services. I think however, that this will give us some more flexibility in the future, particularly for things like authentication and authorization (as we'll see in the next episode) or if we have interactions with services that don't expose an HTTP API (for instance, I'm pretty sure we will play a bit with [gRPC](https://grpc.io/) at some point).

Links in the post:
- [You're using HttpClient wrong and it is destabilizing your software](https://aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/)
- [Singleton HttpClient? Beware of this serious behaviour and how to fix it](http://byterot.blogspot.com/2016/07/singleton-httpclient-dns.html)
- [HttpClientFactory in ASP.NET Core 2.1](https://www.stevejgordon.co.uk/introduction-to-httpclientfactory-aspnetcore)
- [Polly](https://github.com/App-vNext/Polly)
- [Simpler error handling in .NET applications using Polly](https://blog.codingmilitia.com/2016/11/08/simpler-error-handling-in-net-applications-using-polly)
- [ProxyKit](https://github.com/damianh/ProxyKit)
- [Ocelot](https://github.com/ThreeMammals/Ocelot)

The source code for this post is [here](https://github.com/AspNetCoreFromZeroToOverkill/WebFrontend/tree/episode020).

Sharing and feedback always appreciated!

Thanks for stopping by, cyaz!