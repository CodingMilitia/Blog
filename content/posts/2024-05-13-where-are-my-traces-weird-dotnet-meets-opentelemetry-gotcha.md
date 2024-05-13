---
author: João Antunes
date: 2024-05-13 08:55:00+01:00
layout: post
title: Where are my traces? (weird .NET meets OpenTelemetry gotcha)
summary: Quick post about a weird gotcha I found when using distributed tracing with OpenTelemetry .NET, in a system where not all services have otel setup
images:
  - /images/2024/05/13/where-are-my-traces-weird-dotnet-meets-opentelemetry-gotcha.png
categories:
  - csharp
  - dotnet
tags:
- opentelemetry
- observability
- logs
- traces
slug: where-are-my-traces-weird-dotnet-meets-opentelemetry-gotcha
---

## Intro

This will be a brief post about a weird gotcha I found when using distributed tracing with OpenTelemetry and .NET, in a system where not all components have OpenTelemetry tracing setup (or only do so partially), causing traces to not be recorded by downstream services.

I wasted a few hours to figure out what was going on, so documenting it to remember later (and hopefully help someone running into the same issue).

## How the issue manifests

We can reproduce the issue rather simply ([sample code](https://github.com/joaofbantunes/OpenTelemetryDotnetTracePropagationGotchaSample)):

- Create a couple of ASP.NET Core APIs, one to act as a server another as the client
- In the client API, don't configure tracing with OpenTelemetry, just add an endpoint that makes an HTTP request to the server API
- In the server API, configure OpenTelemetry tracing and exporting

If we `curl` the server API directly and check if the trace was exported to our chosen backend, we'll see it there, as expected. However, if we `curl` the client API, even though it invokes the server API behind the scenes, no trace shows up. What gives?

## Why does it happen?

From what I could figure out after poking around (plus a couple of ideas from folks on [Mastodon](https://mastodon.social/@joaofbantunes/112384469266644189), including a brief exchange with [Martin Thwaites](https://martinjt.me)), it seems that even when we're not using OpenTelemetry, ASP.NET is still using activities behind the scenes, as well as propagating the [`traceparent`](https://www.w3.org/TR/trace-context/#traceparent-header) header, to use the id for log correlation.

What causes the downstream service not record the trace, is that in these cases where the activity was created by a service that doesn't have tracing configured, `traceparent` has the [`sampled` flag](https://www.w3.org/TR/trace-context/#sampled-flag) set to false. As by default .NET services with OpenTelemetry tracing are configured to use the `ParentBasedSampler` to decide which traces to record, when it sees the `sampled` flag set to false, it discards the trace ([sampling](https://opentelemetry.io/docs/concepts/sampling/) is an important aspect of otel and distributed tracing, as we can't sample every trace, as it would be prohibitively costly, as well as unnecessary).

I kept the explanation high-level, but hope it was enough to understand the issue. Maybe [Martin](https://martinjt.me) writes a more complete post at some point, with his much more in depth perspective.

## A slight but interesting (and annoying) variation

I started by explaining the issue in the way that it seemed easier to grasp, however, that wasn't my initial encounter with it. My first encounter, was a bit more confusing 😅.

So, I had a similar scenario, with a client API and a server API, in this case using gRPC. The client API had OpenTelemetry configured, but not gRPC instrumentation. I was making gRPC calls in a `IHostedService`, at startup. I hadn't instrumented that hosted service, so there was no active activity in that context.

In this scenario, without an activity already present and without gRPC instrumentation enabled, the same behavior described earlier can be witnessed: somewhere in the gRPC client code, some activity is being created, the `sampled` flag is set to false, and the request isn't sampled by the server app.

What made it even more confusing to me, is that while troubleshooting, I added a couple of traditional HTTP calls to the same hosted service, and those were being sampled by the server. Even if I disabled HTTP client instrumentation, or OpenTelemetry completely, the calls were still sampled, with the difference that with instrumentation enabled the `traceparent` header was included, while with things disabled it was not. So it seems the HTTP client itself is not creating an activity behind the scenes when there are no listeners nor parent activities, while the gRPC client apparently does.

## So what's the problem and what can be done about it

By now the problem is hopefully clear, but just in case it isn't: with this behavior, a single .NET service with distributed tracing not configured (or partially configured) can mess up the tracing behavior of a larger system, depending on how it interacts with the remaining components.

I imagine this can be particularly annoying in situations where we have large systems, composed of a lot of services, and we want to gradually introduce OpenTelemetry, without touching all the services at the same time (I'm actually going to face this at work).

Regarding what can be done about it... not completely sure 🤷‍♂️. One thing is certain, `ParentBasedSampler` isn't really useful in this scenario.

If you don't mind touching all the .NET services you have, disabling logging for the category `Microsoft.AspNetCore.Hosting.Diagnostics` seems to avoid the issue, according to [Martin](https://hachyderm.io/@Martindotnet/112390947243593723) (and we can go back to `ParentBasedSampler` if we want to).

If touching potentially dozens of services (or more, depending on how applications are designed) isn't an option, I guess another possibility is using a different sampler, for example the `AlwaysOnSampler`, and fully delegating sampling to the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) (not sure this is ideal for very high load systems though, never tried it 😅). This is assuming we control all the components we're enabling distributed tracing on though. If we have some third party provided component in our infra, this option goes out the window and we're back to needing to fiddle with the .NET services.

Gotta study the topic a bit more to try and figure out additional work around options. Let me know if you have any other ideas on how to work around this issue.

## Outro

That's it for this quick post. As mentioned, this post is more of a reminder for myself, but if someone faces the same issue, hopefully can find this post before wasting as many hours on it as I did.

TLDR: be careful when introducing OpenTelemetry and distributed tracing into a system with .NET services in the mix, because in the right circumstances, they can mess up things for all the others.

I hope the .NET team can revisit this behavior and fix it at some point, though I'm not sure I'm very confident that'll happen, given it seems to be have been a deliberate design decision.

So far, from what I experimented with regarding .NET and OpenTelemetry integration, I've been happy with the support, but this issue certainly has the potential to cause a bunch of hardcore WTF moments for those that aren't aware of it, plus some headaches to work around it.

Relevant links:

- [Sample code](https://github.com/joaofbantunes/OpenTelemetryDotnetTracePropagationGotchaSample)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [OpenTelemetry Docs - Sampling](https://opentelemetry.io/docs/concepts/sampling/)

Thanks for stopping by, cyaz! 👋
