---
author: João Antunes
date: 2018-04-21 19:00:00+01:00
layout: post
title: 'Using cancellation tokens on ASP.NET Core MVC actions'
summary: 'Cancellation tokens are really useful when working with async stuff, and this is the simplest way of getting some nice free benefits in ASP.NET applications using them.'
images:
- '/images/2018/04/21/2018-04-21-aspnet-ct-cover.jpg'
categories:
- dotnet
- video
tags:
- dotnet
- aspnet
- tasks
- cancellationtoken
slug: using-cancellation-tokens-on-aspnetcore-mvc-actions
---

{{< embedded-image "/images/2018/04/21/2018-04-21-aspnet-ct-cover.jpg" "Using cancellation tokens on ASP.NET Core MVC actions" >}}

Today I'm back for a smaller post, on a topic that has already some amount of posts about, but I think is not spread enough and it's not being used as much as it could, and probably should - using cancellation tokens with ASP.NET Core MVC actions ([video walk-through at the end of the post](#video-walk-through)).

If you want to follow along with the code, it's [here](https://github.com/joaofbantunes/AspNetCoreMvcActionCancellationTokenSample).

## A scenario
Imagine we create a Web API to power our frontend. We have a search box with autocomplete, have the usual debounce logic to avoid a request on every keystroke, but even with this logic in place, sometimes it takes longer for the user to complete the train of thought. When this happens, the client side application makes a new request and disregards the previous ones. On the server side, if we're not expecting this to happen, the application will just complete the requested operation and issue a response as if nothing happened, having completed a workload for nothing, misusing the server's resources.

## A solution
Fortunately, there's a stupid simple solution for this problem. On our controller's action methods, we can receive a `CancellationToken` instance and pass it around our asynchronous methods. When the client cancels a request, this token will be signaled and our methods are notified and can act accordingly.

## Some samples
For the samples, I'm making something simpler then the autocomplete scenario I talked above, but I think it's more than enough to take the point across.

```csharp
[Route("thing")]
public async Task<IActionResult> GetAThingAsync(CancellationToken ct)
{
    try
    {
        await _httpClient.GetAsync("http://httpstat.us/204?sleep=5000", ct);
        _logger.LogInformation("Task completed!");
    }
    catch (TaskCanceledException)
    {
        _logger.LogInformation("Task canceled!");
        
    }
    return NoContent();
}
```

So here we have a stupid simple action, that has no client supplied arguments, only a `CancellationToken`. This `CancellationToken` is injected by the framework and will be signaled for cancellation by the framework. A case in which it is signaled - when the client cancels the request. In the demo code I'm just invoking an external endpoint that will take 5 seconds to complete, passing the token to the `GetAsync` method. I'm wrapping it all up in try catch and expecting a `TaskCanceledException`, as it's the one thrown when the operation is canceled due to cancellation token being signaled.

Below I added a gif with a quick show of a request being canceled.

{{< embedded-image "/images/2018/04/21/2018-04-21-aspnet-ct-demo.gif" "Cancellation tokens in action" >}}

Here's another quick sample:
```csharp
[Route("anotherthing")]
public async Task<IActionResult> GetAnotherThingAsync(CancellationToken ct)
{
    try
    {
        for(var i = 0; i < 5; ++i)
        {
            ct.ThrowIfCancellationRequested();
            //do stuff...
            await Task.Delay(1000, ct);
        }
        _logger.LogInformation("Process completed!");
    }
    catch (Exception ex) when (ex is TaskCanceledException || ex is OperationCanceledException)
    {
        _logger.LogInformation("Process canceled!");
        
    }
    return NoContent();
}
```

In this case, besides passing the `CancellationToken` along to other asynchronous methods - in this case a `Task.Delay` but the result of the cancellation is similar to the `HttpClient.GetAsync` - I'm checking the token to see if it was signaled for cancellation. So, imagine the `Task.Delay` method didn't accept a `CancellationToken` as an argument, on the next loop iteration the code would check for a cancellation request and would throw an exception (in this case an `OperationCanceledException`).

This may be useful in scenarios we're not doing async work that nevertheless takes a while to complete and would benefit from being cancellable along the way.

## Video walk-through <a id="video-walk-through" class="no-anchor-icon"></a>
If you're more in a mood for video walk-through rather the reading, be my guest :)
{{< yt KLAsOGyv9ZE >}}
<br/>
Hope this was useful.
Thanks for reading/watching, cyaz!