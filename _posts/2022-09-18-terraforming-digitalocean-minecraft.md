---
title: Terraforming minecraft in Digital Ocean
description: Mines. Oceans. Terraforming. It's all a bit much...
category: Practice
tags:
  - Maintainability
  - Continuous Delivery
  - Terraform
---

Mastery is the ability to apply knowledge in new situations.
Skill is the ability to modify methods that have worked in the past to changing conditions.
I often wonder if I'm actually getting better at designing things, and at using the tools I claim to master.

I've been working with a set of tools for a while now, and I started to wonder if I'm getting stuck in a rut.

Draw a picture, identify dependencies, build an image from a Packer template, deploy cloud resources with Terraform. Rinse, repeat.

I wanted to apply this same approach in a new context, something to clear the cobwebs, with some aspects which I had not faced before.

The problem I wanted to solve was the creation of a minecraft server that my family could use to play safely with their friends online.

## Acceptance criteria

For once, the starting point started at the web, and not on the machine.
I needed to consider a user which arrived from the web.

When organising a gaming session, we come up against the following criteria:

1. The server should be discoverable: I can't expect friends to be co-ordinating with me to find out what the IP of the server is
1. It should be cost-effective: the environment should be up for only as long as we need it and cost only a few euros
1. It should be persistent: The environment data should be persistent even if other resources are shut down to save money
1. It should be safe: I don't want the server exposed to the internet as a whole, but protected by a CDN.

## General approach

Now, I had no idea what minecraft was or how it worked, so the first thing I did was see if we could deploy it on a single instance.
You'll note that I'm not starting with "First, deploy a kubernetes cluster".
It turns out creating a minecraft server involves running a single java JVM, and it can be customised by adding a few plugin files.

### Provisioning a machine

Immediately, the first step in my default approach seemed like overkill.
Do I really need to build a reference image for a minecraft server when I just need to place a few files on it, from well-defined places on the internet?
All of the configuration comes in templating a few configuration files and permissions on directories.
What is more, in our case we don't have autoscaling or continuous deployment of changes -- at least not yet -- so the startup latency isn't a deciding factor.
I could merely start an instance and use cloud-init to provision it.

### Infrastructure

## Minecraft @ Home
