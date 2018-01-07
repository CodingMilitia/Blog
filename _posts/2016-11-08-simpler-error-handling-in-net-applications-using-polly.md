---
author: johnny
comments: true
date: 2016-11-08 17:10:40+00:00
layout: post
link: https://blog.codingmilitia.com/2016/11/08/simpler-error-handling-in-net-applications-using-polly/
slug: simpler-error-handling-in-net-applications-using-polly
title: Simpler error handling in .NET applications using Polly
wordpress_id: 262
categories:
- dotnet
- libraries
tags:
- dotnet
- errorhandling
- goldenshovel
---

Just think how many times you had to write some code to handle errors, and that handling required more than just logging and proceed with life as if nothing happened. If retries are required, or a cool down time, that's a good amount of logic to roll for those situations. Well, there's a NuGet for that (I guess that's our version of there's an app for that).

[Polly](https://github.com/App-vNext/Polly) is a "library that allows developers to express transient exception and fault handling policies such as Retry, Retry Forever, Wait and Retry, or Circuit Breaker in a fluent manner."

Although I've just recently came across Polly, it's been around for a while and there are a good bunch of posts about it (like [this](http://www.hanselman.com/blog/NuGetPackageOfTheWeekPollyWannaFluentlyExpressTransientExceptionHandlingPoliciesInNET.aspx) or [this](http://www.lybecker.com/blog/2013/08/07/automatic-retry-and-circuit-breaker-made-easy/)), so I'm not going to get in too much detail. Also, the _readme_ in their GitHub page is so good at showing how to use it, I don't think much more should be needed. Regardless, I rolled my own sample ([here](https://github.com/joaofbantunes/PollySample)) to try it for myself.



## Retry


Using Polly in general is really straightforward. We can express retry policies of three types: retry a number of times, retry forever and wait and retry.
(All the boomerang references are from my sample)

For instance, if we want to express a policy to retry an operation two times:
[code lang="csharp"]
var retryTwoTimesPolicy = 
	Policy
		.Handle<DivideByZeroException>()
		.RetryAsync(2, (ex, count) =>
		{
		 Console.WriteLine("Failed! Retry number {0}", count);
		 Console.WriteLine("Error was {0}", ex.GetType().Name);
		}
		);
[/code]
Then, to actually execute the code to which this policy should be enforced.
[code lang="csharp"]
retryTwoTimesPolicy.ExecuteAsync(async () =>
                {
                    await SomeOperation();
                });
[/code]
I think the code is pretty clear on what's going on, no need for much explanations. We're handling `DivideByZeroException` and telling it to retry two times. When an error occurs, the lambda passed to `RetryAsync` is called (there are some overloads, receiving different arguments). After the configured retry attempts the exception is propagated, like it would normally happen if the retry logic wasn't in place.

Retry forever is basically the same, without specifying the number of times to retry. Wait and retry may be configured directly with a collection of `TimeSpans` or a provider that may contain some logic for the creation of the `TimeSpans`.

Using wait and retry with the `TimeSpan` collection.
[code lang="csharp"]
var retryAndWaitPolicy = Policy
               .Handle<DivideByZeroException>()
               .WaitAndRetryAsync(
                   new TimeSpan[] { TimeSpan.FromSeconds(4), TimeSpan.FromSeconds(3), TimeSpan.FromSeconds(2) },
                   (ex, span) =>
                   {
                       Console.WriteLine("Failed! Waiting {0}", span);
                       Console.WriteLine("Error was {0}", ex.GetType().Name);
                   }
               );
[/code]
The amount of retries in this case depends on the length of the `TimeSpan` array.

Using wait and retry with the `TimeSpan` provider.
[code lang="csharp"]
var retryTimeSpanMap = new TimeSpan?[] { null, TimeSpan.FromSeconds(4), TimeSpan.FromSeconds(3), TimeSpan.FromSeconds(2) };
            var retryAndWaitUsingTimeSpanProviderPolicy = Policy
               .Handle<DivideByZeroException>()
               .WaitAndRetryAsync(
                   3,
                   (retryCount) => retryTimeSpanMap[retryCount].Value,
                   (ex, span) =>
                   {
                       Console.WriteLine("Failed! Waiting {0}", span);
                       Console.WriteLine("Error was {0}", ex.GetType().Name);
                   }
               );
[/code]
The first lambda passed on to `WaitAndRetryAsync` is the `TimeSpan` provider. This isn't the best logic for the provider (I'm basically replicating the array logic) but it's just an example :)


## Circuit Breaker


The [circuit breaker pattern](https://msdn.microsoft.com/en-us/library/dn589784.aspx) is used to avoid an application repeatedly trying to execute an operation that will probably fail.
Polly provides two policies to use this pattern: `CircuitBreaker` and `AdvancedCircuitBreaker`. When a circuit is broken, and until the circuit is closed again, an exception is thrown (`CircuitBrokenException`) whenever the target operation is invoked.

Circuit breaker is (as expected) simpler than the advanced circuit breaker. 
[code lang="csharp"]
var breakCircuitAfterTwoFailuresPolicy = Policy
              .Handle<DivideByZeroException>()
              .CircuitBreakerAsync(
                   2,
                   TimeSpan.FromSeconds(1),
                    (ex, span) =>
                    {
                        Console.WriteLine("Failed! Circuit open, waiting {0}", span);
                        Console.WriteLine("Error was {0}", ex.GetType().Name);
                    },
                    () => Console.WriteLine("First execution after circuit break succeeded, circuit is reset.")
                   );
[/code]
In this case, I configured the number of times the operation can fail before the circuit is open, the amount of time the circuit should remain open before allowing more attempts, a lambda to be invoked when an error occurs and a lambda that's invoked when the circuit is reset. As usual, there are overloads.

The advanced circuit breaker allows for a more (as the name gives away) advanced usage. Instead of just saying that after n errors the circuit should break, we can say something like "break the circuit if during 1 second, 75% of the operations fail, with a minimum throughput of 5 operations". You can see below how this is expressed with Polly.
[code lang="csharp"]
var breakCircuitWhen75PercentFailuresIn5ExecutionsFailuresPolicy = Policy
                .Handle<DivideByZeroException>()
                .AdvancedCircuitBreakerAsync(0.75,
                    TimeSpan.FromSeconds(1),
                    5,
                    TimeSpan.FromSeconds(1),
                    (ex, span) =>
                    {
                        Console.WriteLine("Failed! Circuit open, waiting {0}", span);
                        Console.WriteLine("Error was {0}", ex.GetType().Name);
                    },
                    () => Console.WriteLine("First execution after circuit break succeeded, circuit is reset."),
                    () => Console.WriteLine("Half open state, transitioning from open.")
                   );
[/code]
Besides the initial arguments whose usage you can infer from the description above, in this case I provided one more lambda that is invoked when the circuit breaker transitions to half-open state (when transitioning from open to closed, there's a moment when, if another error occurs, the circuit is broken again without taking the other configured rules into account).



## Wrapping up


Handling errors with a somewhat more complex logic often results in cumbersome code, mainly when we don't take the time to think about a good strategy for it. In the little time I spent trying it out, I think Polly is an awesome tool to help with this task. We get pretty simple, readable code and don't go spreading around similar error handling logic throughout our solution.

Cyaz
