---
title: Changing the unchangeable
description: What if a principle doesn't apply anymore?
category: reliability
tags:
  - images
  - CI-CD
  - principles
---

## Immutability

Immutability. The certainty that a thing, once defined, never changes. This simple practice has so many implications which help improve the reliability of systems that it is often taken for granted in architectural and design decisions.

Immutable images. Images that cannot be changed? Well, not exactly, since of course they _had_ to undergo some primordial change in order to even exist!
What we mean when we say that an image is "immutable" is that an image is produced from a _declaration_ and that its final state can be _predicted_ based on that declaration.
The declaration, in my case, often comes in the form of a combination of components; a Packer template invokes Ansible roles to provision a declarative configuration, extending an initial state.

Each of these "things" -- the Packer template, the Ansible role, the initial state -- are specific, versioned and unambiguous.
We are not talking about an arbitrary initial state, we are talking about _the image versioned with this specific version, from this specific repository_[^sha2].

Immutability and Idempotency go hand in hand.
Once we have a declared state, we always generate the same resulting state, no matter how many times we perform that action. This is, unless we have side-effects, which it is wise to assume may be present.

We often insist on these principles being adopted in production systems, not so that we can know exactly what is happening, but so that we can reason about the smallest possible unit of change.
Immutability means that once the declared state has been delivered, _it stays that way_.
Any further evolution of the **system** must involve a change to the **declaration**, and we don't need to inspect the system to be able to reason about its state.
The ability to do this makes it orders of magnitude to make changes to a system with a modicum of certainty (_i.e._ predictability) about the impact of that change on the system.
No doubt, the more complex the system, the more moving parts, the more nonlinear the interactions between them, the harder it is to reason about the global impact that a little change may have, but indeed this is the best possible scenario.
Compare that with trying to predict the impact of a known change (_i.e._ pull request) on an unknown state, _i.e._ a system which has been manually changed over time.

Immutability also gives the ability to _undo_ changes to a system, to a certain extent.
When done right, declarative compositions of components are specific states, with a state machine defining transitions between them. If a transition between states

## Declarative Theatre

Often, the principle is adopted, but implemented only in a superficial way. This often happens when there is a declarative format wherein a set of imperative instructions are embedded in it.
This is exacerbated of course when there are nondeterministic steps in those instructions, which may depend -- mostly in explicit, but often in subtle, implicit ways -- on the _iteration_ or _conditions_ under which the new desired state is provisioned.

Another way which the principle can be broken more subtly is when the declaration of the state becomes to large to comprehend.
When a declaration becomes thousands of lines of code, or has a very large cyclomatic complexity.
If a human can't read the declaration and reason directly about the end state which is declared, it may as well be a black box operator which one must then simply trust does what it says on the box.
The obvious and all too common scenarios which follow are that the box is opaque, contains no description of what it does, and indeed does different things over time without updating its description.

---

## References

[^sha2]: Of course, the image version is often referred to by an image _tag_ and unless that tag itself is immutable, we are introducing some ambiguity into the system. There is no formal difference between the tag `latest` and `v1.2.0-build-4`, although there is clearly a _semantic_ difference. If we really want to have declarative certainty, we need to refer to images by their SHA2 hash.
