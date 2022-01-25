---
author: João Antunes
date: 2019-12-01 18:00:00+00:00
layout: post
title: "\"Equals\" and \"==\" are not... equal"
summary: "\"Equals\" vs \"==\" is an old topic, but as it's often times forgotten, there's probably no harm in talking about it again."
images:
- '/images/2019/12/01/equals-and-equals-are-not-equal.jpg'
categories:
- dotnet
tags:
- dotnet
- csharp
slug: equals-and-equals-are-not-equal
---

## Intro

By now (almost 20 years since .NET was released) this topic has been discussed a looooooot, but still, sometimes, I feel like it's forgotten, so there's probably no harm in talking about it again.

Java makes it a bit simpler in this regard, because it doesn't allow for operator overload (e.g. `==`), so developers know the rule really well: `==` compares references, `equals` actually compares the objects.

Because .NET allows for operator overload, .NET developers end up using `==` and `Equals` interchangeably, but it isn't as straightforward.

## A quick example

Let's create a simple class that overrides the `Equals` method, as well as overloads the `==` operator.

```csharp
public class A
{
    private readonly int _someValue;

    public A(int someValue)
    {
        _someValue = someValue;
    }

    public override bool Equals(object other) => other is A a && _someValue == a._someValue;
    
    public static bool operator == (A left, A right)  => object.Equals(left, right);

    public static bool operator != (A left, A right)  => !object.Equals(left, right);
}
```

Now let's play around with the available comparisons.

```csharp
class Program
{
    static void Main(string[] args)
    {
        A a1 = new A(1);
        A a2 = new A(1);

        Console.WriteLine($"a1.Equals(a2): {a1.Equals(a2)}");
        Console.WriteLine($"object.Equals(a1, a2): {object.Equals(a1, a2)}");
        Console.WriteLine($"a1 == a2: {a1 == a2}");
    }
}
```

This program console output is the following:

```
a1.Equals(a2): True
object.Equals(a1, a2): True
a1 == a2: True
```

So far so good! The objects are the equal, so every comparison returns `true`.

The static `object.Equals` ends up also calling the overridden `Equals` method, the advantage of using it is to avoid getting null reference exception (or having to check for `null`s manually).

Now let's make an apparently harmless change:

```csharp
class Program
{
    static void Main(string[] args)
    {
        object a1 = new A(1); // changed the variable type from A to object
        object a2 = new A(1); // ditto

        Console.WriteLine($"a1.Equals(a2): {a1.Equals(a2)}");
        Console.WriteLine($"object.Equals(a1, a2): {object.Equals(a1, a2)}");
        Console.WriteLine($"a1 == a2: {a1 == a2}");
    }
}
```

Now, does anything change in the output?

```
a1.Equals(a2): True
object.Equals(a1, a2): True
a1 == a2: False
```

So now `==` returns `false`, not good!

Why is that?

The `Equals` method is a "normal" method defined in `object` and overridden in our `A` class, so it follows the usual polymorphism rules: what matters is the type of the object, not the type of the variable that references it.

The `==` operator is simply defined to compare `A` objects, as you can see by the types of parameters used in its definition above. This means it won't be called on non `A` variables. In this case, it ends up using the default object comparison, which means reference comparison. As the references are not the same, we get `false`.

## Another twisted example

Now let's look at a slightly more twisted example, using a built-in type, the `string`.

```csharp
class Program
{
    static void Main(string[] args)
    {
        object s1 = "s";
        object s2 = "s";
            
        Console.WriteLine($"s1 == s2: {s1 == s2}");
    }
}
```

This outputs:

```
s1 == s2: True
```

So, what's different here?

For optimization, .NET (and others) uses something called [string interning](https://en.wikipedia.org/wiki/String_interning) to keep a single copy of the each distinct value, so even though we are declaring it two times, it ends up keeping a single copy.

Due to string interning, the `==` works as desired in this situation, because the reference is actually the same.

We can however break this with a slight tweak:

```csharp
class Program
{
    static void Main(string[] args)
    {
        object s1 = "s";
        object s2 = new string(new []{'s'});
        
        Console.WriteLine($"s1 == s2: {s1 == s2}");
        Console.WriteLine($"s1.Equals(s2): {s1.Equals(s2)}");
    }
}
```

And now we get:

```
s1 == s2: False
s1.Equals(s2): True
```

So, even though most of the time, with strings, things will just work, there are situations where it won't, so better be careful.

## Outro

Wrapping up, the goal was not to say not to use the `==` operator, but keep in mind the pitfalls associated with it.

You're comparing built-in types or types you control and have the correct reference type? Sure, go ahead. Every other case, better go with `Equals`.

Thanks for stopping by, cyaz!