---
author: johnny
date: 2018-10-06 18:20:00+01:00
layout: post
title: 'Episode 001 - The Reference Project - ASP.NET Core: From 0 to overkill'
summary: "In this post, we'll take a look at the objective of the reference, before starting to code like there's no tomorrow without anyone understanding the context."
image: '/assets/2018/10/06/aspnet-core-from-zero-to-overkill-e001.jpg'
categories:
- fromzerotooverkill
tags:
- dotnet
- aspnetcore
---

I had already prepared the first episode starting to code the project when I realized, maybe it’s a better idea to start with explaining what’s the goal “product” wise, for everyone to have the context before starting coding like a maniac on something no one would understand.

Given this, the following video provides an overview, but if you prefer a quick read, skip to the written synthesis.

{% youtube "https://youtu.be/ABTT-YhESzc" %}

The playlist for the whole series is [here](https://www.youtube.com/playlist?list=PLN0oN9Azm_MMAjk3nhRnmHdr1l0160Dhs).
<br />
## Concepts
Like I mentioned previously, the goal of the project is to make a friendly sports event managing application, to be used by those who gather some friends to play something with their friends. The initial idea was centered around football (or soccer) but I’ll try to make it generic.

With this in mind, trying to find the main concepts that fit this idea I came up with:
- Users - well… this one is unexpected right? 😆
- Groups - groups of friends that gather to play
- Players - players are a different concept than a user to allow for groups of people where not everyone wants to use the application, but the users who want to can still manage everything even if there’s someone who’s not a registered user
- Match/event/game - represents the specific event that has/is/will happen
- Teams - the teams that play the match - this may very well be something fixed, but also something scoped to a match, where the teams are defined each time
- Statistics - something interesting is to keep statistics os the various matches, like match scores, goal/point/other scorers, fouls, etc

## Desired features
While writing down the concepts, the desired features started to come to mind (along with some components that’ll be written down in the next section):
- Create/manage users
    - Define email, name, nickname, profile picture, …
- Communication
    - Send emails, notifications, …
    - Manage preferences
- Create/manage groups
    - Invite users
    - Create players
    - Associate players with users
    - Set a specific sport for the group
    - ...
- Matches
    - Schedule
    - Manage information
        - When, where, players involved, teams chosen, …
    - Add events
        - Goal scored, foul suffered, … (more and sport dependent)
    - Live
        - Be able to add match events during the match and be able to see the stats evolve realtime

## Components
Ok, with the concepts and desired features laid out as best as possible for this kind of project (if a business analyst gave something like this to one of us, we would probably be enraged 😝) we can start to picture our over-engineered system, where we try to fit in every possible hipster technology… and as this is that kind of project, that’ll be exactly what’ll happen!

The following is something of the sort of a very high level architecture. It’ll most certainly change along the way (for the best hopefully) but I think can summarize the initial ideas.

![High level architecture](/assets/2018/10/06/architecture-overview.jpg)

We can see basically an organization of the various components, where even if we’re over-engineering the solution, we don’t want the components to be like CRUD services, but be as much as possible self contained services that can implement a specific feature set.

With this in mind we have:
- `Users` - to handle registration, authentication and other user management requirements
- `GroupManagement` - to handle everything group related, from associating users to managing players.
- `Communication` - to handle all communication requirements, including the user’s communication preferences.
- `Matches` - to handle everything match specific, like when and where, scores, match events and more, except the live part.
- `LiveMatches` - will probably be separated from the main matches component as it’ll have the realtime requirement and maybe useful to be isolated.
- `Statistics` - this will be a central place to handle all statistics, even if they originally come from the match events - we'll see in the long run how (and if we should) orchestrate this service with the matches one.

Besides this components responsible for the application logic, we'll need a frontend to see it all working. To feed the frontend I'm thinking of going with a "backend for frontend", to provide everything in the best way possible for the web frontend. The mobile part is out of scope, just added to the image as a possibility.

## Outro
Not being a specification (not even close), I think this post can give a good initial idea of what’s the desired end result. If not, let me know so I can improve it.

Thanks for stopping by, cyaz!
