---
title: We are all platform engineers now
description: Commentary on an inevitability
category: PlatformOps
tags:
  - blog
  - platformops
---

## We are all platform engineers now

Platform Engineering is clearly the hotness amongst the cool kids, and has been so since at least early 2022.
As with all the good ideas that eventually end up mangled by the IT industry, Platform Ops has both been around for a longer time than I bet most folks think, and also doesn't actually express anything about its nature.

Terms like these are useful for driving the marketing department, naming products and generally starting useful conversations.
It's an opener, but doesn't actually get you all the way to actually solving a problem.

One thing I am convinced by is that these terms arise because there really are problems worth solving, and perhaps since the birth the term "Agile", these are ever more business problems rather technological problems.
With nigh infinite compute capacity, the questions then became "how the heck do we get things done now?".

In this vein, the exhortation to collaborate in DevOps movement, the codification of good practices for reliability in SRE and the realisation in DevSecOps that collaboration with the security and safety side of business gave greater benefits if included early, finds a continuation in this pattern of "Platform".

It's hard to tell from within my little bubble, but the term "Platform Engineering" seems to have an irresistible attraction in 2023.
There will certainly be "haters" -- contrarian folks who find their intelligence or experience insulted by the very idea that the knowledge and skill contained within them can be packaged into a _product_ (ugh).

> "Where's the workmanship in that?"

they will scowl and who can blame them?
Our market is swimming in terrible products which only exist to make their vendors a dime.

> "I can do that in a weekend!"

they will counter, and who can blame them?
Most of the products we're seeing come out are just compositions of other things.

> "There's no way a product, no matter how customisable and extensible, will be able to meet all use cases"

is the line they will draw in the sand, and who can blame them?

Well, yes, these are all good points.
I've been involved in building things we called "platforms" for over 10 years and I can say for sure that all the things people are talking about in this wave, we've been trying to build in one way or another for that whole time.

We didn't ever call them "internal developer platforms" when we were building science gateways back in 2013, we definitely didn't treat the worldwide compute grid as a product back in 2003, but these were definitely platforms. You want something else? Build it your own damn self.

So, of course people did -- frameworks under toolkits under apps, all in such a delirious state of entropy that nobody could accurately predict what would be around.
Remember GIFEE[^google]? Yeah, I almost forgot about that too, but luckily I was around in 2016 and I've seen enough failed experiments know why spidey sense is tingling now that we're making a hotness.

## What's changed

The OG platforms (including early AWS) were born to serve the entire damn planet and were by necessity inflexible. You get these functions, that's it. Everybody gets the same.
Self-service? yes, you can consume these very specific things, and we'll bill you for them.
We had effort after effort to build a better piece of infrastructure, to expose more specific functions, to bring in new capabilities.
I'll bet that a it's a commonly-held opinion that all of this converged in the creation of Kubernetes: the codification of all conceivable functions in a data center, in a network, and between applications.

This didn't solve any damn problems -- it created more of them!

What's changed since the first iterations of platform is that we are now coming to terms with Spidey Rules: with great power comes great responsibility.

Now, I've been around, but I haven't been to every business out there, I don't know the story of your struggles, I'm not going to pretend I do.
But I'm willing to bet that many organisations over a certain age have found themselves mired in transition due to the inability to make decisions responsibly across the lifecycle of the application.
Each individual group or function chose "the right tools for the job", perhaps doing their best to optimise locally[^or_not]

## What exactly are we talking about

I find it better to my taste to be explicit when talking about a subject, so as to help myself understand the boundaries of my own knowledge, and better hone my own opinions.
For this reason, I tend to think about -- and hence reason about -- Platform Engineering and PlatformOps differently.

---
[^google]: Did you just google that? Did you have to go to page 2 to feel the same irony I'm feeling? In 2016 had a hashtag and drive IBM to adopt the cloud model, if you believe the things you read on page 2.
[^or_not]: This is the generous point of view. A more realistic one is that choices were driven by vendors and personal connections at every level.