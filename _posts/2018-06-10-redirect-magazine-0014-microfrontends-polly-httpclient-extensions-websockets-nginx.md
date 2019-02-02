---
author: João Antunes
comments: true
date: 2018-06-10 10:00:00+01:00
layout: post
title: '[Redirect Magazine] #14 - Micro Frontends, Polly extensions for HttpClient, WebSockets and Nginx'
categories:
- redirect magazine
tags:
- dotnet
- microfrontends
- polly
- httpclient
- websockets
- aspnetcore
- nginx
---

Have at it!

# Articles
## ["Micro Frontends"](https://micro-frontends.org/)
On the same page as a video I shared on Redirect Magazine #12, this article talks about taking the microservices architecture into the frontend, so we don't end up with a fancy microservice backend coupled with a monolithic frontend, kind of defeating the purpose of the architectural decision in the first place.
<br/>
## ["HttpClientFactory in ASP.NET Core 2.1 (Part 4) - Integrating with Polly for transient fault handling"](https://www.stevejgordon.co.uk/httpclientfactory-using-polly-for-transient-fault-handling)
I've made some posts on using Polly to improve .NET applications resilience in the past, so it's nice that it now as some helpers to be integrated with `HttpClient` in ASP.NET Core.
<br/>
## ["Things I Wish Someone Told Me About ASP.NET Core WebSockets"](https://www.codetinkerer.com/2018/06/05/aspnet-core-websockets.html)
Nice write-up on some things to be aware of when using WebSockets with ASP.NET Core (not using SignalR in this article).
<br/>
# Videos
## ["Nginx for .NET Developers - Ian Cooper"](https://youtu.be/Z2dE7OpL0Fc)
Interesting talk introducing using Nginx with ASP.NET Core. It has some slightly outdated bits, as Kestrel has seen some changes with the release of ASP.NET Core 2.1 (like having replaced libuv by default), but it's still a valid and interesting talk.

{% youtube "https://youtu.be/Z2dE7OpL0Fc" %}

<br/>
Cyaz!