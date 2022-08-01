---
title: Consul in Digital Ocean (part II)
description: A few more steps towards a production-grade Terraform module for Consul in Digital Ocean
---

<div style="text-align: center;">
  <a href="https://www.digitalocean.com/?refcode=ed3b69c0eec6&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge"><img src="https://web-platforms.sfo2.digitaloceanspaces.com/WWW/Badge%202.svg" alt="DigitalOcean Referral Badge" /></a>
</div>
<div style="text-align: center; text-size: small">Use this link of you want to get $5 free Digital Ocean credits. That's about how much it cost me to build this module.
</div>

**How far is a production-ready Terraform module from the tutorial?**

_In my estimation, and calibrated to my skill level, it's about 3-5 days of work._
Read further for how I came to this estimation, and what it means.

## 3/5 Hashifinity Stones

Over the course of about 3 days of work, I developed a few Terraform modules for Digital Ocean with the goal of deploying a full Hashi environment -- Consul, Vault and Nomad.
Only Boundary and Waypoint would be needed to complete the full collection of Hashicorp "infinity stones"!

More to the point, these are products which I use personally and professionally, and which I often propose as technical solutions to customers where I work, so I really need to know how practical and effective my knowledge of them is[^certification].

My goal with this post is to talk seriously and objectively about how much work is involved in making a production-ready Terraform module, and what I've learned so far making one for Consul on Digital Ocean.
Now, I'm not a newbie to Terraform or Consul, and Digital Ocean is probably the world's simplest well-known cloud, so the threshold to getting something serious done is pretty low.

That being said though, there are a few things which tripped me up on the way, which the cool kids are calling "learnings" these days (ugh, gross).

<!-- Axes of competence -->

<!--

- architecture and design
- networking design
- SSL/TLS
- terraform
- Consul
- Cloud platform (DO)

 -->

## What are we building

The first lesson is one in design
It's rare that I get the chance to build something from scratch, so I don't have much experience when it comes to designing the architecture of something, even something as simple as this.
That is not to say that it is particularly _difficult_, just that it's an unexercised muscle.

I took the [Consul Reference Architecture](https://learn.hashicorp.com/tutorials/consul/reference-architecture) as the starting point for creating a module which would deploy something similar to it in Digital Ocean.
Similar to the [datacenter deployment guide](https://learn.hashicorp.com/tutorials/consul/deployment-guide?in=consul/production-deploy) I started with a single availability zone AMS3.

Following the tutorial, the idea would be to create a set of droplets for the servers, a set of droplets for the agents, and a load balancer to front the servers[^lb-dns].
These would all be included in a VPC to allow the agents and servers to communicate with each other over a local network.
Cloud firewalls would then protect the instances by allowing only traffic on the [Consul ports](https://www.consul.io/docs/install/ports)

<figure>
  <img src="{{ base_url }}/images/consul-digitaloceandrawio.png">
  <div class="figcaption">Schematic diagram of a Consul reference architecture in Digital Ocean.</div>
</figure>

### Design goals

The goals for this architecture are, in no particular order:

1. **Concise**: As little code should be written and as few tools invoked as necessary, no more.
1. **Complete**: The architecture should be _fully_ described by the modules
1. **Zero-touch deploy**: This should not require human intervention
1. **Zero-trust**: Sensitive data should be retrieved by authorized actors, rather than allowing access to it implicitly.
1. **Immutable and idempotent**: The state should be declared as code, and only change when changes to the code are made.

### Implementation

A common approach here follows the 12-factor approach. Start with a base image, add the base layer of the application that we are deploying (Consul in this case) and then inject configuration via environment variables into that derived image at runtime.
That derived image should contain no specific configuration for the application until it is instantiated in a deployed environment.
This can include the service advertise address, as well as sensitive data such as TLS certificates and the gossip encryption key.

### Tooling

- Terraform. Specific providers:
  - digital ocean
  - http
  - random
- Vault
  - key-value for digital ocean tokens
- Cloud init

## Speedbumps

### Key generation

---

[^certification]: Some might say that I should just spend the time and money on a Hashicorp certification. Although I don't disagree with this sentiment, I would also point out that many folks like me only learn how something really works when they break it. It's the physicist in me... I feel far more comfortable saying that I am proficient in a tool when I have used it to solve a problem.
[^lb-dns]: The loadbalancer should probably be replaced with some good old DNS records.
