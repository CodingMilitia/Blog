---
author: João Antunes
date: 2022-09-01 18:30:00+01:00
layout: post
title: "Beware of records, with expressions and calculated properties"
summary: "I’ve been using C# records a lot since the feature was introduced. However, when using them, we really need to understand how they work, otherwise we might face unexpected surprises."
images:
- '/images/2022/09/01/beware-of-records-with-expressions-and-calculated-properties.png'
categories:
- csharp
- dotnet
tags:
- records
slug: beware-of-records-with-expressions-and-calculated-properties
---

## Intro

I’ve been using C# records a lot since the feature was introduced, probably too much 😅.

Given their terseness, even ignoring the other records characteristics, as soon as I’m creating a type that I’m expecting to be immutable, I immediately start with a record. However this might not always be adequate, and in this post we’ll look at a very specific example where I introduced a bug in the code due to not taking into consideration all of the records characteristics.

## A record with a calculated property

Let’s take the following record as an example:

```csharp
public record SomeRecordWithCalculatedProperty(string SomeValue)
{
    public string SomeCalculatedValue { get; } = SomeValue + " *calculated*";
}
```

What’s going on here, is that we’re using the primary constructor to declare and initialize the `SomeValue` property, but we also have another property, `SomeCalculatedProperty`, that’s calculated using `SomeValue`. Seems simple enough, right?

Now let’s take a look at the following usage of this record:

```csharp
var x = new SomeRecordWithCalculatedProperty("This is some value");
Console.WriteLine(x);
var y = x with {SomeValue = "This is another some value"};
Console.WriteLine(y);
```

So, we’re creating a record, printing it to the console, then using the `with` expression to create a copy of that record, changing the `SomeValue` property.

Now give it a think for a while, what do you think will be printed to the console?

It’s the following (new lines added for readability):

```bash
SomeRecordWithCalculatedProperty 
{ 
    SomeValue = This is some value,
    SomeCalculatedValue = This is some value *calculated*
}
SomeRecordWithCalculatedProperty 
{
    SomeValue = This is another some value, 
    SomeCalculatedValue = This is some value *calculated* 
}
```

Can you spot the problem?

`SomeCalculatedValue` in the second line still has the same value as in the first. This makes sense, as what happens when we use the `with` expression, is that the record is cloned and then the properties provided within the brackets are overwritten.

This shouldn’t be a surprise, but again, given I was just blindly using records for the terseness, it did, in fact, get my surprise when I was trying to figure out the issue 🙂.

## Calculating the property on the fly

An immediate and easy way to fix this, is to, instead of calculating the property value and setting it, just calculate it on the fly, like so:

```csharp
public record SomeRecordWithOnTheFlyCalculatedProperty(string SomeValue)
{
    public string SomeCalculatedValue => SomeValue + " *calculated*";
}
```

This solves the problem, and might be a valid solution in general. It has one potential issue though: every time we use `SomeCalculatedValue`, like the method that the property get accessor is, the value will be calculated. Depending on the way the property is used, it might not be a problem, or it might be, as we’re always repeating the same logic and creating new objects to return. In my case, I was using the property multiple times, and the calculation logic was a bit more complex than this simplified example, so it would be nice to avoid executing this code more often than actually needed.

## What if we make it lazy

Spoiler alert, you’ll probably see the issue as soon as I drop the code snippet, but, what if we make it lazy, using a `Lazy<T>` type? The code would look the following:

```csharp
public record SomeRecordWithLazilyCalculatedProperty(string SomeValue)
{
    private readonly Lazy<string> _someCalculatedValue = new(() => SomeValue + " *calculated*");

    public string SomeCalculatedValue => _someCalculatedValue.Value;
}
```

What do you think, is this the solution to our problems? Give it a thought before looking the the output below 😉.

```bash
SomeRecordWithLazilyCalculatedProperty
{
    SomeValue = This is some value,
    SomeCalculatedValue = This is some value *calculated*
}
SomeRecordWithLazilyCalculatedProperty
{
    SomeValue = This is another some value,
    SomeCalculatedValue = This is some value *calculated*
}
```

Yup, it’s the same. The problem is the same, but this time, instead of copying the property, it was the backing field that was copied.

Just if you’re curious, using a decompilation tool (I used [sharplab.io](http://sharplab.io/)), we can see the generated copy constructor, which takes care of the object cloning:

```csharp
protected SomeRecordWithLazilyCalculatedProperty(
[System.Runtime.CompilerServices.Nullable(1)] SomeRecordWithLazilyCalculatedProperty original)
{
	<SomeValue>k__BackingField = original.<SomeValue>k__BackingField;
	_someCalculatedValue = original._someCalculatedValue;
}
```

So, if the lazy field is copied, which was already initialized as we had already printed the first record instance, its contents are the same in the new cloned instance.

## Overriding the copy constructor

Let’s take another stab at fixing the issue. Let’s keep the lazy field, but we’ll override the copy constructor.

A copy constructor is a constructor that gets an instance of the same type as the sole parameter. It should be protected when the record isn’t sealed, private otherwise.

```csharp
public record YetAnotherRecordWithLazilyCalculatedPropertyAndCopyCtor(string SomeValue)
{
    private readonly Lazy<string> _someCalculatedValue = new(() => SomeValue + " *calculated*");

    public string SomeCalculatedValue => _someCalculatedValue.Value;

    protected YetAnotherRecordWithLazilyCalculatedPropertyAndCopyCtor(
        YetAnotherRecordWithLazilyCalculatedPropertyAndCopyCtor original)
    {
        SomeValue = original.SomeValue;
        _someCalculatedValue = new(() => SomeValue + " *calculated*");
    }
}
```

As we can see, we’re overriding the copy constructor, and when the time comes to initialize `_someCalculatedValue`, instead of copying the value from the original, we’re creating a new instance (but the record definition isn’t looking so terse now 😭).

Does this solve our problem? The output is:

```bash
YetAnotherRecordWithLazilyCalculatedPropertyAndCopyCtor
{
    SomeValue = This is some value,
    SomeCalculatedValue = This is some value *calculated*
}
YetAnotherRecordWithLazilyCalculatedPropertyAndCopyCtor
{
    SomeValue = This is another some value,SomeCalculate
    dValue = This is another some value *calculated*
}
```

It does! Because we create a new instance of the lazy backing field, the value is only calculated when needed, which in this case is when we print to the console, having the new `SomeValue` in place.

Note that this wouldn’t work if it wasn’t for the lazy, as at the time the copy constructor is invoked, we still don’t know what the new `SomeValue` will be.

## Other options

There are, of course, other options to solve this issue. 

We could, for example, declare `SomeValue` outside of the primary constructor and make it have a getter only, instead of the getter and init setter that’s generated when using the primary constructor. The side effect would be that we no longer can use a `with` expression to clone the record and change `SomeValue`.

We could also manually implement the init setter of `SomeValue`, so it would force recalculation of `SomeCalculatedValue`.

I imagine there are more options that I’m not remembering right now, but you get the point.

## Outro

To summarize, I’ve misused records and got myself head-scratching while trying to figure out what was going wrong. To be clear, it was a mess of my own doing, I’m not blaming the feature at all, but thought I might not be the only one caught off guard with something like this, so thought of sharing this post 🙂.

At the end of the day, just keep in mind the characteristics of records, how they differ from classes, and use the right tool for the job, don’t be like me and get bedazzled by one of the features, ignoring everything else that comes along with it.

Thanks for stopping by, cyaz! 👋
