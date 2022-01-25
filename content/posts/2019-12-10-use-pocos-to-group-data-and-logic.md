---
author: João Antunes
date: 2019-12-10 20:00:00+00:00
layout: post
title: "Use POCOs to group data and logic - there's more to life than services and DTOs"
summary: "In C# land, the abuse of services and DTOs is prevalent. How about grouping data and behavior using POCOs? You know, ob)ect oriented stuff 🙂."
images:
- '/images/2019/12/10/kevlinhenney-tweet.png'
categories:
- dotnet
tags:
- dotnet
- csharp
- poco
- dto
- service
slug: use-pocos-to-group-data-and-logic
---

## Intro

Something I see is prevalent in .NET (C#) projects is the abuse of DTOs (data transfer objects) and services.

DTOs (as per definition) just keep some data and the services keep all the logic, manipulating said DTOs. If a bit of logic doesn't really feel like it belongs to some service, off we go to create some helper or extension to drop said logic in.

This way of doing things is much more procedural than object oriented, which would be the primary programming paradigm we'd like to use in C# projects, with some sprinkles of functional and, sure, procedural.

## POCOs, DTOs and services

To start with, POCO means **P**lain **O**ld **C**lr **O**bject (being the original term [POJO](https://en.wikipedia.org/wiki/Plain_old_Java_object), referring to Java).

For a quick definition, I refer to Kevlin Henney's [tweet](https://twitter.com/KevlinHenney/status/1021670992152866817).

{{< embedded-image path="/images/2019/12/10/kevlinhenney-tweet.png" alt="Kevlin Henney tweet" target_url="https://twitter.com/KevlinHenney/status/1021670992152866817" >}}

> "Once more, with feeling:<br/>
• POJO does not mean DTO.<br/>
• POJO does not mean that a class has to be as dumb as rocks, with nothing more than getters and setters.<br/>
• POJO means that a class is not tied to a particular framework, whether by inheritance or annotation."

The definition of POCO is broad, and if we want to be completely clear on it, a service (as long as not dependent on anything framework specific) is also a POCO, but the category of classes I want to discuss here are normally smallish ones, that represent a simple concept.

Comparing with a DTO (that may also be a POCO itself), which by definition doesn't really have logic, just groups data, a POCO doesn't really have that restriction, having logic is welcome.

If we go back to the basics of OOP, data and behavior are meant to be grouped together, and by abusing DTOs and services we're doing the exact opposite. This doesn't mean we shouldn't use DTOs and services, they have their place, just not everywhere 🙂.

## A practical example

Let's see an example. Imagine we want to represent a football (of the soccer kind 🙃) match score sheet with a class, named `ScoreSheet`.

The DTO version could be something like:

```csharp
public class ScoreSheetDto
{
    public TeamDto Team1 { get; set; }
    public TeamDto Team2 { get; set; }
    public uint Team1Goals { get; set; }
    public uint Team2Goals { get; set; }
    public List<GoalInfoDto> Goals { get; set; }
}
```

With class like this, the responsibility to set the team information correctly, initializing the goal information and updating everything is delegated to some service. As everything in the class is a property with getters and setters, any piece of code that gets a hand of an instance of the class can make some changes, possibly ignoring rules that apply to its behavior.

Instead, to make sure the score sheet concept's logic is always applied, we can (should?) move it together with its related data.

```csharp
public class ScoreSheet
{
    private List<GoalInfo> _goals;

    public ScoreSheet(Team team1, Team team2, uint team1Goals, uint team2Goals, IEnumerable<GoalInfo> goals)
    {
        Team1 = team1 ?? throw new ArgumentNullException(nameof(team1));
        Team2 = team2 ?? throw new ArgumentNullException(nameof(team2));
        Team1Goals = team1Goals;
        Team2Goals = team2Goals;
        _goals = new List<GoalInfo>(goals ?? throw new ArgumentNullException(nameof(goals)));
        ThrowOnGoalCountAndInfoMismatch();
    }

    public Team Team1 { get; }
    public Team Team2 { get; }
    public uint Team1Goals { get; private set; }
    public uint Team2Goals { get; private set; }
    public IReadOnlyCollection<GoalInfo> Goals => _goals.AsReadOnly();

    public void Score(Team forTeam, Player scorer, short minute)
    {
        if(forTeam is null)
        {
            throw new ArgumentNullException(nameof(forTeam));
        }

        if(scorer is null)
        {
            throw new ArgumentNullException(nameof(scorer));
        }

        if(Team1.Equals(forTeam))
        {
            ++Team1Goals;
        }
        else if(Team2.Equals(forTeam))
        {
            ++Team2Goals;
        }
        else
        {
            throw new ArgumentException("The provided team is not part of this score sheet");
        }

        _goals.Add(new GoalInfo(forTeam, scorer, minute));
    }

    private void ThrowOnGoalCountAndInfoMismatch() => throw new NotImplementedException("TODO");
}
```

Ignoring the fact that the logic could be better organized, the gist of it is that now the class contains the logic to interact with its data. The data can be accessed by external code, via its public getters, but it cannot be changed (private setters). All the changes go through the class logic, so if there is an incorrect interaction, the class will signal it, in this case through exceptions.

Instead of creating massive services that have all the logic, interact with DTOs that have no encapsulation, using POCOs to group data and behavior regarding specific concepts can make your code more cohesive, less error prone and easier to reason about.

## Looking at the framework for examples

The framework itself is a great place to look for POCOs in the wild. There's no shortage of examples, but let's use a simple one, the [`Uri` class](https://docs.microsoft.com/en-us/dotnet/api/system.uri?view=netcore-3.0).

The `Uri` class is a nice example of a pretty simple POCO, which if you're using web applications (or at least making HTTP requests to APIs) you probably used it.

Instead of just using a `string`, then having a bunch of functions somewhere to do URI related stuff, the `Uri` class itself already exposes such functionality, besides storing the data related to a specific URI.

As a side note, the `Uri` class is also great to use to fight [primitive obsession](https://blog.ploeh.dk/2011/05/25/DesignSmellPrimitiveObsession/), by avoiding passing around `string`s everywhere.

## The DDD connection

A thing I noticed is that as soon as I start talking about things like the `ScoreSheet` class introduced above, some people immediately think "ah, you're talking about DDD".

I understand the reasoning, as much of the DDD literature shows implementation examples using object oriented languages, in which the recommended approach for implementing the domain concepts is through POCOs that represent the entities and value objects (would probably be different in other paradigms, namely functional).

So, sure, if you're implementing something with DDD in an objected oriented language, you're very likely using POCOs in the way I'm talking about, but you can also take advantage of this POCO approach even if you're not doing DDD. A clear example is if you use the `Uri` class previously discussed. The URI concept doesn't need to be part of your domain to make sense to use it.

## Outro

I'm certainly not the first one to write about this, having certainly learned a lot from other people that talked about it, but as it seems the prevalent practice is to abuse services and DTOs, I thought it wouldn't harm to give it another push.

If you agree, "spread the word"! Friends don't let friends DTO all the things! 😆

Thanks for stopping by, cyaz!
