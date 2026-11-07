---
title: Semantic Release Plus ça change
description: Method still good, tooling even better, documentation barf
layout: post
category: Tooling
tags:
  - tooling
  - methods
  - semantic release
  - release engineering
  - blog
toc: true
---

Since my earliest days at UEFA, where versions and dependencies mattered, I realised that one could not be sentimental about releasing artifacts.

Several teams working on independent projects with dozens of artifacts had to be able to work at their own pace, uncoupled from both their upstream and downstream dependencies.

Since _our_ team was responsible for the overall deployment of products into the platform, we made it policy that all artifacts should have a _semantic version_ associated with them.
As an overall policy it was a good start, but the next step was to put tooling in place so that it releasing artifacts with semantic versions became the easiest and therefore default way of doing things.

## Semantic Release and Agency

I write from the perspective of a "supporter".
Call the role what you want -- DevOps engineer, SRE, Platform Engineer, _etc_ -- I have never been the developer, nor the product owner, but always something in the background whose ambition is to make them go faster, smoother, safer, better.
The first principle then is "get out of the way", which is very hard to do when you are by nature intruding on people's behaviour, in this case aa beloved release process.
We nevertheless needed a way to slot in a predefined[^read-declarative] method to perform tasks that help the developer to act safely, in accordance with the guardrails and stay on the "golden path" for whatever they are doing.

Having semantic releases of software is one of these guardrails.
It is a way of saying "I made this, it consists of well-known changes of predictable effect to the user's experience, and one day I want to let the user actually use it".

Did you, dear reader, notice that I have drawn a line between "releasing" and "deploying"[^quotes]?
This harks back to the concept of "You build it, you run it, you own it".
We provide the platform, dear workload owner, but _you_ are in charge of deciding what, when and to whom you want to deliver your releases.
This approach give the workload owner agency.

To reinforce the concept of what we are actually doing here as those "supporters" I identified myself with above -- we have a responsibility to give you a reliable means to know what you are deploying, and decide based on that knowledge when and what to deploy.

Condone my preamble, dear reader, it was not intended for you, but rather for me to gather my thoughts, indeed to recognise that their same formation and nature.
Once I have found a solution I deem elegant my attention turns to the underlying reasons for _why_ I may deem it so, less as a justification for the opinion and more for the comprehension thereof.

We will proceed now with the technical bits.

## No Change, a big change, a small change

The change I am referring to is a change in tooling.
It consists of replacing one tool with another, in order to perform the same thing better: Semantic Releases of software.

The tooling _being replaced_ is [Semantic Release](https://semantic-release.org/).
It is being replaced with [SemRel](https://www.semrel.io/).

The principle of getting out of the way of the developer, I hope, is maintained, meaning that to the developer and indeed the workload owner, there is no change.
The same rules apply for determining what kind of release will be made, the semantics are the same, their behaviour is not expected to change, the same reliable process is invoked.

### Actual changes

The small change is made on _my_ side[^my-side].

I have decided to make this change[^change-where] because I want to pay down the "NodeJS debt", _i.e._ I prefer having executables rather than a runtime and dependencies.

These two tools -- Semantic Release and SemRel -- perform the same task: they perform semantic releases according to a declared method, but in the former case I need to pull in a NodeJS ecosystem of packages and their nested dependencies, while in the latter I need to bring in only a few executable binaries.
I have used Semantic Release since UEFA days, circa 2019.
It is a good tool, there is no objective reason not to use it, so I reiterate that my choice is based on personal aesthetics and considerations.
It was well-documented, had plugins necessary for the tasks I needed, and was fairly stable.
I could write a `.releaserc` in YAML or JSON, slot it into a global configuration state, push it out to all repositories under our governance, and trust that those projects would be semantically released in our platform, according to the governed integration pipelines[^governed-ci].
The rules therein declared the rules for release, almost always relying on the commit message convention, branch rules and where release artifacts were to be delivered to.
The main difference from workload to workload was which assets needed to be included in the release, but even here we managed to come up with patterns on a language or ecosystem basis that generally captured what should be included in the release.

SemRel does the same thing, with a slightly different configuration and tooling.

SemRel also relies on plugins to perform the specific sub-tasks of release engineering, with similar configuration syntax, in similar format (YAML or JSON).

The only deficiency SemRel presents is that I cannot yet assign custom labels to release artifacts in the GitHub release plugin.



### The Big Change

I rely on one or the other of these tools.
I need to be able to trust them, but unfortunately that trust is coming to an end.
The big change is that the Big Slop has come to both of them, starting as is recently my experience, with the documentation.

#### SemRel docuslop

SemRel's documentation has the sloppy smell of throwing an incoherent grid of infoboxes in my face right on the front page.
I _read_ documentation -- I cannot perceive any other reason for documentation to exist other than _to be read_! -- and the front page is an unreadable jumble of excitable noise; it defeats its own purpose.
A further sign that this documentation has been **generated** rather than **written** is the pattern of placing a single link to the _actual_ documentation tree where I can start to reason about it, find what I need to know, and generally _use_ the documentation.
Is this a side effect of the use of Astro[^semrel-docs]?
Is it a direct effect of the use of Copilot and Claude?
Let me point out that **every single commit** of the documentation repository is [co-authored by one human and a combination of their bots](https://github.com/SemRels/semrel-docs/graphs/contributors?from=8%2F1%2F2026).
Now, I empathise with you, whoever you are [`mwaldheim`](https://github.com/mwaldheim), I too have had to write a bunch of docs all by myself, but I cannot overcome the repulsion I feel when reading what has been generated.
I cannot find it in me to trust what is written, I find myself in a perpetual state of skepticism when reading anything in it.
This documentation hurts me, insults the fact that _I_ am investing time in reading it, but _you_ did not invest time in writing it.

I am left with a feeling of profound despondency.

Why did you do this?
The documentation could have just as well been written by your own hands, by your own brain!
Why did you delegate this to something so untrustworthy?
Could you not have told us, through your own craft, that you cared about this, that we could trust that it was your sincere intent to create _this_ documentation, and not whatever was spilled out of the prediction machine?
I am discouraged now to actually read your documentation, I return to it only to find those tiny hints of specification or configuration that I need to make your tool work.

And as I have taken pains to write above, I really actually do want to use this thing!

I'm not even saying don't generate anything! 
Generate the predictable bits; we have been doing this for decades, however it was still something that didn't insult the reader's intelligence.

I am saying use the tools, but please write **something** yourself.

#### Is it everywhere?

On a closing note, I see that Semantic Release [moved from GitBook to Astro](https://github.com/semantic-release/docs/pull/2).
Do I care about this?
Do I really care what they use to build their documentation?
The answer to that direct question is "No. I cannot presume to impose on your choice".
However, there are implications.
Astro is new, LLMs are well-versed in it, there will be an seductive temptation to "improve" things by "rewriting" content now that we have this pliable, conducive new container to throw our slop into.

I am disheartened to [see that their layout](https://semantic-release.org/) now has the same sloppy tell of a single "Get Started"[^already-started] on the front page, nothing in the nav, gives the impression of trying to _convince_ me rather than _inform_ me.

So far there doesn't seem to have been explicit infection of the documentation from the slop machine, but I've seen enough of these tells to suspect that the it's even worse this way: the author has co-opted the patterns and does not discriminate any more in terms of quality.

[Olabode](https://github.com/babblebey), I don't know you.
I applaud your work to change the Semantic Release documentaition.
I suspect you did it of your own volition and from a fount of optimism, desirous of making the world a slightly better place.
I implore you, though you will never hear it, please, write bad prose if you must, but write it yourself.

Documentation is for the reader.
The reader is me.
I am a human.
I read things in order to understand and remember them. 
Forcing me to read things you did not write is an insult to my intelligence, and a stain on your craft.

## Closing 

There, I said it.
I'm going to use SemRel instead of Semantic Release, but I'm very disappointed in whoever is writing their documentation.

----

# Footnotes and References

[^read-declarative]: Read: "declarative" process. Something which we owned outside of the developer tooling, which could read a declaration of configuration and reliably be executed by our tooling to reach a given, predictable outcome. In this case of course the creation of a new semantic release.
[^quotes]: You might notice that I am using a lot of quotation marks in this post, around terms which you, dear reader, might consider well-defined and therefore unnecessary. I am not so sure, towards the end of 2026, that we all share that common understanding of these terms, so I prefer to imbue them with a level of ambiguity that I suspect represents the divergence of what they might mean to you and I.
[^change-where]: What is implied here is that I have decided to progressively roll out this change opportunistically across projects I own or manage. It will take a while.
[^my-side]: A reminder that my side is the invisible side, the side that works continuously to make your side better, faster, more reliable, more comfortable. I am the DevOps, I am the SRE, I am the gardener, I am the caretaker, I watch over the environment and work to ensure that every day you, dear developer, dear workload owner, awake to build whatever it is you're making for the users, you find an environment that is invites you in and makes you effective.
[^governed-ci]: What I mean here about "governed integration" is that there should be a part of the continuous integration pipeline that is owned by the platform team, where we define where and how release is performed. Teams get this "for free", meaning it's not optional, it's your safety belt and you have to wear it. This was usually done by declaring Terraform resources in a GitHub org and enforcing policy.
[^semrel-docs]: See https://github.com/SemRels/semrel-docs
[^get-started]: Bitch, I have been using this since 2019, I do not need to "Get Started", I need to go directly to where I need the information your docs provide. Why are you writing exclusively for the first-time user?! Put something else in your header.
