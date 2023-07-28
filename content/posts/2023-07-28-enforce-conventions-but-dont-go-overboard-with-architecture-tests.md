---
author: João Antunes
date: 2023-07-28 08:45:00+01:00
layout: post
title: "Enforce conventions (but don't go overboard) with architecture tests"
summary: "I've been trying to flee the over-engineered C# solution structures, with multiple projects, with related code scattered around. Even if an imperfect solution, I've turned to NetArchTest to help in guide on following some conventions in a simplified solution structure."
images:
- '/images/2023/07/28/enforce-conventions-but-dont-go-overboard-with-architecture-tests.png'
categories:
- csharp
- dotnet
tags:
- NetArchTest
- 'architecture tests'
---

## Intro

This will be a quick one, as I posted about it on LinkedIn, and thought I might as well add it to the blog as well, so it's in a central spot for me to find if I need it in the future 😅.

Yesterday I was working on a new solution, and messing around with its structure, trying to come up with something simpler than the typical over-engineered solutions, with a bunch of projects to compose a single self-contained application, with things scattered around, so related code ends up not being as close as it should ideally be.

While thinking on how to go about things, and chatting with colleagues, it became apparent that even with a simplified structure, folks still value some sort of automated help in enforcing internal conventions. With this in mind, I turned to NetArchTest to help implementing this enforcement.

## Using NetArchTest and getting rid of multiple of projects

This post isn't intended to be an in-depth look at NetArchTest, but just show a couple of examples of things we can do with it, to help in enforcing conventions.

Something that's generally a good idea, is to have domain code independent of application and infrastructure code (["functional core, imperative shell"](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell) , ["hexagonal architecture", "ports and adapters"](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)) and all of that). One of the ways this is typically achieved, is by spreading things in multiple projects, so that, for example, the domain code can't reference the infrastructure. The side effect of this, is that we complicated our solution structure, and have related code scattered.

I prefer a more [vertical slice](https://en.wikipedia.org/wiki/Vertical_slice) approach to structuring solutions, but the "problem" is that now the separation between domain and infrastructure code can't be enforced (I put problem in quotes, because I don't really find it to be a problem, depends on your consistency when modeling these concepts in code).

Enter NetArchTest to help with enforcing that the domain code doesn't reference the infrastructure.

Imagine the following project structure:
```
Api
├─ Features
│  ├─ Hellos
│  │  ├─ Application
│  │  │  ├─ Dtos.cs
│  │  │  ├─ Endpoints.cs
│  │  │  ├─ PersistenceContracts.cs
│  │  ├─ Domain
│  │  │  ├─ Model.cs
│  │  ├─ Infrastructure
│  │  │  ├─ HelloRepository.cs
│  ├─ Goodbyes
│  │  ├─ Application
│  │  ├─ ...
│  ├─ Shared
│  │  ├─ ...
├─ ...
```

With this structure, we have things a bit less scattered all over the solution. Everything related to the set of `Hellos` features is in the `Hellos` folder, likewise for `Goodbyes`, and anything that's shared between them, can go into the `Shared` folder (apologies for the weak example, but the brain didn't feel like coming up with something better 😅).

Please note that I'm not saying this is the best structure ever, it's just a structure, that I believe can be better than the multiple projects approach. I don't believe in the typical dogmatic approach of folks when talking about these kinds of things, like there was a "one true way" 😐. In fact, even though I'm proposing this structure, it's not really my favorite, as it has too many folders for my liking. Depending on the code's complexity, I'd probably get rid of the `Application`, `Domain` and `Infrastructure` folders, and simply use a different way of naming the files to make things flatter but still easily understandable. Folks do typically prefer to use folders, so I'm making a concession here 😅.

So, with this structure in mind, how can NetArchTest help us? We can define the following test (side note, besides NetArchTest, these tests are using [xUnit.net](https://xunit.net) and [FluentAssertions](https://fluentassertions.com)):

```csharp
using NetArchTest.Rules;  
using static Api.Tests.NetArchTestExtensions;  
  
namespace Api.Tests;  
  
public class ArchitectureTests  
{  
    [Fact]  
    public void When_Developing_Domain_Types_They_Should_Not_Reference_Infrastructure_Types()  
        => ApiTypes()  
            .That()  
            .ResideInDomainNamespaces()  
            .ShouldNot()  
            .HaveDependencyOnAny(ApiTypes().InfrastructureNamespaces())
            .Evaluate();  
}

// created these extensions on top of NetArchTest to make tests easier to read

file static class NetArchTestExtensions  
{  
    public static Types ApiTypes()  
        => Types.InAssembly(typeof(Program).Assembly);  
  
    public static PredicateList ResideInDomainNamespaces(this Predicates predicates)  
        => predicates.ResideInNamespaceMatching( /*language=regexp*/".*\\.Domain.*");  
  
    public static PredicateList ResideInInfrastructureNamespaces(this Predicates predicates)  
        => predicates.ResideInNamespaceMatching( /*language=regexp*/".*\\.Infrastructure.*");  
   
    public static string[] InfrastructureNamespaces(this Types types)  
        => types  
            .That()  
            .ResideInInfrastructureNamespaces()  
            .GetNamespaces();  
  
    public static void Evaluate(this ConditionList conditionList)  
    {        
        var failingTypeNames = conditionList.GetResult().FailingTypeNames ?? Array.Empty<string>();  
        failingTypeNames.Should().BeEmpty();  
    }

    private static string[] GetNamespaces(this PredicateList predicateList)  
        => predicateList.GetTypes()  
            .Where(t => t.Namespace is not null)  
            .Select(t => t.Namespace!)  
            .Distinct()  
            .ToArray();  
}
```

Now if, for example, in a domain entity, we referenced a repository, we'd get a failing test:

```
[xUnit.net 00:00:00.53]     Api.Tests.ArchitectureTests.When_Developing_Domain_Types_They_Should_Not_Reference_Infrastructure_Types [FAIL]
  Failed Api.Tests.ArchitectureTests.When_Developing_Domain_Types_They_Should_Not_Reference_Infrastructure_Types [30 ms]
  Error Message:
   Expected failingTypeNames to be empty, but found {"Api.Features.Hellos.Domain.Hello"}.
```

One cool thing that NetArchTest allows us to do, is to use patterns to match things, like namespaces. If it didn't, these tests would be a lot more painful to write and maintain, or, to avoid that pain, we could fall back to having top level folders for `Domain`, `Application` and `Infrastructure`, which would kinda defeat the purpose of moving everything to the same project, because we'd no longer have vertical slices, and would go back to having things completely scattered around.

## Another example with NetArchTest

I didn't delve too much into the capabilities of NetArchTest, cause I wanted to keep things simple, but it does have other options besides just checking references.

One thing I'm convinced of, is that classes should be sealed by default (which  I'm sure many folks will disagree with, as I've already had some arguments on the subject 😅). Not being the case, one needs to remember to add the `sealed` modifier, but that's very easy to forget to do. Enter NetArchTest again:

```csharp
// I think this is a reasonable thing to enforce, but if I'm forgetting some valid situation, we can revisit  
[Fact]  
public void When_Developing_Non_Abstract_Or_Static_Types_Then_They_Should_Be_Sealed()  
    => ApiTypes()  
        .That()  
        .AreClasses()  
        .And()  
        .AreNotAbstract() // static classes are considered abstract, so this suffices  
        .Should()  
        .BeSealed()  
        .Evaluate();
```

So, basically, I'm enforcing that all non-abstract classes should be sealed. It's probably a bit aggressive, but in the last few projects I remember, this was true, so I'm going with it. If I find a very good reason for this rule to not apply, then I'll revisit it.

If, for example, we had some generated code which didn't fulfill this rule, we could ignore that code with something like:

```csharp
// ...
	.That()
	.DoNotResideInNamespaceMatching(/*language=regexp*/".*\\.Generated.*")
	.And()
	.AreClasses()
// ...
```

## Don't go overboard

Now, as we can see with these brief examples, we can enforce a bunch of stuff with tools like NetArchTest. However, I think we should keep it as simple as possible and not go overboard.

The goal, when using these kinds of tools, should be to provide some extra confidence to folks when developing, to try to push them onto the ["pit of success"](https://blog.codinghorror.com/falling-into-the-pit-of-success/), not to enforce every little thing and remove thinking and creativity from the process.

Also, don't consider any rules enforced as "written in stone". If at some point they start to not make sense, adapt or remove them. We shouldn't be enforcing things just because... reasons, we should, again, enforce things that provide some extra confidence when developing.

## Outro

That does it for this very brief post on architecture tests in general, with a look at NetArchTest in particular.

If it came down to only my own preferences, I would just put things closer together, and rely on code reviews to ensure folks don't make a mess of things (that's the point of the reviews in the first place right? ensure the quality of the code), but as I alluded before, folks seem to value having some sort of automated guidance to follow certain practices, so in this case, NetArchTest (or other libraries or tools with similar capabilities, like [ArchUnitNET](https://github.com/TNG/ArchUnitNET) or [NDepend](https://www.ndepend.com)) can be helpful.

Like I mentioned though, don't go overboard. The goal is to provide some extra confidence to folks while they work, not to make people not think, so enforce the minimum amount of things that provides such confidence.

Relevant links:

- [Sample code](https://github.com/joaofbantunes/ArchitectureTestsSample)
- [NetArchTests](https://github.com/BenMorris/NetArchTest)
- [ArchUnitNET](https://github.com/TNG/ArchUnitNET)
- [NDepend](https://www.ndepend.com)
- [xUnit.net](https://xunit.net)
- [FluentAssertions](https://fluentassertions.com)
- [Hexagonal architecture](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software))
- [Functional core, imperative shell](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)
- [Vertical slice](https://en.wikipedia.org/wiki/Vertical_slice)

Thanks for stopping by, cyaz! 👋