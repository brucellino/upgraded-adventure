---
title: Terraform off the beaten track
description: |
  Got $5 and some time? Here's a Terraform Module for Consul on DigitalOcean using Cloudflare
---

- [Choosing a stack: design and priorities](#choosing-a-stack-design-and-priorities)
- [Terraforming DO and Cloudflare](#terraforming-do-and-cloudflare)
- [The best part of Terraform](#the-best-part-of-terraform)
- [Some finer details](#some-finer-details)
- [Securing the Consul cluster](#securing-the-consul-cluster)

---

# Off the beaten track

There has been, in my experience, no better way to check that one understands something, than to cautiously veer off the beaten path.

For about 3 years my job was to put AWS to work.
When you do something for a living, and most importantly, for specific business goals, there is not a lot of priority placed on perfecting your trade.
The first mark of success is whether what you do _actually works_ or not; then come the other aspects such as whether it is performant, maintainable, cost-effective, secure, _etc_.
Of course, if it is not all of these things, it's no good anyway, but the **first** thing that really matters is whether it works at all.

I have always been interested in doing things in a slightly nonstandard way.
However, this is usually not a good idea in a professional context, not least because you have to collaborate with others and thus make every attempt to write components that are easy to understand.
Often this means defining a default path, documenting and then following it in all but the most exceptional cases.

Satisfied with my contributions at work, I nevertheless still had an itch to scratch.
While I felt quite comfortable with the [Hashicorp](https://hashicorp.com) tools (Terraform, Vault, Consul, Nomad) and [Ansible](https://www.ansible.com), things like DNS, routing and CDN still felt a little magical to me.
So, I set myself a little challenge: **Deploy a secured Consul cluster on a cloud other than AWS**.

## Choosing a stack: design and priorities

Priorities and conditions for success are quite different in this case.
In this case, I would be paying out of my own pocket, and the value came not from consumption of the final product, but the lessons learned while breaking it, or failing at the task.

These aspects further drove me away from choosing AWS as a platform and to select rather [Digital Ocean](https://www.digitalocean.com/) (DO), with a far simpler cloud service offering and very predictable pricing.
In fact, I had $5 of credit waiting for me to do something with it there!

While Digital Ocean does offer DNS and certificate provisioning, it does not have a CDN and much in the way of access control.
I was however already using [Cloudflare](https://www.cloudflare.com/) for domain and certificate management, and their service offering has greatly expanded recently.

## Terraforming DO and Cloudflare

Both of these cloud services also have good Terraform providers, and Digital Ocean has [Consul cloud autojoin](https://www.consul.io/docs/install/cloud-auto-join#digital-ocean) so it seemed that the way off the beaten path was set: I would write a Terraform statement to

- Create a DO VPC into which we deploy our DO droplets with private IPs:

{% highlight hcl %}
resource "digitalocean_vpc" "vpc" {
  name        = "terraform-consul-vpc"
  region      = "ams3"
  description = "VPC for Consul"
  ip_range    = var.vpc_cidr
}

{% endhighlight %}

- Deploy a set of 3 DO droplets with Consul pre-configured on them.

{% highlight hcl %}
resource "digitalocean_droplet" "consul_server" {
  count             = 3
  name              = "consul-server-${terraform.workspace}-${count.index}"
  image             = data.digitalocean_image.consul_server.id
  region            = "ams3"
  ssh_keys          = [data.digitalocean_ssh_key.test.id]
  size              = "s-1vcpu-1gb"
  backups           = false
  monitoring        = false
  vpc_uuid          = digitalocean_vpc.vpc.id
  tags              = ["consul-server"]
  graceful_shutdown = false
}
{% endhighlight %}

- Create DO cloud firewalls to restrict traffic
- Attach them to a Digital Ocean load balancer

{% highlight hcl %}
resource "digitalocean_loadbalancer" "consul" {
  algorithm                = "least_connections"
  redirect_http_to_https   = true
  enable_backend_keepalive = true
  name                     = var.lb_name
  size_unit                = var.lb_size_unit
  region                   = "ams3"
  vpc_uuid                 = digitalocean_vpc.vpc.id
  droplet_tag              = "consul-server"

  forwarding_rule {
    entry_port     = "443"
    entry_protocol = "https"
    target_port     = var.consul_ports.http
    target_protocol = "http"

    certificate_name = digitalocean_certificate.cert.name
  }

  healthcheck {
    port     = var.consul_ports.http
    protocol = "http"
    path     = "/v1/health/service/consul"
  }

}

{% endhighlight %}

- Assign DNS records for the load balancer and droplets

{% highlight hcl %}
resource "cloudflare_record" "consul" {
  zone_id = data.cloudflare_zone.hashiathome.id
  name    = "consul-${terraform.workspace}"
  value   = digitalocean_loadbalancer.consul.ip
  type    = "A"
  ttl     = 60
}

resource "cloudflare_record" "droplets" {
  count   = 3
  zone_id = data.cloudflare_zone.hashiathome.id
  name    = digitalocean_droplet.consul_server[count.index].name
  value   = digitalocean_droplet.consul_server[count.index].ipv4_address
  type    = "A"
  ttl     = 60
}
{% endhighlight %}

- Issue a certificate to the load balancer -- in this case, it actually consists in:
  - Create a CSR ( `tls_private_key` resource)
  - Create the cert request (`tls_cert_request` resource)
  - Issue the Cloudflare cert ( `cloudflare_origin_ca_certificate` resource )
  - Import it into DO ( `digitalocean_certificate` resource )
  - and finally, use the certificate in the forwarding rule from HTTPS inbound traffic.
- Added bonus restrict access using CloudFlare Access

## The best part of Terraform

This design may not be the greatest for any practical purpose, but it demonstrates what to me is the best part of Terraform: the ability to pull co-ordinate several providers into a single state.
While the bulk of the work is done on Digital Ocean, in fact several other pieces are necessary to produce a working setup.
This makes reference not only to the other main provider, Cloudflare, but also the [TLS provider](https://registry.terraform.io/providers/hashicorp/tls/latest) necessary for creating the certificate signing request to send to Cloudflare, and the [Vault provider](https://registry.terraform.io/providers/hashicorp/vault/latest) necessary for retrieving secrets.

Without this ability to declare a host of different resources and tie them together into a coherent state using a single language, I would have had to write several scripts or functions myself, and tie them together with as much glue code.
I can focus on _what_ I want, rather than _how_ to get it and honestly, it feels like cheating.

Whole thing takes less than 3 minutes to provision and is just as easy to destroy.
With comfort like this, who _doesn't_ want to do full end-to-end testing of infrastructure as code!?

## Some finer details

Of course, there are a few bits left "as an exercise to the reader" in the snippets above, left out for the sake of readability.
However, I wanted to highlight some of the elegant tricks Terraform allows me to play, beyond orchestrating a few very different cloud services.

The first is how I am able to use the Vault provider to easily manage access to secrets.
In order to actually use the Cloudflare and Digital Ocean clouds, I need to authenticate against them, typically via the use of tokens.
Having pre-issued these tokens via the respective cloud dashboards, I have stored them in a Vault instance (running locally on one of my Raspberry Pis!).
With a few `vault_generic_secrets` lookups, I can retrieve this encrypted sensitive data and void storing it in the code.
This is kept in the state, which is in a separate Consul cluster (also running on my raspberry Pis locally).

The second is somewhat trivial but I found it delightful, which was the use of a Terraform `map` to define the Consul ports that need to be connected to each other and load balancer:

{% highlight hcl %}
variable "consul_ports" {
  type        = map(number)
  description = "Ports to expose Consul on. See https://www.consul.io/docs/install/ports"
  default = {
    "dns"      = 8600
    "http"     = 8500
    "serf-lan" = 8301
    "server"   = 8300
  }
}
{% endhighlight %}

This not reduces the amount of code I need to write for the firewalls:

{% highlight hcl %}
resource "digitalocean_firewall" "droplet_consul" {
  for_each    = var.consul_ports
  name        = "consul-servers-${each.key}"
  droplet_ids = digitalocean_droplet.consul_server.*.id
  inbound_rule {
    protocol   = "tcp"
    port_range = tostring(each.value)

    source_droplet_ids        = digitalocean_droplet.consul_server.*.id
    source_load_balancer_uids = [digitalocean_loadbalancer.consul.id]
  }
}
{% endhighlight %}

but also documents quite clearly _why_ these ports need to be open.

## Securing the Consul cluster

There are a few things missing before I would consider this setup to be safe.

The first is of course the [Consul ACLs](https://www.consul.io/docs/security/acl).
They are not considered at all here, but the ACL system should be bootstrapped during the creation of the Consul cluster.
Secondly is the protection of the entire system behind some form of VPN.
My first choice here is to declare it as a CloudFlare service and add some authentication in front of the loadbalancer.
After a few manual attempts at setting this up I haven't been successful yet, so I guess that is left for "Part II".

---

So there you have it: **a Consul cluster on Digital Ocean, fronted by CloudFlare, in less than 300 lines of code, less than 3 minutes to deploy and less than USD 5 total.**

Not bad. Not bad at all.
