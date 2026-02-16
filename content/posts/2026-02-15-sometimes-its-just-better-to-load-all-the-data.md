---
author: João Antunes
date: 2026-02-15 17:00:00+00:00
layout: post
title: Sometimes it's just better to load "all" the data
summary: A quick post on adapting your approach to the problem at hand, not just following the same ideas indiscriminately.
images:
  - /images/2026/02/15/sometimes-its-just-better-to-load-all-the-data.png
categories:
  - csharp
  - dotnet
tags:
  - smalltalk
  - performance
  - batch processing
  - databases
slug: sometimes-its-just-better-to-load-all-the-data
---

## Intro

Time for a quick post on something that is probably obvious for many folks, but clearly not for many others, so worth discussing: sometimes you're better off (performance wise) bulk loading "all" the data into memory to work with it, instead of trying to fetch one thing at a time, as its need arises.

Note that this is highly context dependent, including what "all" means, but we'll get there during the post 🙂.

There won't be code in this post, but we're certainly gonna talk about it.

## Practical example

Imagine you have a batch process that handles a few hundred thousand items. These items already exist in the database, so we need to check their current data, apply some logic, then update the database accordingly.

Following the most obvious approach, we'd probably do a `foreach` through the batch items, querying the database for the corresponding data, applying the logic, finally persisting the changes.

While obvious, and one by one doesn't seem that slow, once you look at it closely, it's actually terrible. Let's say that handling each item like this takes 100ms. If we've got a batch of 100k, we're looking at a bit under 3 hours. Include more complex logic that adds more I/O to the mix, or simply increase the batch size, and we're looking at several hours for something that should be trivially handled.

## That's an actual real world example

So, this example above, it's actually a real world one (oversimplified for the post). A couple of years ago we had such a scenario at work, where we needed to process a text file containing a batch of a few hundreds of thousands of items. At times, it took close to one day to run 🙃.

So, how did we solve it, with not a lot of effort?

Let me preface this by reiterating that it's very context dependent, so keep that in mind, don't think this applies for everything.

We had several thousand rows being created daily, so when earlier I said loading "all" the data, the quotes weren't an accident. It's not effectively all the data, but it is much more data than would actually be needed. Given it's not easy (depending on the scenario) to know exactly what is needed, we used some heuristics to decide what to load.

In this case, we were lucky that the batch file included a date, so we got those dates, grouped by days, then instead of iterating per item, we iterated per day, loading into memory all relevant data for each day. This means that instead of doing hundreds of thousands of database accesses, we did a couple per day we needed to process.

After these changes, which weren't super complex, the close to 24 hours execution time dropped to around 15 minutes.  I'm certain we could do better, but there's always more work to do and that improvement was enough for us to be happy (from a product perspective, from a techie perspective, I'd love if we could drop it to seconds 😅).

## Generalizing

The post as a whole is mostly focused on a specific example of batch processing a large-ish amount of items (it's not that large, but considering the more typical web development flows, we can kind of say it's large). However, this can easily be generalized to different use cases.

Chatty I/O is one of the easiest ways to kill an application's performance, and batching, as we saw earlier, is typically a great way to combat it.

First things first, even though this post focused on batch processing, the ideas still apply, in a different way, to something like handling an HTTP request, where, most of the times, you want to limit the amount of round trips to the database. That's one of the reasons which makes having ORMs configured with lazy loading terrible: some queries may happen behind the scenes, without the developer noticing. Later down the line, in production, people start asking why is an endpoint slow, to eventually figure out that there are 50 queries occurring just to handle a single HTTP request.

Besides database access, other potential use cases may be:

- Messaging
- Invoking HTTP APIs
- File I/O

## Some quick fire bits

Before concluding, wanted to drop some quick bits, that I believe are relevant to the topic. Not going to detail much though.

### Parallelizing work

Considering a problem like the one described earlier, parallelizing work would probably be one of the first ideas folks would go with.

While parallelizing work would indeed reduce the time it would take the job to run, it wouldn't be as impactful as batching, for two main reasons:

- We're still making the same amount of unneeded roundtrips, so we're not really making things more efficient, we're just using brute force to complete the job faster
- As we're massively hammering the database server, we're potentially impacting other flows using the same database, eventually requiring more powerful hardware or more complex database setups to handle the load

### Move logic into the database

Another potential strategy to try and improve the job performance, would be to move logic into the database server, from the application server.

Even if I'm not a fan of this approach, it does have its merits, as it would reduce the data being moved back and forth the two servers.

By itself, It still wouldn't be as impactful as batching, because there still is disk I/O, and at least in our scenario, bringing more data in a single query would still be more efficient than making a lot more single data point queries.

### Over-estimating scale

One common thing I notice when discussing a problem such as this, is that folks over-estimate scale.

Remember the file with hundreds of thousands of items I mentioned earlier? It's sized on the low dozens of megabytes 🤷‍♂️. Sure, when loaded into data structures in memory, it'll have more impact than my shrugging might make it seem, but it's still a very small scale that can be easily handled without throwing much hardware at the problem.

With this, I don't want to incentivize terrible designs, as I've seen in the past, like implementing a paginated API endpoint with in memory pagination, instead of only bringing the correct page data from the database. What I do want to incentivize, is folks thoroughly understanding the problem at hand, then evaluate the potential solutions and their tradeoffs.

### Really understand your tools

Another important tidbit I wanted to mention, is that it's really important to understand the tools you use when working on these problems.

I don't remember if it was related to the use case I'm mentioning here, or in another case, but we had one scenario, where we were doing this kind of work, and we were using an ORM (Entity Framework Core), which provided some... "entertainment".

(I'll ignore for a minute the fact that I'm not really sure it's such a good idea to use an ORM for this kind of batch processing)

EF Core, besides having a change tracker to make it easy to persist any changes, also has, when enabled, lazy loading capabilities, which are implemented with proxy objects. These two features combined, when not well understood by developers, can cause some serious headaches in terms of memory management. As folks are only thinking about the data they loaded into memory, they can forget the data structures the ORM had to put in place to support its features.

(In our particular case, a change in how the lazy loading proxies worked from one version of EF to another also didn't help, but our design was still flawed)

## Outro

That does it for this post. I thought it would be an interesting share, as even if it seems rather obvious to me, when talking to several folks, it doesn't seem that obvious to them. Many folks are a bit stuck in the common flow of receiving request -> query database -> respond, so when something slightly different comes up, they follow the exact same patterns, resulting in these kinds of issues.

I presented an example, but there's loads of them, and lots of different ways to handle these kinds of situations, so keep that in mind when you approach such a problem.

Thanks for stopping by, cyaz! 👋
