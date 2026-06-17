---
title: Design Doodle - Test Automation
layout: post
description: What does clean design for test automation look like
category: architecture
toc: true
tags:
  - blog
  - doodle
  - design
  - test-automation
  - functional-testing
  - performance-testing
  - cloud-events
mermaid: true
---

- [How do you build a test automation platform](#how-do-you-build-a-test-automation-platform)
  - [Loops and Feedback](#loops-and-feedback)
  - [Expressing tests](#expressing-tests)
  - [Executing tests](#executing-tests)
    - [Execution Runtime](#execution-runtime)
    - [Triggering Execution](#triggering-execution)
- [Reporting tests](#reporting-tests)

## How do you build a test automation platform

Consider the following scenario:

We have a distributed system which is nominally providing a set of integrated functions, but is in reality delivered independently by loosely-co-ordinated teams.

Let's call this system "the platform".

We know it's not a platform yet, but that's where we want it to evolve towards, as we collect user feedback, and iterate the implementation.

### Loops and Feedback

At the heart of testing is the concept of **feedback**.
This is an oft-overloaded term, so we emphasise that we mean feedback in the _biological_ sense, not the _eletronic_ sense:

> The modification, adjustment, or control of a process or system (as a social situation or a biological mechanism) by a result or effect of the process, esp. by a difference between a desired and an actual result;
> <br>-- <a href="https://www.oed.com/dictionary/feedback_n?tab=meaning_and_use#4568120"><small>Oxford English Dictionary</small></a>

The system is our deployed software; the modification is a "change" to the system, and the feedback is the effect of the change, obtained by means of observation of the system.

<center class="mermaid">
flowchart LR
  Change -->|Apply| System -->|Modify| Observe -->|Evaluate| Change
</center>

Feedback implies some form of measurement, but before we get to _how much_ or _if_ (some attribute), we should be a bit more explicit about **what** we are measuring[^entities].

In our case, we are measuring attributes of something we do to the system.
What should we measure then?
This is perhaps the hardest part of testing -- what should we invest time and effort into making sure it is as we expect it to be?

Hold on to your hat, because we are about to bump up against an axiom of software: **It exists to satisfy its users**.
Corollary: If it's not satisfying users, it may as well not exist.

Yes, this is not a fundamental fact of software; indeed I doubt such a thing event exists!
However, if everything is "just like, your opinion man" then we cannot make progress, so for us the user is the entire reason we write anything.

The entities therefore should be things which have valence in _user's_ **_experience_**.

**Let's start with the most basic entity: the function of the software.**

Now it becomes possible to enumrate functions, and to assgin them attributes like "does this work y/n" or "how well does this work".

### Expressing tests

Now that we have some direction on what to write tests _for_, we can decide on tools that will be used to implement these tests in software.
The system we are testing is exposed via the web, so our users will, be mostly using browsers to use it.
Therefore, we will need to express tests that can be run via the browser.
We use Robot Framework for this, since it is the easiest to write something close to human language, comprehensible to business and developers alike.

Robot Framework is a set of libraries and tools used to write test cases in human readable language.
It allows us to translate compliance, functional, and end-to-end requirements into specific test cases which can be executed and seen to either pass or fail, providing an understanding of the state of our system.
Although Robot Framework does not cover all of our needs for testing out of the gate, the libraries available in the ecosystem -- in particular the browser library -- cover more than 80% as a rough estimate.
Further coverage of functionality is provided by custom libraries, resting on the large ecosystem available in python.

Some tests are unwieldy to write in the human readable Robot Framework DSL and are more elegantly expressed in plain python with Pytest.

The one main test category which is not well dealt with by Robot are the nonfunctional performance tests.
These are written in Javascript for K6.

### Executing tests

Expressing the tests is only one of the steps required to create the feedback loop -- the tests must also be actually executable.

#### Execution Runtime

These two tools, Robot Framework and K6, imply two necessary runtimes: python and NodeJS respectively.
Aside from the runtime, we need to provision necessary configuraiton and data.
Note that we are not specifying up-front where exactly the tests will be executed.
This means that we should design the execution environment as something that can be privisioned on arbitrary infrastructure -- or at least a few specific implementations with necessary tools for others.

#### Triggering Execution

## Reporting tests

---

[^entities]: I have spent an inordinate amount of time trying to find the "correct" set of entities to measure.
