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
It turns out creating a minecraft server involves running a single JVM, and it can be customised by adding a few plugin files.

### Provisioning a machine

Immediately, the first step in my default approach seemed like overkill.
Do I really need to build a reference image for a minecraft server when I just need to place a few files on it, from well-defined places on the internet?
All of the configuration comes in templating a few configuration files and permissions on directories.
What is more, in our case we don't have autoscaling or continuous deployment of changes -- at least not yet -- so the startup latency isn't a deciding factor.
I could merely start an instance and use cloud-init to provision it.

### Infrastructure

I chose to use **Digital Ocean** and **Cloudflare** to provide infrastructure for my little project.

The motivation for this choice goes like this:

1. They are basically free for personal use
1. I already have accounts on them
1. They have rich Terraform providers
1. The pricing model is clear and straightforward

There are two bits of infrastructure needed in this case, roughly:

1. compute and storage
1. connectivity and resolution

_I.e._ "Where would my minecraft instance run; where would it persist its data?" -- the answer to this question would be "Digital Ocean as cloud provider".
Whilst the answer to "What do I need to tell my boy's friends' moms in order to get them to connect" would be the domain I've purchased and manage with Cloudflare.

## Exposing the service

Exposing the machine to the internet in a way that kids and their parents could understand would be most of the challenge to address here.
I could bring up a machine or a container quite easily to run the process, and possibly quite easily assign it a public IP, but I still need to take care of the firewall, DNS, port mapping _etc_.

A typical approach would be to bring up a Load Balancer, assign rules incoming and outgoing traffic rules to it and add a few redirection rules.
The public IP of the load balancer could then be added as an "A" record in DNS.

In any case, I would need a DNS provider in a zone I already own.

## Protecting the service

Now I had a service up and running, how can I ensure that only the relevant people get to connect to it?
I originally thought about putting the entire thing in a Tailscale VPN, but then I'd have to ask the kids to install a Tailscale client on their computers, distribute access keys, _etc_.
On the other hand, there is also the zero-trust functionality of Cloudflare, but this also required the installation of something on the client side.

## Minecraft @ Home
