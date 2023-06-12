---
title: Deploying Krateo with Terraform
description: One way to skin a cat
category: PlatformOps
tags:
  - blog
  - platformops
  - evaluation
  - unboxing
  - Terraform
  - Krateo
---

This post will describe my experience in getting up and running with [Krateo](https://krateo.io) in a toy environment on Digital Ocean.

I will pay particular attention to the paper cuts encountered while getting this up.

## Problem Statement

Let's set a few criteria for ourselves, to see if this little experience was successful:

1. **Zero-touch declaration**: I shouldn't have to do anything but write a declaration of what I want. No scripts, no manual intervention, no checking things somewhere.
1. **Zero-magic**: The declaration contains everything I need to know. I shouldn't have to invoke a _deus ex-machina_ at some point, assuming that there is something else I already know.
1. **Minimise work**: I should have to write the smallest possible declaration, the smallest possible set of moving parts

The solution will consist of a Kubernetes cluster with Krateo installed on it, exposed by a load balancer, with a DNS name associated with it.
You will need:

1. A Kubernetes cluster
1. A DNS zone which you control

## Approach

I'm using [Digital Ocean (DOKS)](https://docs.digitalocean.com/products/kubernetes/) to create the Kubernetes cluster, and [Cloudflare to manage my DNS zone](https://developers.cloudflare.com/dns/manage-dns-records/).
This is an almost zero-cost way to set up the basic infrastructure required for the problem[^cost], but acounts on these platforms are a prerequisite to using them.

My desire was to be able to implement solutions to the problem in different ways and evaluate them.
These solutions are in an accompanying repository: [brucellino/jubilant-umbrella](https://github.com/brucellino/jubilant-umbrella)

### Terraform implementation

The first implementation (and the only one so far) was done with [Hashicorp Terraform](https://terraform.io).
This choice satisfies the three criteria state above, by containing a single definition of all of the infrastructure, with zero manual intervention and no undeclared steps.

The core of the implementation are two resources:

- [`digitalocean_kubernetes_cluster`](https://registry.terraform.io/providers/digitalocean/digitalocean/latest/docs/resources/kubernetes_cluster)
- [`cloudflare_record`](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/resources/record)

The first is the actual K8s cluster required to install Krateo to, while the second satisfies the requirement that the endpoint be passed to the ingress controller in order to expose the services.

There are several other tidbits which I found necessary to add, whether for conciseness, elegance, or one of the three requirements stated above.

#### Vault provider to configure cloud providers

I have tokens for Cloudflare and Digital Ocean stored in my Hashicorp Vault, which are consumed as Terraform `data` sources in order to pass them to `provisioner {}` blocks for the respective cloud providers.
While this is not strictly required, it's a default engineering practice I always employ when building infrastructure with Terraform and adds a bit of safety to the process.
Making the provider secrets declarative instead of hiding them in environment variables or special files which cannot be committed to the repository makes a clear separation of concerns between the infrastructure's _code_ and _data_.

Somebody else can thus more easily re-use this terraform module, just by passing the relevant Vault parameters to their data lookup:

{% highlight hcl %}
data "vault_kv_secret_v2" "do" {
  mount = var.do_vault_mount
  name  = var.do_vault_secret
}

provider "digitalocean" {
  token = data.vault_kv_secret_v2.do.data["terraform"]
}
{% endhighlight %}

This has the downside of having to include another provider (Vault), but that's a satisfactory tradeoff for the safety and re-usability that we gain, in my opinion.

#### Installing Krateo

Krateo installation is an imperative task; the Krateo CLI has to be executed against the cluster, and cannot simply be _declared_ into existence.
This fact breaks the first requirement (zero-touch declaration) at first glance, unless we can find some way around it.
The default approach would be to

1. Terraform Digital Ocean to declare the K8s cluster into existence
1. execute `krateo init --kubeconfig ${output from first step}`
1. Terraform Cloudflare to declare the DNS record into existence

This kind of imperative, step-by-step approach is prone to being unreliable, and is one of the main reasons that [declarative formats are strongly suggested](https://12factor.net/).
Yes, we can codify these steps into a pipeline, and that's a great start if there's no alternative, but any pipeline can fail unpredictably.
Besides the reliability of the pipeline, we're adding extra work by forcing steps to be taken, in a specific order.
We have to keep several things in our head at the same time, which are not explicitly and irrevocably linked between them. In the software development world, this is the kind of thing that would compile fine, and then generate runtime errors, forcing the developer (operator in our case), to break a state of flow with ana interruption, go back and debug.

Long story short, I really wanted to make the deployment as declarative as possible, so I chose to add a [`null_resource`](https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource) resource linked to the creation of the kubernetes cluster, in order to represent the imperative Krateo installation.

## Results and Discussion

The final results of this experiment may be summarised as such:

1. The criteria set in the problem statement were respected, save for one
1. Krateo installation was done successfully, but
1. I couldn't get it to expose the `app` endpoint properly

These results are discussed in a bit more detail below, but all-in all, I'd give myself a 70% satisfaction rating.

### Interactivity and Concerns

The first paper cut I encountered was having to deal with the Krateo CLI interactivity.
While I could use the attributes from the `digitalocean_kubernetes_cluster` to write the kubeconfig file using a `local_file` resource, and thus pass it to `krateo init --kubeconfig`, but the CLI expects human input in order to configure the app endpoint.
I found this somewhat inelegant, based on the criteria I've set myself, and I would have preferred to _declare_ the domain name in some way.
I ended up having to pass it to the CLI via the command line in the `local_exec` provisioner used in the `null_resource` representing the Krateo installation:

{% highlight hcl %}
echo '${var.cf_zone}' | ./krateo init --kubeconfig ${local_file.k8sconfig.filename}"
{% endhighlight %}

Here we can see that we pass the cloudflare zone represented by the variable `cf_zone` to the `krateo init` execution via a shell pipe -- old school.

The second thing I needed to take care of was to implement a way to cleanly remove all of the resources that were created _by Krateo_ during installation.
In this small experiment, the only such resource was a loadbalancer created to expose the `app` service.
This is in the Terraform state, but instead in the Krateo state.
Since Krateo isn't in the Terraform state either, only the `null_resource` representing it -- so if we do a `terraform destroy`, the resources that Terraform knows about will be destroyed, but _not those Krateo made_.

Luckily, Krateo implements a cleanup target for its CLI, so we can invoke that at destroy time by adding a relevant Terraform provisioner, which should run.
Putting it all together:

{% highlight hcl %}
resource "null_resource" "k_install" {
  triggers = {
    kube_config = local_file.k8sconfig.filename
  }
  provisioner "local-exec" {
    when        = create
    command     = "curl -fSL ${local.krateo_release_url} | tar xz krateo >krateo"
    interpreter = ["/bin/bash", "-c"]
  }

  provisioner "local-exec" {
    when        = create
    command     = "echo '${var.cf_zone}' | ./krateo init --kubeconfig ${local_file.k8sconfig.filename}"
    interpreter = ["/bin/bash", "-c"]
  }

  provisioner "local-exec" {
    when        = destroy
    command     = "./krateo uninstall --kubeconfig kubeconfig-krateo-control-plane"
    interpreter = ["/bin/bash", "-c"]
  }
}
{% endhighlight %}

### Change the endpoint

What about if I wanted to change the endpoint?
Krateo expects, as mentioned above, an input parameter to allow it to tell the ingress controller how to expose its services. This is a hardcoded to `app.<domain>` where `<domain>` is the top-level domain that you are deploying Krateo to.
This is almost certainly something that can be changed by applying a different configuration, but it would have been nice to have this configurable via the same `init` function.

The good news was that running `init` again with a different TLD resulted in the desired configuration being applied.

### Unpredictable load balancer name

During installation, Krateo creates an ingress controller which manages a Digital Ocean loadbalancer.
The public IP of that load balancer is required in order to add the `A` record to the DNS in order to interact with the Krateo App, but since this load balancer is managed by Krateo, it is not known to the Terraform state.

I first tried to add an external loadbalancer, which I wanted to tell Krateo about, but that didn't work out of the box - Krateo ignored it and added it's own.
The `data` block that should discover this loadbalancer does depend on the Krateo installation `null_resource`, but there is a delay between when Krateo exits and when the loadbalancer becomes available.
However, the real deal breaker is the inability to **declare the name of the loadbalancer** _a-priori_, which necessarily introduces esoteric knowledge -- magic information that I just need to know and can't derive.

I ended up breaking requirements 2 and 3 described in the beginning of this post, by

1. having to add esoteric knowledge (the name of the Krateo-managed loadbalancer)
1. having to run terraform twice (ugh, gross) in order to pick up the load balancer

{% highlight hcl %}
# LB created by krateo.
data "digitalocean_loadbalancer" "krateo" {
  depends_on = [null_resource.k_install]
  # id         = "d72d4916-9023-4616-b292-33032dda4799" # <- obtained from the console
  name = "a6434671d1dde4647804e9cd6261d5d6" # <- obtained from the console.
}

resource "cloudflare_record" "k" {
  zone_id = data.cloudflare_zone.k.id
  type    = "A"
  proxied = true
  name    = join(".", ["app", var.cf_zone])
  value   = data.digitalocean_loadbalancer.krateo.ip
}
{% endhighlight %}

The `name` attribute, in my ignorance of how to use Krateo effectively, cannot be known in advance, and is computed by Krateo.
I could probably add a null resource to run a `doctl` in order to look up the load balancer and pass its attributes to the `cloudflare_record` resource, or a `kubectl` to do something similar, but I didn't want to add extra tooling at this point and indeed, I wanted to force the issue by surfacing this "problem".

This is, in my opinion, a documentation problem more than a design problem, since I can definitely imagine ways to get around this, but they all make me throw up in my mouth a bit.

### SSL and ingress errors

The showstopper for me was the inability to actually access the `app.<domain>` URL due to SSL and ingress errors.
Behind the scenes, I could see that all Krateo components had been installed, and everything was reporting healthy.
However, I was unable to access the UI, since the URL gave [HTTP 522](https://http.cat/522) errors (timeouts).
I didn't spend too much time investigating, but my suspicion was that

1. A firewall rule was blocking the connection between the LB and the services in the cluster - either at the infrastructure (Digital Ocean) level, or at the API gateway[^kong] level
1. Somewhere a selector was improperly configured -- perhaps an authentication service was missing which the API gateway was sending requests to,  resulting in the 522

Whatever the true reason, I'm confident that this could be resolved by adding a few `kubernetes_xyz` resources using the [Terraform Kubernetes provider](https://registry.terraform.io/providers/hashicorp/kubernetes/latest).

## Summary

The goal of this little exercise was to get my hands dirty with Krateo, while staying true to some of the engineering principles I hold dear.
I was about 70% successful at this.
I have no doubt that it's possible -- easy even -- to deploy Krateo in this way, with a bit more understanding of how it is supposed to work.
I have a suspicion that a bit more detail in the documentation could have helped, or perhaps a tutorial showing how to modify the vanilla installation with a few `kubectl`s after `krateo init`.
I don't expect the Krateo CLI to do everything after all!

I should also remind the reader that deploying Krateo is very likely a one-time event.
This is a service which will act as the control plane for your entire infrastructure after all, so I don't expect folks to be doing `krateo init` once a week in the end-use environment.
However, for people like me who will eventually end up doing it _for clients_, the process isn't 100% yet.

### Who governs the governor

There is however a deeper question which has bugged me throughout this exercise:

> Am I doing it wrong?

Krateo is for governance, it's supposed to contain the control plane for everything.
But it needs to _emerge from the void_, something [I've written about before](https://hashiatho.me/blog/2022/10/22/base-platform/).
The demos I've seen before start with "Create a Kubernetes cluster"[^other_resources], and I can imagine that when used in an enterprise environment, it will be a bit more like "Install Krateo into an existing cluster".
But who creates those resources?
It can't be Krateo because it doesn't exist in that environment yet.
Does Krateo become self-aware after installation and resolve the bootstrap paradox by then managing itself?

I can't shake the ghost of Godel whispering in my ear:

> "the system is necessarily incomplete".

If something extra is always required to invoke a governor, if this is indeed an emergent property, what is the most elegant way of expressing this?

I do not have an answer to this yet. Do you?

---

[^cost]: To give an idea, the cost for the cluster and associated resources was 10 euro cents for 3 hours of use.
[^kong]: The API gateway used by Krateo is [Kong](https://konghq.com/)
[^other_resources]: As we've seen, this is also not sufficient -- you need a few other resources in order to properly deploy Krateo. While the DNS domain is mentioned, there are indeed some other requirements which are not declared explicitly.
