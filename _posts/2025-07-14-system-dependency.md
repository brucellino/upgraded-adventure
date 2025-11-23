---
title: Managing dependencies in a complex system
layout: post
description: Tools and methods for maintaining risk posture and managing exposure in a system with thousands of dependencies.
category: methods
toc: true
tags:
  - blog
  - platform-engineering
  - dependency-management
  - risk-management
mermaid: true
---

* TOC
{:toc}

## Problem Statement

In this post, we'll tackle a common problem faced by those responsible for shipping software for a client: ensuring that software supply chain is kept secure and compliant with requirements of the contract.


## Setting the stage

Let's say you're building a set of software services for a client, which takes the form of a product composed of multiple services, managed on behalf of the client.
The goal of this set of services is to provide access to a defined set of cloud services, data sets and other APIs, all of which are also part of your product.
Now, let's say that the client intends to release this product for use by third parties and in order to do that, they need to state with certainty that the product is secure and compliant with all relevant regulations and standards.

It is safe to assume that each of these services is implemented directly using one or another programming language, which inevitably depend on a set of libraries or frameworks.
These direct dependencies may themselves also depend on other libraries, packages or modules.
In reality then, the system is composed not just of the specific implementation of services, but also all of this graph of dependencies.

Given this understanding of the composition of the system, of known, assumed and unknown dependencies, let us consider what it would take to keep the system safe.
This would, at a minimum, require the ability to observe the full composition of the system, as well as the presence of known vulnerabilities in the dependencies.

## Seeing Risk

Let's assume we can actually see all of the dependencies: how would we achieve that?
It would involve some tool which inspected the system and extracted the graph of all of the dependencies.
This is only feasible when the system _declares_ what it depends on.
This varies from case to case:
  - In the case of an application, this will come in the form of a language-specific manifest file, such as a `requirements.txt` for Python, a `go.mod` for Go, or a `package.json` for Node.js.
  - In the case of a service, this will come in the form of a service-specific file, such as a `Dockerfile` a Docker Compose file for Docker, a manifest file or Helm chart for Kubernetes, or the providers section of a Terraform configuration.
  - In the case of a virtual or bare-metal machine, this will come from interrogating the filesystem for installed packages or modules.
