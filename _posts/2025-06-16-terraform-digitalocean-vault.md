---
title: A terraform module for Vault on DigitalOcean
layout: post
description: Architecting for clean cloud deployment
category: architecture
toc: true
tags:
  - blog
  - terraform
  - hashicorp
  - vault
mermaid: true
---
* TOC
{:toc}

## Vault in a Managed Platform

This is a short writeup of the Terraform module I wrote for a Vault deployment.
I had to solve a few technical challenges and take a few decisions during the development process, so I wanted to write them down for posterity.

### Solution Context

There are plenty of ways to deploy Vault, but this one has a few considerations which you may find yourself identifying with.
The overall context is that this service (Vault) is a component of an internal developer platform which supports the delivery of workloads to downstream users.
I am writing as the platform engineer who is responsible for delivering the platform as a whole.

I followed the [Hashicorp Well-Architected Framework](https://developer.hashicorp.com/well-architected-framework/zero-trust-security/raft-reference-architecture) for Vault, applying this to the Digital Ocean platform.

Another important design decision here was that I consider Vault to the be the first component of a platform, with zero dependencies.
This means that, at the time of deployment, we have no knowledge of any other services which may enhance the operations, such as service discovery, or observability.
These will indeed come later, as part of the platform's iterative deployment, but it is important to explicitly exclude them from our scope since we want to make a clean Terraform module.

There are a few questionable decisions which I have made during the design of this module, specifically the choice of Tailscale as an overlay network, and the use of self-signed certificates for TLS.
I will justify these and perhaps discuss how changes to the module may make them optional.

## One Goal -- clean architectural interfaces

I wanted to see if I could reduce the goal of this module to a single concern, _i.e._ it should _only_ deploy Vault.
Nothing else should be contained within it -- if anything else is needed, it should be expressed as input variables.
Likewise, the module should be amenable to anything that depends on it upstream -- it should produce outputs which are consumable by other modules which need to build on it.
Finally, the module should deploy Vault with zero opinions about what Vault should do.
There should be no internal configuration for secret stores, authentication mechanisms, or other configuration.
This will vary greatly from team to team and use case to use case, so should be left up to the actual users of the service to define.
The overall picture then is of a production-ready, workload and userbase agnostic Vault deployment which can be customised by whoever requests it.
This will make the module better for use in a **self-service** environment, such as the internal developer platform we are considering in our context.

### Basic dependencies and design decisions

Although we declare zero dependency on other services, we do indeed need some bedrock on which to deploy our Vault.
These are our _design dependencies_.
If you find yourself disagreeing with the need for these, then this module is not for you.
I find it important however to explicitly state these dependencies so that these divergences in expectations can be immediately identified.
Rather than designing a module that is generic and satisfies any deployment scenario, I wanted to develop separate modules for specific scenarios.
It is perhaps true that there will be some redundancy and repetition across these modules, but this is done consciously in order to maintain clean interfaces.

The dependencies of this module are:

- A virtual private network (VPC) which we can use to isolate our Vault deployment from other platform services, or managed workloads.
- A Digital Ocean project, so that we can add resources to it and manage them.
- A tailscale organisation which we join droplets to.
- A pre-existing Vault cluster which contains the secrets necessary to initialise the providers of the dependencies above.

That last one might seem redundant

## Future improvements

### Loadbalancer

- Add cert and load balancer


### Cross-region deployment

- Create VPCs, with peering, across 3 regions and deploy a vault instance into each of them.
- Create separate clusters in regions, and enable WAN federation.


### Domain Records
