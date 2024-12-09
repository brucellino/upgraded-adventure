---
---

## How do you build a test automation platform


### How do we express our tests

Robot Framework is a set of libraries and tools used to write test cases in human readable language.
It allows us to translate compliance, functional, and end-to-end requirements into specific test cases which can be executed and seen to either pass or fail, providing an understanding of the state of our system.
Although Robot Framework does not cover all of our needs for testing out of the gate, the libraries available in the ecosystem -- in particular the browser library -- cover more than 80% as a rough estimate.
Further coverage of functionality is provided by custom libraries, resting on the large ecosystem available in python.

Some tests are unwieldy to write in the human readable Robot Framework DSL and are more elegantly expressed in plain python with Pytest. 

The one main test category which is not well dealt with by Robot are the nonfunctional performance tests.
These are written in Javascript for K6.