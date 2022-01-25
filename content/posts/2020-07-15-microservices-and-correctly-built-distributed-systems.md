---
author: João Antunes
date: 2020-07-15 18:30:00+01:00
layout: post
title: "Microservices and correctly built distributed systems"
summary: "Microservices are all the rage for some time now, but are we using the right foundations to create them? In this post, we'll look into a common design issue in distributed systems."
images:
- '/images/2020/07/15/microservices-and-correctly-built-distributed-systems.jpg'
categories:
- smalltalk
tags:
- microservices
- distributed systems
slug: microservices-and-correctly-built-distributed-systems
---

## Intro

One of these days I was watching a [talk about microservices](https://www.youtube.com/watch?v=7uvK4WInq6k) and found it really interesting (and kind of funny) when the presenter showed InfoQ's architecture and design trends graph, focusing on two things: "microservices" and "correctly built distributed systems".

![/assets/2020/07/15/infoq-architecture-and-design-2020-q2.png](/assets/2020/07/15/infoq-architecture-and-design-2020-q2.png)

(source: [https://www.infoq.com/articles/architecture-trends-2020/](https://www.infoq.com/articles/architecture-trends-2020/))

Looking for these two topics on the graph, we notice that "microservices" show up in the "late majority" section, while "correctly built distributed systems" appear in the "early adopters" section. Even if it maybe wasn't the goal of the authors, I found this amusing, as I'd expect "correctly built distributed systems" to be a pre-requisite to microservices, but alas, it seems it's not what actually happens.

## What's so funny

As I noticed this interesting tidbit, I grabbed the image and shared with some colleagues on the chat. This caused the start of a conversation: what are "correctly built distributed systems" and why aren't we building them?

My initial reaction was, for starters, we need to stop coding distributed systems as if we were building completely self-contained applications, particularly when it comes to interaction between services.

This certainly isn't the only thing I've found lacking in these kinds of projects, but it's such a foundational subject, that it's the first thing that comes to mind when these discussions start.

## Self-contained vs distributed practices

So, what is it about the way things are usually done that make them problematic in the context of distributed systems?

To exemplify, I'll borrow from Jimmy Bogard's awesome ["Six Little Lines of Fail" presentation](https://www.youtube.com/watch?v=VvUdvte1V3s) (highly recommended!).

Note the following code:

```csharp
public ActionResult ProcessPayment(CartModel model)
{
    var customer = db.GetCustomerById(model.CustomerId);
    var order = CreateOrder(customer, model); // creates and adds the order to the database
    paymentService.PostPayment(order);
    emailService.SendPaymentSuccessEmail(order);
    eventBus.Publish(new OrderCreatedEvent { Id = order.Id });
    return RedirectToAction("Success");
}

// assume a database transaction surrounds the code above
```

(code based on Jimmy's presentation example, but slightly adapted for clarity)

Here we have a (C#) method to handle the final submission of an order in an e-commerce application. Briefly looking at the code it seems pretty nice and clean, only six lines and all of them are pretty readable, we can figure out what's going on rather quickly.

Now let's take a look at it again, remembering that we're in the context of a distributed system with:

- the application this code belongs to
- a database where the application stores its information
- an external payment service
- a service to send emails to users
- an event bus, used to broadcast events to interested services

With this fresh in our mind, are issues more apparent?

Let's briefly look at some of the possible issues. Keep one thing in mind though: when there are service interactions, it's not a question of **if**, but rather **when** will is a failure occur.

### Payment service failure

Imagine the above code is running and there is a failure when invoking the payment service. Shouldn't be a big problem, as the failure would cause an exception, the transaction would be rolled back and everything would be consistent (although the customer probably wouldn't be very happy).

Now imagine a slightly different scenario, where for example the payment service call times out. The same would happen, an exception would abort things. But just because we got a timeout, it doesn't mean that things didn't continue running on the payment service side, being the customer's credit card actually charged.

This is a much bigger issue, as I'm pretty sure the customer won't be amused with being charged without actually getting the order. Things can get even worse, if the customer retries and the proper checks aren't in place, resulting in being charged multiple times.

When these failures happen, the order isn't created, as the transaction wasn't rolled back, so we better have logging in place, otherwise we won't even have the slightest information of what happened.

### Other failures

We could continue to think about other things that can go wrong with the payment service, but let's skip ahead and check out other possibilities.

What about if the email service fails? Again, everything is rolled back minus the payment, which happened and now we have no record of it.

Next line: publishing the event bus fails. Well, again the same problem as before, even worse due to the fact that an email was sent informing the customer that everything was ok.

Finishing up, what if the transaction commit, the last thing to do, fails? Again, it builds on the previous issues. Credit card was charged, an email was sent, an event was published, leading other services to believe an order was actually created, but looking at the local database, it's as if nothing happened.

By now I think you get the point, there are just too many ways things can go wrong.

## What we can do

Going back to the beginning of the conversation, what we can, or better yet, need to do, is to not code such service interactions as if we were calling methods in-process, keeping in mind things are not bound to the same transaction scope in such cases.

Other types of patterns and practices need to be used to implement reliable distributed systems. Just throwing the "latest and greatest" technologies at the problem won't solve it. The complete flow, all the interactions need to be taken into consideration and coded for.

As for the actual patterns and practices to apply to these problems, they're outside the scope of this article, which is more like a PSA style article, but needless to say, there are tons of books, articles and conference talks on the subject.

## Outro

Moral of the story: those five or six simple lines of code you have, just casually invoking multiple services? They're likely a hiccup away from messing things up.

An indispensable first step is to acknowledge these problems, not coding as if it's all good. Then, invest in understanding the problems and the patterns and practices that help tackle them.

Also, don't forget to check out Jimmy Bogard's ["Six Little Lines of Fail" presentation](https://www.youtube.com/watch?v=VvUdvte1V3s) (and others), it's really great stuff!

Thanks for stopping by, cyaz!