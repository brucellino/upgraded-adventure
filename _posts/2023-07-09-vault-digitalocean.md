---
title: A Terraform Module for Vault
description: Deploying a Hashicorp Vault cluster on Digital Ocean, exposed via Cloudflare
category: platform-engineering
tags:
  - blog
  - platformops
  - unboxing
  - Terraform
  - Vault
  - DigitalOcean
  - Cloudflare
---
- [What we will be building](#what-we-will-be-building)
  - [Problem statement](#problem-statement)
- [Footnotes and References](#markdown-mermaid)

## What we will be building

[Hashicorp Vault](https://developer.hashicorp.com/vault) is, at the time of writing, one of the key components of any self-respecting platform engineer's toolbox[^platformopsnow].
Knowing how to provision and then configure a Vault cluster not only brings immediate value to the team you're working in, but also helps to understand key concepts such as auto-discovery.

The easiest way by far to deploy a production-grade Vault cluster is via the [Hashicorp Cloud Platform](https://cloud.hashicorp.com), but that hides a lot of the details which we on the other hand are quite interested in[^cc].
In this article, I want to explore one of the practices which infrastructure engineers need to master in order to build robust systems: ***composability***.
Using Terraform, we will create individual components which will allow us to deploy a Vault cluster into a cloud environment, for the purposes of critically analysing the method and results.

### Problem statement

**Primary Objective**: Deploy a 3-node Vault cluster into a Digtial Ocean VPC, with a load balancer issued with a public DNS record. The Vault cluster Raft should auto-discover itself.

**Secondary Objective**:

The Raft state of the Vault cluster should be separate from the Vault cluster nodes, in order to allow blue-green deployments, fault-tolerance and backups.

## Components

How should we develop this solution in terms of architectural components?

I propose to separate the boundaries as follows:

1. A cloud footprint
1. A cluster of machines with Vault configured in it

Note that we are only laying down infrastructure components here, not internal configuration and they should very rarely change, if ever.
This is still useful however, since we can test new versions of Vault in independent environments, or we can replace DNS providers, _etc_.

To this end, I have developed two Terraform modules so far:

1. [DigitalOcean VPC]()

## Discussion

This should translate into two terraform modules

---

## Footnotes and References

[^platformopsnow]: I guess we're all Platform Operators now. If you call yourself a DevOp or an SRE, you're welcome here too.
[^cc]: It also requires a credit card, so no thanks.
