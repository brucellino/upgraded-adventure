---
title: A terraform module for Vault on DigitalOcean
layout: post
description: Architecting for clean cloud deployment
category: architecture
toc: true
tags:
  - blog
  - platform-engineering
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

TL;DR You can find the module [in the terraform registry](https://registry.terraform.io/modules/brucellino/vault/digitalocean/latest)

### Solution Context

There are plenty of ways to deploy Vault, but this one has a few considerations which you may find yourself identifying with.
The overall context is that this service (Vault) is a component of an **internal developer platform** which supports the delivery of workloads to downstream users.
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

## Design Decisions

Design decisions are the constraints and opinions imposed on the module that express what it is designed to do.
Our design decisions are informed by the the single goal we have set above, and constrain the user experience of the service such that it becomes consistent with the rest of the services in the platform.

These decisions are meant to reduce the scope of the module in order to make it maintainable, and help operators decide when it is appropriate to use it.

The design decisions are listed below:

1. Design decisions will be documented using the ADR format.
1. The resource provider will be DigitalOcean.
1. Vault will be served by configured processes on virtual machines.
1. Vault will be configured using userdata at provision time.
1. Vault data will be persisted to attached storage.
1. The virtual machines will be configured with private networking in a virtual private network.
1. Networking over the public interface will be disabled via firewall rules which permit access only to specific services, and **not** the Vault service.
1. An overlay network will be provisioned and Vault will communicate only via this overlay network.
1. mTLS will be configured between Vault instances.
1. Certificates will be generated with a private CA, as part of the Terraform state.
1. The final state will be a sealed Vault cluster with no configured secret stores, authentication mechanisms or policies.

### Discussion

Why do we take these specific decisions?
Are they objectively better than alternatives?
The answer is no, they are not objectively better, indeed there can be no objectivity here, because we are building something with a specific _intent_ in mind.
This is a module for use by teams who depend on a platform which is built by a small team of platform engineers.
There are constraints all over this scenario, so what we design must take those constraints into account, and respond to the subjectivity of the situation.
We could have decided to deploy Vault via a helm chart on a DOKS cluster, or decided to expose it only via a load balancer.
We could have decided to add a backup operator which backed up raft state to an object store, or we could have decided to use a managed database as Vault backend.
All of these would have introduced additional complexity and dependencies, while keeping the actual service (Vault) constant.

### Design dependencies

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
- A **pre-existing Vault cluster** which contains the secrets necessary to initialise the providers of the dependencies above.

The first two are useful for organising and billing resources and are provided by an upstream module.

The third is a design choice to ensure that we have secure overlay to provide access to the instances of the cluster, and we can expose the service only via that interface.

### Control plane vs security plane

The last dependency one might seem like a unsatisfiable requirement -- how can I deploy Vault if I need Vault to deploy Vault!?
The impression of redundancy may change if one remembers that we are deploying an _instance_ of Vault for a specific team or community, so it is one of potentially many instances, provided as a managed service to that team or community.
It is therefore part of a security plane, not part of the control plane.

Another way to put it might be that the users of the provisioned Vault instance are the members of the team or community which are _served by_ the platform, whilst the users of the Vault cluster we depend on are the members of the platform engineering team which _build_ the platform itself.

## Using the module.

Let's close out this long navel gaze by giving a demonstration of how to use the module.
The module contains a few examples of how to use it, we will use the "simple" example here, which makes no assumptions about the pre-existing resources, and deploys into a commonly-used region (AMS3).

We first declare the provider configurations: We will need a DigitalOcean and Tailscale API tokens in order to create the resources, and those tokens are stored in Vault.
We look them up with a few `data` calls, then use them to configure the providers:

{% highlight hcl %}
# Vault configured with environment variables.
# The user only needs to know where the central operations Vault instance is, and have a token with the relevant permissions issued to them.

provider "vault" {}

# We have declared a variable to hold the value of the KV mount path.
# The path is also given to us by the operations team, who are responsible for
# separation of concerns.
data "vault_kv_secret_v2" "do" {
  mount = var.do_kv_mount_path
  name  = "tokens"
}

# Similarly, the tailscale tokens are stored in the same mount,
# using a different key.
data "vault_kv_secret_v2" "tailscale" {
  mount = var.do_kv_mount_path
  name  = "tailscale"
}

# Now we can configure the DigitalOcean provider
provider "digitalocean" {
  token = data.vault_kv_secret_v2.do.data["terraform"]
}

# And the Tailscale provider
provider "tailscale" {
  api_key = data.vault_kv_secret_v2.tailscale.data.api_key
}
{% endhighlight %}

Vault needs to be deployed into a VPC, but that is not of concern to the Vault module, so it needs to created with its own module.
In this case, it's being kept in the same state as the Vault module itself, so it will be deleted when Vault itself is deleted.
This may be the case in a short-lived, entirely standalone project for example.
In other cases, an existing VPC may be desired, provisioned by a different team.

Having created the VPC and project resources, we can then deploy Vault into it:

{% highlight hcl %}
# First, we make the VPC and project, using a handy existing module in our registry
module "vpc" {
  source          = "brucellino/vpc/digitalocean"
  version         = "2.0.0"
  # The project and VPC have been declared above as variables,
  # but we omit the details here for brevity.
  project         = var.project
  vpc_name        = var.vpc_name
  vpc_region      = "ams3"
  vpc_description = "Vault VPC"
}

# Now we make our Vault service
module "vault_cluster" {
  create_instances         = true
  instances                = 3
  depends_on               = [module.vpc]
  source                   = "../../"
  # These use the same variable as the VPC module,
  # we could have used the outputs from the module instead.
  vpc_name                 = var.vpc_name
  project_name             = var.project.name
  # We have declared a local variable previously,
  # which contains the IP we want to allow SSH access from.
  ssh_inbound_source_cidrs = [local.addr]
  region_from_data         = false
  region                   = "ams3"
  # This is the Digital Ocean token which allows Vault to lookup
  # other instances in the platform to peer with.
  auto_join_token          = data.vault_kv_secret_v2.do.data["vault_auto_join"]
}

{% endhighlight %}

## Conclusion

Hey presto, you now have a self-service module for creating Vault clusters on Digital Ocean.
Using this module requires knowledge of the location of a central Vault instance where platform secrets are kept, and a token from that Vault that allows access only to the secrets you need to provision resources in Digital Ocean and Tailscale.

Once you've created the cluster, you have have full control over it, and can subsequently terraform it with the another module, fit for your specific needs.
Maybe you want to connect it to your Identity Provider, maybe you want to use it to issue Nomad tokens for jobs, maybe you want to connect your managed databases in your application workloads to it... the choice is yours.

And that's the whole point at the end of the day!

Self service can be hard to do right.
With this simple module, we have taken explicit design decisions, paring down the scope of what one can do with these powerful providers, and putting just enough control in the hands of the user that they can create their resources themselves in a predictable and consistent manner.
