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

## How do you build a test automation platform

Consider the following scenario:

We have a distributed system which is nominally providing a set of integrated functions, but is in reality delivered independently by loosely-co-ordinated teams.

Let's call this system "the platform".

We know it's not a platform yet, but that's where we want it to evolve towards, as we collect user feedback, and iterate the implementation.

### Loops and Feedback

At the heart of testing is the concept of **feedback**.
This is an oft-overloaded term, so we emphasise that we mean feedback in the _biological_ sense, not the _eletronic_ sense:

> The modification, adjustment, or control of a process or system (as a social situation or a biological mechanism) by a result or effect of the process, esp. by a difference between a desired and an actual result;
<a href="https://www.oed.com/dictionary/feedback_n?tab=meaning_and_use#4568120"><small>Oxford English Dictionary</small></a>

### Expressing tests

Robot Framework is a set of libraries and tools used to write test cases in human readable language.
It allows us to translate compliance, functional, and end-to-end requirements into specific test cases which can be executed and seen to either pass or fail, providing an understanding of the state of our system.
Although Robot Framework does not cover all of our needs for testing out of the gate, the libraries available in the ecosystem -- in particular the browser library -- cover more than 80% as a rough estimate.
Further coverage of functionality is provided by custom libraries, resting on the large ecosystem available in python.

Some tests are unwieldy to write in the human readable Robot Framework DSL and are more elegantly expressed in plain python with Pytest.

The one main test category which is not well dealt with by Robot are the nonfunctional performance tests.
These are written in Javascript for K6.

### Executing tests

Expressing the tests is only one of the steps required to create the feedback loop -- the tests must also be actually executable.

## Reporting tests
