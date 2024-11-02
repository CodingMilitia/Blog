---
author: João Antunes
date: 2024-11-02 21:45:00+00:00
layout: post
title: Testcontainers and Podman
summary: Spent way too much time trying to get Testcontainers to work with Podman, to find out I just needed to set a flag 🙃.
images:
  - /images/2024/11/02/testcontainers-and-podman.png
categories:
  - csharp
  - dotnet
tags:
  - testcontainers
  - podman
slug: testcontainers-and-podman
---

## Intro

This will be a very fast one, as I'm mostly writing it for my future self. As the title implies, this is a post about working with [Testcontainers](https://testcontainers.com) and [Podman](https://podman.io), more specifically how I spent hours trying to figure out how to get them to play along, to find out in the end I just needed to set a flag 🙃.
## The scenario and the symptoms

To introduce the scenario: I'm working on an M1 MacBook, writing some tests for a C# library that interacts with MySQL. Enter Testcontainers, which fits like a glove for implementing these kinds of tests.

Then I run the tests and get a weird `NullReferenceException` somewhere inside the Testcontainers library:

```
[xUnit.net 00:00:01.44]     YakShaveFx.OutboxKit.MySql.Tests.Polling.OutboxBatchFetcherTests.Test1 [FAIL]
  Failed YakShaveFx.OutboxKit.MySql.Tests.Polling.OutboxBatchFetcherTests.Test1 [1 ms]
  Error Message:
   System.NullReferenceException : Object reference not set to an instance of an object.
  Stack Trace:
     at DotNet.Testcontainers.Containers.DockerContainer.get_State() in /_/src/Testcontainers/Containers/DockerContainer.cs:line 177

...
```

Seeing a `NullReferenceException`, at first I didn't consider it could be a Testcontainers/Podman issue, as it seems more like a coding error, either by me (more likely) or by the library authors.

I scoured the internet, with little results, and because I had played with Testcontainers before, I started to be suspicious, given the main difference between my first experience with the tool and now, was that I moved from Docker Desktop to Podman.

As I didn't find anything that helped much, I decided to give it a try with Node.js (and later Go), just to see if it was a .NET libraries issue. So I copied some samples from the Testcontainers website and gave it a go (no pun intended 😅).

The attempt with Node.js also resulted in an error, though, like in C#, nothing particularly helpful in understanding the issue. Then I tried with Go, and again got an error, but this one seemed to provide more clues: `create container: reaper: new reaper: run container: container create: Error response from daemon: container create: unable to find network with name or ID bridge: network not found`.

Still, after searching the web for some time, no luck... until, somehow, I bumped into this [Stack Overflow answer](https://stackoverflow.com/a/71549857) 🤯.  There's a bunch of things in there, but the relevant bit for me was "*Ryuk is a technology for Docker and doesn't support Podman*".

## The solution

This Stack Overflow answer finally unblocked me. [Ryuk](https://github.com/testcontainers/moby-ryuk), which is also called the reaper (remember this from the Go error message?), seems to be a technology that only works, or at least works better with Docker's Moby (Podman is supposed to be compatible, from what I understand, but clearly, at least in this case, something didn't work).

Time to give this potential solution a shot. We can disable Ryuk in a couple of ways, as documented [here](https://dotnet.testcontainers.org/custom_configuration/), using environment variables or a properties file. I went with the properties file, looking like the following:

```properties
ryuk.disabled=true
```

Ran the tests again and... success! 🥳

Also, even though we removed a component that's used to clean things up, because I'm controlling the containers lifetime within the tests (using [xUnit](https://xunit.net) fixtures), it's been working well. Maybe there's some probability, if some test run goes south, that something gets left behind, but doesn't seem like much of an issue in general.

## Outro

That's it for this short but annoying saga, to try and run Testcontainers with Podman. Writing it for my future reference, but if helps someone, even better.

As a side note, there's one little detail I left out. This whole thing actually started with [Rancher Desktop](https://rancherdesktop.io) which was also not playing nicely with Testcontainers. I tried for a bit, got annoyed, and ended up using it as an excuse  to try Podman, to see if it solved this issue (and also if it was any good in general).

Not sure if this solution also works for Rancher, as I uninstalled before installing Podman, and after I got things working, I couldn't be bothered to go back to the original setup and try things out. If it does work, let me know, and I'll leave a note for anyone interested.

Relevant links:

- [Testcontainers](https://testcontainers.com)
- [Podman](https://podman.io)
- [Rancher Desktop](https://rancherdesktop.io)
- [xUnit](https://xunit.net)

Thanks for stopping by, cyaz! 👋
