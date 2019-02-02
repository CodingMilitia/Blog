---
author: João Antunes
date: 2018-04-28 12:00:00+01:00
layout: post
title: "Using .NET's HttpClient without following redirects"
summary: "It's likely not a common scenario, but here's how one can use .NET's HttpClient without following redirects."
image: '/assets/2018/04/28/2018-04-28-httpclient-no-redirect-cover.jpg'
categories:
- dotnet
tags:
- dotnet
- aspnet
- httpclient
- redirect
---

[![Using .NET's HttpClient without following redirects](/assets/2018/04/28/2018-04-28-httpclient-no-redirect-cover.jpg)](/assets/2018/04/228/2018-04-28-httpclient-no-redirect-cover.jpg)

I'm pretty sure (or maybe I'm wrong) that we seldom need to avoid following redirects, but I came across such a need one or two times, so I might as well write about it :)

Accompanying source code [here](https://github.com/joaofbantunes/HttpClientNotFollowingRedirectsSample).

## Scenario

So, for a concrete scenario where I actually needed the `HttpClient` not to follow redirects.

At work we were implementing a new feature that was dependant on a legacy system. This feature consisted in showing some banners (similar to ads) that could be clicked. When a banner is clicked it has to do a bunch of things: 

- Open a modal window to do some stuff
- Ping a tracking url indicating that the banner was clicked
- The tracking url may or may not do a redirect, in which case a new tab should be open

Let's disregard the usability of this... those were the requirements.

So for the above steps, the problematic one was the third one - the legacy system we were integrating with is the one responsible for the banners and tracking urls, so we had to make it work with what we had. The first idea that came to mind was, well, always open a new tab, if it redirects then we're good, if not... we have an open blank tab :)

Blank tabs aren't very cool so we thought we could do better. After some more or less complicated ideas, the new plan was to create an indirection<sup>1</sup> that would make the request to the tracking url without following the redirect - if the response was a 200, we're done, if it was a 302 this middleman would return the target url and the application could open it in a new tab.

## Solution

To do this in .NET we're using as usual an `HttpClient`, but as its default behavior is to follow redirects, a little configuration was required.

{% highlight csharp linenos %}
var handler = new HttpClientHandler()
{
    AllowAutoRedirect = false
};
var httpClient = new HttpClient(handler);

var response = await _httpClient.GetAsync(trackingUrl, ct);

var targetUrl = response.StatusCode == HttpStatusCode.Redirect
        ? response.Headers.Location.OriginalString
        : null;
{% endhighlight %}

And that's it! If you're reading the article just to know the required configuration, this is it - so many words to end with about ten of lines of code... sorry :)

Now for the complete context, we're doing this in a ASP.NET application, and by now one should be aware that we can't go around instantiating http clients everywhere or we'll end up with port exhaustion, so we ended up creating a class to abstract this.

{% highlight csharp linenos %}
//Usings...

namespace CodingMilitia.HttpClientNotFollowingRedirectsSample.Web
{
    public class TrackingHttpClientWrapper
    {
        private HttpClient _httpClient;

        public TrackingHttpClientWrapper()
        {
            var handler = new HttpClientHandler()
            {
                AllowAutoRedirect = false
            };
            _httpClient = new HttpClient(handler);
        }

        public async Task<string> TrackAsync(string trackingUrl, CancellationToken ct)
        {
            var response = await _httpClient.GetAsync(trackingUrl, ct);

            return response.StatusCode == HttpStatusCode.Redirect
                 ? response.Headers.Location.OriginalString
                 : null;
        }

        //IDisposable stuff...
    }
}
{% endhighlight %}

And configured as singleton in the DI container.
{% highlight csharp linenos %}
//...
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();

    //old school version
    services.AddSingleton<TrackingHttpClientWrapper>();
    //not the best place to put this but... it'll suffice for the sample
    ServicePointManager.FindServicePoint(new Uri("http://httpstat.us/")).ConnectionLeaseTimeout = (int)TimeSpan.FromMinutes(1).TotalMilliseconds;
    ServicePointManager.DnsRefreshTimeout = (int)TimeSpan.FromMinutes(1).TotalMilliseconds;
}
//...
{% endhighlight %}

Also doing `ServicePointManager` configurations to make sure DNS refreshes as seen [here](https://github.com/dotnet/corefx/issues/11224).

Finally the controller action could simply do something like the following.

{% highlight csharp linenos %}
[Route("track")]
public async Task<IActionResult> TrackAsync(string trackingUrl, CancellationToken ct)
{
    var targetUrl = await _tracker.TrackAsync(trackingUrl, ct);
    if (string.IsNullOrWhiteSpace(targetUrl))
    {
        _logger.LogInformation("No target url -> it wasn't a redirect");
    }
    else
    {
        _logger.LogInformation("Target url: \"{targetUrl}\" -> it was a redirect", targetUrl);
    }
    return Ok(new {targetUrl});
}
{% endhighlight %}


## Bonus round: same solution ASP.NET Core 2.1 version

With ASP.NET Core 2.1 there is some new stuff to work with `HttpClient` and avoid all these shenanigans because of port exhaustion and DNS refreshes, so I added to the sample a V2 solution using the new `HttpClientFactory` features.

{% highlight csharp linenos %}
//Usings...

namespace CodingMilitia.HttpClientNotFollowingRedirectsSample.Web
{
    public class TrackingHttpClientWrapperV2
    {
        private HttpClient _httpClient;

        public TrackingHttpClientWrapperV2(HttpClient httpClient)
        {
            _httpClient = httpClient;
        }

        public async Task<string> TrackAsync(string trackingUrl, CancellationToken ct)
        {
            var response = await _httpClient.GetAsync(trackingUrl, ct);

            return response.StatusCode == HttpStatusCode.Redirect
                 ? response.Headers.Location.OriginalString
                 : null;
        }
    }
}
{% endhighlight %}

And to register with the DI container:

{% highlight csharp linenos %}
//...
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();

    //ASP.NET Core 2.1 version
    services.AddScoped<TrackingHttpClientWrapperV2>();
    services.AddHttpClient<TrackingHttpClientWrapperV2>()
        .ConfigurePrimaryHttpMessageHandler(() =>
        {
            return new HttpClientHandler
            {
                AllowAutoRedirect = false
            };
        });
}
//...
{% endhighlight %}

## Other reading material

For more info on ASP.NET Core 2.1 `HttpClient` related features, Steve Gordon has a series of posts, the first one [here](https://www.stevejgordon.co.uk/introduction-to-httpclientfactory-aspnetcore).

## Wrapping up

Ok, this post is probably bigger than it needed to, just to tell how to configure an http client not to follow redirects, but as I had a real world scenario for its usefulness, I thought I might as well share it.

On a side note, the `HttpClientHandler` has some other options besides the `AllowAutoRedirect` I used, so if you're needing to do something that `HttpClient` doesn't appear to do directly, you may want to take a look at what the `HttpClientHandler` provides.

Thanks for reading!

---

1 - [Fundamental theorem of software engineering](https://en.wikipedia.org/wiki/Fundamental_theorem_of_software_engineering)