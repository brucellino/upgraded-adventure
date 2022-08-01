---
title: Consul in DigialOcean (part II)
description: A few more steps towards a production-grade Terraform module for Consul in DigitalOcean
---

**How far is a production-ready Terraform module from the tutorial?**

In my estimation, and calibrated to my skill level, it's about 3-5 days of work. Read further for how I came to this estimation, and what it means.

## Hashifinity Stones

Over the course of about 3 days of work, I developed a few Terraform modules for Digital Ocean with the goal of deploying a full Hashi environment -- Consul, Vault and Nomad.
Only Boundary and Waypoint would be needed to complete the full collection of Hashicorp "infinity stones"!

More to the point, these are products which I use personally and professionally, and which I often propose as technical solutions to customers where I work, so I really need to know how practical and effective my knowledge of them is[^certification].

My goal with this post is to talk seriously and objectively about how much work is involved in making a production-ready Terraform module, and what I've learned so far making one for Consul on Digital Ocean.
Now, I'm not a newbie to Terraform or Consul, and Digital Ocean is probably the world's simplest well-known cloud, so the threshold to getting something serious done is pretty low.

That being said though, there are a few things which tripped me up on the way

---

[^certification]: Some might say that I should just spend the time and money on a Hashicorp certification. Although I don't disagree with this sentiment, I would also point out that many folks like me only learn how something really works when they break it. It's the physicist in me... I feel far more comfortable saying that I am proficient in a tool when I have used it to solve a problem.
