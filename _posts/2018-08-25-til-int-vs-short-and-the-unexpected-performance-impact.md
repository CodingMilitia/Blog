---
author: João Antunes
date: 2018-08-25 12:30:00+01:00
layout: post
title: '[TIL] int vs short and the (unexpected) performance impact'
summary: 'Quick little post on a probably unexpected performance difference when using an int instead of a short in C#'
image: '/assets/2018/08/25/int-vs-short-post-image.jpg'
categories:
- dotnet
- til
tags:
- dotnet
- til
- todayilearned
---

[![int vs short](/assets/2018/08/25/int-vs-short-post-image.jpg)](/assets/2018/08/25/int-vs-short-post-image.jpg)

One of these days, going through some code at work I noticed a colleague had made a for loop explicitly defining the iteration variable as a `short` (which is an `Int16`) instead of an `int` (or `Int32`) as usual (or in my case `var`, which ends up being an `int` anyway).

Something like the following, just for context:

{% gist 3bf6e03a764493e25ece78b5b51d35a2 %}

After seeing this I thought, yeah, he's right, we should be more careful and use the types we really need. If we're iterating on a short range of values, why not use `short` (or a even smaller type)? It may even have a (probably almost unnoticeable) gain in performance as we're saving memory right? Oh how wrong was I 😇

I began to wonder, why is it so unusual for us to see `short`s used in this context, and in the spirit of finding excuses for doing anything but what one's supposed to do, I went on searching for an answer (you can experience this massive search engine endeavor [here](http://lmgtfy.com/?q=c%23+why+use+int+vs+short+in+for)).

I ended up in [this MSDN question](https://social.msdn.microsoft.com/Forums/en-US/839b9948-4fc7-471e-8ad6-f8473e2cc317/why-do-we-always-use-int-why-not-short?forum=csharplanguage) from 2006 that has answers for this.

TLDR:
> It's a performance thing.  A CPU works more efficient when the data with equals to the native CPU register width. This applies indirect to .NET code as well.

> In most cases using int in a loop is more efficient than using short. My simple tests showed a performance gain of ~10% when using int.

Some more link clicking took me to Stack Overflow and [this question](https://stackoverflow.com/questions/1097467/why-should-i-use-int-instead-of-a-byte-or-short-in-c-sharp) with some more explanations if you're interested in checking it out.

Of course I wanted to check it out for myself and rolled a kind of stupid benchmark just for fun (using the awesome [BenchmarkDotNet](https://benchmarkdotnet.org) library).

{% gist 5b87572b4a44bf711e4384795e9db611 %}

I even added a `long` (`Int64`) to the mix just to see the result, which was:

|        Method |      Mean |     Error |    StdDev | Scaled | ScaledSD | Allocated |
|-------------- |----------:|----------:|----------:|-------:|---------:|----------:|
|   ForUsingInt |  9.601 us | 0.1850 us | 0.2594 us |   1.00 |     0.00 |       0 B |
| ForUsingShort | 17.963 us | 0.3835 us | 0.4565 us |   1.87 |     0.07 |       0 B |
|  ForUsingLong |  9.112 us | 0.2013 us | 0.1977 us |   0.95 |     0.03 |       0 B |

So, as expected given the previous explanations, the `int` outperforms the `short`. The `long` keeps up with the `int` (maybe because I'm on a 64 bit machine, so a `long` is of native register size?). I would take this performance differences with a grain of salt, as in the real world we don't have an empty `for` (hopefully), but yup, there is a difference.

If you want to take the benchmark and dig around a little more, you can get it from [this repo](https://github.com/joaofbantunes/IntVsShortBenchmark).

**NOTE:** if I messed up on the benchmark by not taking something into consideration, please let me know.

And with this I guess I'm starting a new category of posts here in the blog - TIL (Today I Learned). I think it could be a fun little category, with smallish posts with quick bits of interesting stuff I find along the days (or someone points my attention to).

Thanks for stopping by, cyaz!