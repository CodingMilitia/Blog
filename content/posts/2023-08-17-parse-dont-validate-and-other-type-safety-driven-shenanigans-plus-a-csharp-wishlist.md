---
author: João Antunes
date: 2023-08-17 09:00:00+01:00
lastmod: 2023-08-17 14:30:00+01:00
layout: post
title: "\"Parse, don't validate\" and other type safety driven shenanigans (plus a C# wishlist)"
summary: "Making use of the type system is something I feel should be important when working in a strongly typed language like C#. However, I don't feel like that's the case, and I would love for the language to push folks in the direction of creating more robust programs, where the compiler provides more help in proving the correctness of the code."
images:
- '/images/2023/08/17/parse-dont-validate-and-other-type-safety-driven-shenanigans-plus-a-csharp-wishlist.png'
categories:
- csharp
- dotnet
tags:
- 'type safety'
- 'type system'
slug: parse-dont-validate-and-other-type-safety-driven-shenanigans-plus-a-csharp-wishlist
---

## Intro

In the last few years, one of the things I've been more interested in, is to try and make the most of type safety, making things explicit, as well as having correctness proven as much as possible at compile time, instead of runtime.

C# is a strongly typed language, however it has major deficits in some areas, which makes it harder to write safer code, pushing folks into writing less robust code, just to avoid the complexity. While it's certainly better than dynamically typed languages, sometimes it feels not good enough.

PS: I love C#, so I'm not bashing for the sake of bashing, I'm bashing because it feels like, with a couple of improvements to the language, we could benefit not only from the nice things C# already has, but also be able to more easily write safer and more robust code.

PS2: I know many of the problems I mention here could be solved if I just moved to F# (while new ones would arise). That's not the point of the post. I'm well aware of the existence and awesomeness of F#, but this post is specifically about some pains I feel with C#, and would love to see addressed. If you feel some of my pains, and are willing to move to a different language within the same platform, do consider F# 😉.

## Parse, don't validate

One article in particular that resonated with me on this subject is ["Parse, don’t validate"](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/). It's not the easiest of reads, in particular with the Haskell examples, but I highly recommend it.

With this article, mixed with other ideas I was considering at the time, as well as other articles I was reading, one of the things that stuck with me, is to try to represent things as much as possible with the type system, including parsing from one type to another.

As an example, imagine we have a concept that's represented by a string, though it requires that the string has a specific format. The common approach to do this in a C# web API, is to create some validator, be it with the [built-in validation attributes](https://learn.microsoft.com/en-us/aspnet/core/mvc/models/validation?view=aspnetcore-7.0) or with a library like [FluentValidation](https://github.com/FluentValidation/FluentValidation), but then keep using the string everywhere. That works, but it's not particularly better than what a dynamically typed language would give us.

Alternatively, to actually make use of the type system, we could create a type representing the concept, containing any rules associated with it, including a function that would parse a string to create an instance of the type. This function would need to have an awkward return type (awkward for what C# devs are used to), which would be either an instance of the type when parsing is successful, or something else (or even nothing) when parsing is not successful.

I'm talking about parsing, but in reality I believe this reasoning should be applied to most code, in particular domain logic, representing explicitly the successful and failure possibilities, instead of throwing exceptions for non-exceptional reasons.
## Enter C# 

Ok, after this big intro about type safety, parsing and whatnot, how does C# fare in this regard? Well, it's not great.

C# seems to me to be lacking in providing good ways to implement things in this way, pushing folks away from (what I believe are) good practices like making better use of types, avoiding primitive obsession, minimizing exceptions for non-exceptional outcomes and so on.

Things like optional/maybe and result/either types, as well as discriminated unions, all common in languages and libraries with a focus on type safety, are nowhere to be found, and even if we hack our own (more on that later), the lack of language support makes things harder to work with than it should.

Looking specifically at parsing, the best we seem to be able to do, is use the `TryXyz` pattern, like we have in `Guid.TryParse` or `int.TryParse`. It's not great, but it's the closest we seem to get to doing things in a type safe way (with caveats), as well as without throwing exceptions left and right.

## language-ext

As we discussed, we don't have great support neither from the language or the standard library. Regarding the language, there's not much we can do on our own, but regarding types, nobody prevents us from creating our own.

That's what [language-ext](https://github.com/louthy/language-ext) is all about. It's a pretty cool library, providing types like the ones I mentioned above, along with functions to work with them in a functional programming inspired way.

I've been using it, in particular for its `Option` and `Either` types, which makes it easier to write more explicit code. With the accompanying functions, there are a bunch of scenarios that are very streamlined, with a LINQ feel to it.

> Note: this post isn't about language-ext, so there won't be more complete samples, but you can checkout the [samples on GitHub](https://github.com/louthy/language-ext/tree/main/Samples)

So, if I can use a library like this, then what's the problem? Well, the problem is that, it makes some things easy, but others harder, and I wouldn't attribute this to the library, but to the lack of language support.

For a bunch of domain logic, as well as somewhat simple orchestration (e.g. the code that gets things from repositories and invokes the domain logic), everything works smoothly. However, when things get slightly less straightforward, like having to fetch multiple dependencies before invoking the domain logic, sometimes with a particular order, to fail fast with a specific error and stuff like that, things get messier, and the code becomes much less readable.

Of course, this could also be an issue with the design I came up with, but that kinda gives me some reason on the subject of the language (and libraries) not making it easy for me to write the most robust code.

Another issue, even in scenarios where it's doable to do everything with language-ext without things getting messy, is the fact that it's not idiomatic C# at all (again, not criticizing the library, but more the current state of idiomatic C#). If it's some piece of code that will only be touched by me, it's not a problem, as I'm used to looking at other languages, so it's irrelevant if it's idiomatic C# as long as I find it readable. However, when it's code for other folks to work on, while not mandatory to be idiomatic (otherwise I would've given up a long time ago, cause focus on type safety and idiomatic C# is not even a thing), but it needs to be really obvious what's going on, and language-ext ain't obvious for most C# devs (unless maybe the ones that worked with Rx, in which case things are probably more familiar).
## A sample scenario

Let's get a bit more specific now. Imagine we want to implement an API endpoint, which gets a couple of ids for a couple of entities (one a string, the other a GUID), loads these entities from their respective repositories, invokes domain logic on one of them, getting the other as an argument, wrapping up by saving the changed entity.

Simple stuff, that we'd normally solve with:
1. Implementing a validator that would ensure the endpoint inputs were valid, throwing an exception otherwise
2. As things were validated, we use the ids as is, i.e. passing the string and GUID everywhere
3. Fetch the entities, throwing something like a not found exception if we got null as a result (or do nothing and get a null reference exception eventually)
4. Invoke the entity's domain logic, which would throw if something didn't match its logic
5. Save the changed entity
6. Return successful HTTP result

The endpoint code, could look something like this:

```csharp
private static IResult Handle(  
    Request request,  
    SomeThingRepository someThingRepository,  
    SomeOtherThingRepository someOtherThingRepository)  
{  
    Validate(request);  
    var someThing = someThingRepository.Load(request.SomeId);  
    var someOtherThing = someOtherThingRepository.Load(request.SomeOtherId);  
    someThing.DoSomethingRequiringOtherThing(someOtherThing);  
    someThingRepository.Save(someThing);  
    return Results.NoContent();  
}
```

Seems simple enough, and it actually is... that is, if we only consider the happy path, when everything follows the most expected flow. As soon as we're off the happy path, things ain't so simple.

For starters, it can throw a bunch of exceptions. I've said it probably too many times, but I'll say it again: exceptions shouldn't be used for non-exceptional cases. Invalid inputs aren't exceptional; not found entities aren't exceptional; breaking a domain rule isn't exceptional. Not only are we using a mechanism that should be used only when things go terribly wrong, we're hiding the behavior of our components, because it's not part of the functions signature, so we don't even know that they might happen (and even if they were part of the signature, like in Java, things would only get messier, not less).

Regarding the types of the ids, or other types that should represent concepts of our domain, given how simple this example endpoint is, it's probably not the best example of the problems associated with this primitive obsession, but be sure that in a more realistic scenario, having the concepts encapsulated in types, centralizing their rules and disallowing us from using them in ways they're not supposed to, that's one of those aha moments where you truly start to appreciate type safety.

In the code above, you'll notice that I'm not checking if the entities loaded are null or not. Why not? Maybe inside the `Load` method that's checked and an exception is thrown. Or maybe it isn't and I should check, but I have no way of knowing that, because the type system doesn't tell me, so I need to remember I should probably check? That's not great...

> Just a note before proceeding, regarding the possibility of the entities not being found, that's the only part of this whole thing where the current C# type system could actually help us, using [nullable reference types](https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references)

## How could this look like with language-ext

Continuing with the scenario described in the previous section, let's just take a quick glance at how it could be implemented with language-ext (probably could be made better by someone working more extensively with the library than I have).

Before that, just a little additional context, to make it easier to understand (though I can imagine it still won't be that easy):

- The main types used from language-ext are `Validation` (to aggregate the parsing errors), `Option` (to represent something that may or may not be present) and `Either` (in which left represents an error, while right represents a success)
- `SomeId.Parse` and `SomeOtherId.Parse` return an `Option`, which contains a value when parsing was successful, nothing when parsing wasn't possible
- The repositories `Load` method also return an `Option`, to make it clear that the entity may or may not be found
- `DoSomethingRequiringOtherThing` returns `Either<string, Unit>`, where the string represents the error reason, while `Unit` is like `void`, but unlike it, we're able to use it in generic arguments

```csharp
private static IResult Handle(  
    Request request,  
    SomeThingRepository someThingRepository,  
    SomeOtherThingRepository someOtherThingRepository)  
    => Parse(request)  
        .MapLeft(Results.BadRequest)  
        .Bind(input => someThingRepository  
            .Load(input.someId)  
            .ToEither(() => Results.NotFound())  
            .Map(someThing => (input.someOtherId, someThing)))  
        .Bind(input => someOtherThingRepository  
            .Load(input.someOtherId)  
            .ToEither(() => Results.NotFound())  
            .Map(someOtherThing => (input.someThing, someOtherThing)))  
        .Bind(input => input.someThing  
            .DoSomethingRequiringOtherThing(input.someOtherThing)  
            .BiMap(  
                Right: _ => input.someThing,  
                Left: Results.Conflict))  
        .Match(  
            Right: someThing =>  
            {  
                someThingRepository.Save(someThing);  
                return Results.NoContent();  
            },            
            Left: errorResult => errorResult);
            
private static Either<ValidationError, (SomeId someId, SomeOtherId someOtherId)> Parse(Request request)  
    => (  
            SomeId.Parse(request.SomeId).ToValidation("someId"),  
            SomeOtherId.Parse(request.SomeOtherId).ToValidation("someOtherId")  
        )        
        .Apply((someId, someOtherId) => (someId, someOtherId))  
        .ToEither()  
        .MapLeft(errors => new ValidationError(errors));
```

In terms of type safety, as well as being explicit and accounting for the various flow possibilities of our scenario, this code is much better than the one presented previously. However, as I mentioned earlier, this seems way too far from idiomatic C# to get buy-in from folks (and I imagine it's likely you're thinking about what the hell is going on here right now 😅).

So, the first example I showed was apparently nice and simple, but hid a lot of pitfalls. This second example got rid of the pitfalls, made everything explicit, but in a way that's far from familiar to C# devs. Can we get something in between? Not really sure, but we can try.

## Massive shenanigans

Let's try to do something about the scenario and examples we've looked in the previous sections.

What approach can we take? If you have better ideas, please let me know, but the one I came up with, was to create some extensions to ~~Frankenstein~~ bridge the gap between the functional approach of language-ext and the imperative nature of more idiomatic C#. Apologies in advance to the author of language-ext, which makes it clear in the repo he hates the `TryXyz` pattern, and that's exactly the approach I took to try to make this bridge 😅.

Another look at code!

```csharp
private static IResult Handle(  
    Request request,  
    SomeThingRepository someThingRepository,  
    SomeOtherThingRepository someOtherThingRepository)  
{  
    if (!Parse(request).TryGetValue(out var validationError, out var parsedInputs))  
    {        
        return Results.BadRequest(validationError);  
    }    
    
    var maybeSomeThing = someThingRepository.Load(parsedInputs.someId);  
  
    if (!maybeSomeThing.TryGetValue(out var someThing))  
    {        
        return Results.NotFound();  
    }    
    
    var maybeSomeOtherThing = someOtherThingRepository.Load(parsedInputs.someOtherId);  
    
    if (!maybeSomeOtherThing.TryGetValue(out var someOtherThing))  
    {        
        return Results.NotFound();  
    }    
    
    var result = someThing.DoSomethingRequiringOtherThing(someOtherThing);  
    
    return result.Match(  
        Right: _ =>  
        {  
            someThingRepository.Save(someThing);  
            return Results.NoContent();  
        },        
        Left: reason => Results.Conflict(new DomainError(reason)));  
}  
  
private static Either<ValidationError, (SomeId someId, SomeOtherId someOtherId)> Parse(Request request)  
    => (  
            SomeId.Parse(request.SomeId).ToValidation("someId"),  
            SomeOtherId.Parse(request.SomeOtherId).ToValidation("someOtherId")  
        )        
        .Apply(static (someId, someOtherId) => (someId, someOtherId))  
        .ToEither()  
        .MapLeft(errors => new ValidationError(errors));
```

As you can see, I'm using mostly the same things from language-ext, `Validation`, `Option` and `Either`, but mostly ignored the fluent interface, instead bolting a couple of `TryGetValue` extension methods on them.

The result isn't particularly pretty, but I think it is much more familiar and easier to read for C# devs, while retaining most of the type safety we got from the previous approach. I say most of the type safety, because there's one caveat where the compiler won't help us, which is when we use structs.

The signature for one of the `TryGetValue` extensions is the following:

```csharp
public static bool TryGetValue<T>(this Option<T> option, [MaybeNullWhen(false)] out T value)
```

The `MaybeNullWhen` attribute will make the compiler let us know that if the return value is false, then the value might be null, so we can't make the mistake of using it when we shouldn't. That is, if our T is a reference type. If T is a value type, then it can't be null, so now we can mess things up and use it when we shouldn't, because the compiler doesn't warn us about anything.
## Wishlist

If you got so far into the post, thanks and congrats for your patience, I'm pretty sure this hasn't been an easy read, but I hope I've been able to explain my pains well enough.

Given the name of this section, I think you're able to figure out the answer to the question, am I happy with the last approach, or any of them for that matter? Nope, not really.

- The first approach described, for me at least, is unacceptable - we might as well be working with a dynamically typed language if we're gonna do things like that
- The full-on language-ext approach also doesn't feel like the right fit for me, as I want to keep things more familiar to what folks are used to (and can get unwieldy in less straightforward scenarios)
- The final Frankenstein approach is probably what I'll be using until I come up with something better (or you recommend something better 😉), but I'm not super happy with it, as it's not as type safe as it should be, has some awkwardness to it and is much more verbose than it should

I really think C# should make some improvements in this regard, to make it easier to write more robust code. The latest versions of C# have had some interesting features, and I use a bunch of them, but I feel like the priorities aren't well defined, as the latest more mainstream features are more nice to haves, instead of something more impactful in making it easier to write correct code. Some ideas that come to mind:

- Add option/maybe, result/either and other such types to the standard library
- Add language support to make using these constructs straightforward
- The idea is not to turn C# into a functional language, but add language support so that these kinds of things can be easily implemented with the more imperative nature of C# that folks are used to
- An example of a language construct that would make things dramatically easier, would be to be able to, without a bunch of boilerplate, return immediately when an option instance is empty, or a result instance contains an error, for example, [like Rust has the question mark operator](https://doc.rust-lang.org/std/result/#the-question-mark-operator-)
- Add discriminated unions, so that, as a simple example, returning multiple types of errors from the same function is easier to implement and handle

Sadly, I'm not optimist any of these will happen. Most of them don't seem to even be on the radar, and discriminated unions have been talked about for ages and they're yet to see the light of day. Folks seem to be happy with the status quo, adding some quality of live improvements here and there and chugging along. Maybe I'll need to content myself with what we have, or look elsewhere (maybe it's time to learn Rust and see if I like it better 🙂).

Let me know, do you think these wishes make sense, or not at all? Any other ideas you'd add?

## Outro

That does it for this post. I don't feel it was a very "ranty" post, but it certainly originated from my frustration of not being able to make the use of the type system as much I would like to with C# (as well as some frustration with the current state of affairs of idiomatic C#).

We looked at topics like:

- Why type safety is important in helping us ensure correctness at compile time
- What's considered idiomatic C# isn't particularly better than in the dynamically typed languages many C# devs love to bash on
- Using libraries to try and fill some of the language and standard library gaps
- Bridging the gap between functional and imperative ideas
- Some features I'd love to see in future C# and .NET versions

Relevant links:

- [Sample code](https://github.com/joaofbantunes/ParseDontValidateAndOtherTypeSafetyDrivenShenanigans)
- [Parse, don’t validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
- [language-ext](https://github.com/louthy/language-ext)
- [Rust Result and question mark operator](https://doc.rust-lang.org/std/result/#the-question-mark-operator-)

Thanks for stopping by, cyaz! 👋
