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
1. **Zero-touch deploy**: This should not require human intervention. This goal is similar to the concise goal, but emphasises automation.
1. **Zero-trust**: Sensitive data should be retrieved by authorized actors, rather than allowing access to it implicitly.
1. **Immutable and idempotent**: The state should be declared as code, and only change when changes to the code are made.

From an operator's point of view, we should have the experience of setting a few variables, and running `terraform deploy`.

In the perfect world, _no other tasks_ would be necessary to have a full Consul cluster deployed, agents, servers and all, all fully joined.

## Implementation

A common approach here follows the 12-factor approach. Start with a base image, add the base layer of the application that we are deploying (Consul in this case) and then inject configuration via environment variables into that derived image at runtime.
That derived image should contain no specific configuration for the application until it is instantiated in a deployed environment.
This can include the service advertise address, as well as sensitive data such as TLS certificates and the gossip encryption key.

In this case, we will forgo that step and use only basic OS distribution images, using [Cloud Init](https://cloudinit.readthedocs.io/en/latest/index.html).
This removes a large amount of tooling (Packer and Ansible), at the expense of writing a single fairly large YAML template, with all of the limitations of cloud init (more on this later).

The implementation consists of two Terraform modules:

- Digital Ocean VPC
- Digital Ocean Consul cluster

Thus it could be created as follows:

{% highlight hcl %}
module "vpc" {
  source   = "https://github.com/brucellino/terraform-module-digitalocean-vpc/"
  vpc_name = var.vpc
  project  = var.project
}

module "consul" {
  source                   = "https://github.com/brucellino/terraform-digitalocean-consul"
  vpc                      = var.vpc
  depends_on               = [module.vpc]
  project_name             = var.project.name
  servers                  = var.servers
  ssh_inbound_source_cidrs = var.inbound_ssh_cidrs
}
{% endhighlight %}

Here of course, we have declared a few variables in the definition to allow us to create a firewall rules for access from specific locations (typically a Tailscale network), number of servers, _etc_.
Note that there is a dependency on the VPC by the Consul module -- as we describe below, the Consul module looks up existing VPC resources to deploy into.
The architecture _declares_ an explicit dependency between these two modules, but there is not an _implicit_ dependency between the resources in the VPC module and those in the Consul module.

In principle, the VPC could be created in a separate way up front, but since we are starting from scratch, we add it to our state and explicitly declare the dependency.

### Tooling

<figure>
  <img src="{{ base_url }}/images/consul-do-tooling.png">
  <div class="figcaption">Schematic diagram of tooling to create the Cosnul cluster in Digital Ocean.</div>
</figure>

In terms of tooling, these include:

- **Terraform**. Specific providers are:
  - digital ocean
  - http
  - random
- **Consul** (optional): Used as the backend for the Terraform state.
- **Vault**: key-value store for digital ocean tokens
- **Cloud init**: creating the derived image, doing the installation and configuration of Consul on nodes.

Some observations on this tooling:

1. Although the addition of cloud-init seems to break the design goal of immutabililty, since it alters the state of the image, it does allow us to avoid the use of Terraform provisioners and other tooling. The image itself is immutable _after_ cloud init has run, which should only happen at first boot, so I think this is acceptable.
1. The choice of storing the state in a Consul backend is predetermined. I used Consul because I have access to one in [Hashi@Home](https://hashiatho.me), but an S3-compliant object store state could have been used, for example a DigitalOcean space. This is both cost-effective and simple, but has to exist before we can run a plan, just like the Consul backend.
1. Vault is a dependency of this architecture since I am considering it the default means to access sensitive data. We are accessing encrypted key-value data, so this could be provided by a different provider in principle. However, Vault also allows us to **entirely remove** any secrets from the codebase, since even authentication to the backends is done using a `data` lookup of secrets in Vault in Terraform itself.

### Speedbumps

Satisfied with the tooling, I set about implementing the architecture with Terraform.
Most of the objects in the state would of course be `digitalocean_xyz` resources, such as `digitalocean_droplet` for the agent and server cluster, `digitalocean_firewall` rules for managing access, _etc_.
However, we also have a dependency on some external data.
Aside from the tokens used to authenticate to the cloud services, we would need to create an SSH authorized key using a lookup from a user's GitHub keys URL and an existing VPC to deploy into.

So far so good!

That is, until we start enforcing the goal of "zero-touch" deploy.
It soon became apparent that I would have to find some creative ways of configuring Consul on the fly without expanding the toolkit.
A few of the aspects I needed to address to ensure zero-touch are described below.
I call them "speedbumps", since they slowed me down a bit and made me think about what I was doing.

#### Bind, advertise, client addresses

Since we start from scratch, the IP addresses are not known up front.
We would need to know the private IP of the agent, so that we can tell servers to advertise on an address that the other members of the cluster can reach it.
Luckily

#### Cluster joining

#### Key generation

One of the first speedbumps was how to create the relevant Consul configuration.
In particular, the creation of a shared encryption key to allow the servers to join the cluster.

The gossip encryption key is a central value that needs to be injected into the Consul configuration either via the configuration file, or as a [command line parameter](https://www.consul.io/docs/agent/config/cli-flags#_encrypt).
In principle, this does not represent a problem, since we can pass this value into either the `consul.hcl` configuration file, or the startup flag via the systemd unit, both of which are provided by the cloud-init template.
The problem however arises when we ask

> "who knows this value?"

In the past, I have kept the Consul encryption key as a value in one of the key-value stores that Terraform reads from Vault.
However, starting with fresh deploy, we need to _generate_ it.

This looks like a _Catch-22_ at first glance, because we can't generate an encryption key without Consul, but  we can't deploy Consul without an encryption key.
What is more, if it's generated on one Consul server, how is it then shared with the others?
I spent a few hours thinking about this, tempted to revert back to the _"Deus ex Machina"_ approach of pre-registering a key in a Vault KV store... until I read [the documentation](https://www.consul.io/docs/security/encryption#gossip-encryption) again:

> The key must be 32-bytes, Base64 encoded.

This is exactly what the [`random_id`](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/id) resource in the `random` provider does!
I added a `random_id` resource:

{% highlight hcl %}
resource "random_id" "key" {
  byte_length = 32
}
{% endhighlight %}

and then passed it into the userdata  template:

{% highlight hcl %}
resource "digitalocean_droplet" "server" {
  ...
  user_data = templatefile(
    "${path.module}/templates/userdata.tftpl",
    {
      consul_version = "1.12.3"
      server         = true
      username       = var.username
      datacenter     = var.datacenter
      servers        = var.servers
      ssh_pub_key    = data.http.ssh_key.body
      tag            = "consul-server"
      region         = data.digitalocean_vpc.selected.region
      join_token     = data.vault_generic_secret.join_token.data["autojoin_token"]
      encrypt        = random_id.key.b64_std
      domain         = digitalocean_domain.cluster.name
      project        = data.digitalocean_project.p.name
      count          = count.index
    }
  )
  ...
}
{% endhighlight %}

#### Service start

---

[^certification]: Some might say that I should just spend the time and money on a Hashicorp certification. Although I don't disagree with this sentiment, I would also point out that many folks like me only learn how something really works when they break it. It's the physicist in me... I feel far more comfortable saying that I am proficient in a tool when I have used it to solve a problem.
[^lb-dns]: The loadbalancer should probably be replaced with some good old DNS records.
