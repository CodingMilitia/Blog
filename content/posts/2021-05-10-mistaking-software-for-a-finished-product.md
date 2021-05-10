---
author: João Antunes
date: 2021-05-10 18:30:00+01:00
layout: post
title: "Mistaking software for a finished product"
summary: "As much as it would be simpler, software isn't something we can ship and forget about, or even just keep adding features without revisiting architecture and design."
images:
- '/assets/2021/05/10/mistaking-software-for-a-finished-product.png'
categories:
- smalltalk
tags:
- smalltalk
slug: mistaking-software-for-a-finished-product
---

## Intro

The common thought of treating a non-trivial piece of software as finished - be it an application, library, framework or even operating system - is something that has been on my mind to write about for a long time, but I've been procrastinating... until now!

This is a subject that seriously annoys me, even more when coming from people who should know better, particularly folks working in the software space itself.

## General perception

Before getting into the technology industry itself, let's start by looking at the overall picture.

It seems that the general perception people have of software, is that it's just like other things we can buy: it's done, use it while it works. Sadly, that's not at all how things work.

As probably everyone that is reading this blog knows, software needs to be updated, not only to get new features, which for some who are happy with what they have is irrelevant, but more importantly, to fix bugs and security issues.

We can easily find evidence of this disregard for the importance of keeping things up to date just by looking at the operating systems being used in the wild. I regularly bump into computers, for example when going into stores or paying attention at news pieces on the TV, running Windows 7 or even Windows XP, being both, as you might be fully aware, way past their supported lifetime.

Another flagrant example are (most) Android phones. Android OEMs, particularly when selling low-end devices, provide very limited support for their software, if at all. As folks in general are oblivious to why this is far from acceptable, the devices keep getting bought, so OEMs don't see a reason to change their ways.

Even if on one hand I can concede that this whole "software is never finished" subject is not the easiest thing for non-technical folks to understand, mainstream computing devices with internet connectivity are not something new (what are we, 20 years in? more?), so it feels like this should be better understood by now. Something like everyone knowing that the car needs to undergo regular maintenance 🙂.

## Applied to software development teams and companies

Now, the general perception annoys me, but what really gets my blood boiling, is when people that actively work in the software development industry have a similar mindset, although in a different context.

More than once, I have been told things like "yeah, architecture is important, but that's just in the beginning of the project, then it's done and you just add features".

If this was something a business stakeholder said, which is focused on the business and not super into the technology stuff, I'd be just like, yeah, that's not how it works, but ok. But when I hear this from folks who manage technical teams, then it's a whole other level of annoyance.

This mindset is one of the many reasons software projects and codebases are, well, bad! (Note that I'm not excusing developers from all the crap we do 😛.)

With time, new requirements, new bugs and new challenges are found, so software needs to be always evolving. Sometimes small adjustments will do the trick, but many times, larger restructurings are needed, eventually including complete component replacements.

Architecture and design need to be constantly analyzed and evolved. The alternative is to just keep pilling more and more crap on top of the project, making it increasingly harder to add features and maintain. This lack of continuous evolution can eventually lead to the need to rewrite the whole thing, which is likely far riskier and more expensive than if things kept being evolved in the first place. The cost is further increased if we take into account the features and bug fixes that could have been implemented more efficiently if the codebase was in a better state.

On a related note, the same train of thought should be applied to the tools and technologies we use. As much as it's tempting to not update, for example, the underlying framework being used, as some given version doesn't bring anything relevant for the project or it's a bit of work, delaying for too long might come back to bite us at some point. I'm not suggesting that we should update everything without giving it a thought (although I'm a fan of being on the bleeding edge 😅), but if we keep postponing, we might get to a point where the update is much more complex and expensive than if we had done it along the way (you might notice a pattern here 😉).

Maybe all of this is exacerbated by the continuously increasing pace of evolution in the software industry, which wasn't like this not too long ago (insert joke about the amount of JS frameworks 🤪). Regardless, we need to adapt to the times, technologies evolve faster, users expect better, we can't just smile and wave.

## What about the agile mindset?

It's hip for companies to preach their agile mindsets, that you should use Scrum and have all these cerimonies. Where does all this dynamic go when talking about maintaining a project in the long run? Clearly, out the window! 😂

Sometimes it feels folks get lost in wanting to apply these agile methodologies, ceremonies and tools (yes, I'm looking at you and your Jira), instead of focusing on actually building quality software in an agile way, following DevOps practices and all that jazz, but maybe this is a rant for another time 🙂.

For me, the main selling point of the agile mindset is iterative and incremental approach, but this shouldn’t be just applied when we're building things for the first time, nor just to new features. We need to incrementally improve things, even if the value of these improvements isn't immediately apparent from a user perspective.

## Outro

Ok, this post was a bit of a rant, but I had it in me for sometime and it had to come out 😅.

On the general perception topic, maybe we (techies) need to do a better job educating people that software is unlike other types of products, it's more of a "living" thing that evolves over time.

As for the specifics of software development teams and companies, it's sad that something so fundamental to the success of projects and products is still not blatantly obvious to everyone involved. Guess it's another front we need to work harder to get the point across.

Thanks for stopping by, cyaz!