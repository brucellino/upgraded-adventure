---
title: Consul for external services
layout: post
description: Can we use Consul as a service registry for arbitrary infrastructure?
category: platform-engineering
toc: true
tags:
  - blog
  - platform-engineering
  - Consul
  - Terraform
---
Table of Contents
- [Service registration vs service discovery](#service-registration-vs-service-discovery)
- [Registering services in a modern service catalogue](#registering-services-in-a-modern-service-catalogue)
- [Using Consul as Service Catalogue](#using-consul-as-service-catalogue)
  - [Problem statement: Declared vs Actual states](#problem-statement-declared-vs-actual-states)
  - [Terraforming federated services into Consul](#terraforming-federated-services-into-consul)
    - [External Terraform Data source](#external-terraform-data-source)
    - [External Consul Node](#external-consul-node)
    - [Consul Service](#consul-service)
    - [External Service Monitor](#external-service-monitor)
- [Results](#results)
  - [I promised you UX improvements](#i-promised-you-ux-improvements)
  - [Discussion](#discussion)

## Service registration vs service discovery

This is a short experiment in using Consul as a service discovery tool for services in federated infrastructures.

Traditionally, services were not "discovered" but rather registered in a **configuration database**, which then acted as a source of truth for clients wishing to find out information regarding the available resources in the federation.
This configuration database (CMDB) remains an authoritative source of truth, since service owners manually register their inventory therein and there is a high level of manual verification, however it is not easily **queryable**.

One of the main infrastructure services is the **information system**, which is implemented with the [Berkeley Database Information System (BDII)](https://docs.egi.eu/users/compute/high-throughput-compute/querying-information-system/) with a specific LDAP schema.
The BDII provides a hierarchical way to propagate service registration from so-called "sites" up to a centralised service catalogue.
This model was adopted for more than a decade and supported a truly massive computing effort in the pursuit of new knowledge.

My memory may be failing me after almost 7 years away from actually working in this environment, but I remember actually _using_ the BDII being extremely laborious.
There was a lot to actually remember, which ended up written in wiki after wiki, generating a great amount of toil and thus cost, and communities inevitably ended up needing to centralise and codify knowledge about the state of the infrastructure in single instances.

## Registering services in a modern service catalogue

A common design pattern in the 2020s is to use a [service mesh](https://landscape.cncf.io/guide#orchestration-management--service-mesh) to register services and define their connection permissions.
This is conceptually similar to the BDII approach of creating and registering services from the ground up, but instead of using LDAP with LDIF updates, we use DNS.

This brings a radically different approach to actually _using_ the infrastructure, _i.e_ in **user experience (UX)**.
Relying on DNS means we no longer need to remember arcane `ldapsearch` commands with undecipherable filters, we just need to call a DNS name.
The service catalogue also takes care of organising and routing traffic to the desired endpoints so that applications do not need to be aware of changes in the infrastructure.

## Using Consul as Service Catalogue

Services deployed in a Kubernetes cluster for example get these benefits for free as part of the environment they are running in, but we do not have that luxury in the fedearated infrastructure world.
While some of these services _may_ end up deployed in Kubernetes within a specific local context (site), it is inconceivable that the entire federation would be.
The services can still be registered as [external services](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) which services within a Kubernetes cluster can discover.

However, one does not need to adopt the full apparatus of Kubernetes to benefit from this.

Consul from Hashicorp is well-adapted to bridging legacy and cloud-native infrastructure, since it is designed to be [multi-platform](https://www.consul.io/use-cases/multi-platform-service-mesh.)
I wanted to investigate how it could be used together with existing infrastructure in order to improve the UX of communities which have adopted modern tooling but nevertheless need to use existing federated resources.

Before we get to the big picture stuff though, let's get down in the weeds to see how this could be done effectively.

### Problem statement: Declared vs Actual states

The source of truth hasn't changed, and we don't want to break the Service Management standard we're used to: [FitSM](https://www.fitsm.eu/).
Rather, we want to use existing structures to improve UX.

Service discovery can be considered part of the Configuration Management process (CONFM), which has the specific requirements[^FitSMCONFM]:

- **PR11.1** The scope of configuration management shall be defined together with the types of configuration items (CIs) and relationships to be considered.
- **PR11.2** The level of detail of configuration information shall be sufficient to support effective control over CIs.
- **PR11.3** Information on CIs and their relationships with other CIs shall be maintained in a configuration management database (CMDB).
- **PR11.4** CIs shall be controlled and changes to CIs tracked in the CMDB.
- **PR11.5** The information stored in the CMDB shall be verified at planned intervals

The configuration management database is commonly understood to be the [GOCDB](https://goc.egi.eu), the operations database which implements the functionality required to satisfy the requirements above.
It is a source of information, which is is supposed to describe the  actual state of the world.
Unfortunately, it is **not authoritative** in that sense, since it is merely a database, and the items in it have no controls associated.
There are no connectivity checks, health checks, performance, _etc_ - it's all manually added.

Now, it is true that those checks are delegated to a different service (the monitoring service [ARGO](https://argo.egi.eu)), but again there is no way to interrogate the actual state of the infrastructure in an operational sense.
If I my application or community to use the currently "good" services, I need to somehow hook into the monitoring system, find which is the currently good set of services via some arcane query, and keep doing that all the time.

### Terraforming federated services into Consul

What if we could create an environment where the healthy services were simply discoverable by the thing we all know and use every day: DNS.
I'll show now how Terraform can be used to query the GOCDB, register services in a Consul catalogue and then use the Consul DNS interface to use and discover these services.

**UX improvement**:

This exercise will demonstrate how make NGI or ROC-level BDII endpoints available via DNS.

- Before: User needs to query GOCDB to find a relevant endpoint, remember a default one, or hardcode it in the environment :anger:
- After: User can look up the currently healthy endpoint using a DNS name :heart_eyes:

We'll do the following things to:

1. Create a query script to provide external data to Terraform
1. Create a Consul external node to assign the declared services to
1. Register the declared services along with simple health checks in the external node as **external services**
1. Deploy an external service monitor to discover and perform monitoring checks on these external services

We will do all this in Terraform, so first things first, we initialize our providers[^TFNote]:

{% highlight HCL %}
terraform {
  required_version = "~> 1.7.0"
  required_providers {
    consul = {
      source  = "hashicorp/consul"
      version = "~> 2.20"
    }
    external = {
      source  = "hashicorp/external"
      version = "~> 2.3"
    }
  }
}
{% endhighlight %}

#### External Terraform Data source

Next, we'll need to use the GOCDB as a data source in Terraform.
In order to do this, we use the [`external`](https://registry.terraform.io/providers/hashicorp/external/latest/docs/data-sources/external) data source from Hashicorp.
This is implemented as an arbitrary `program` by the user (me), which returns a JSON that Terraform can parse.
I wrote it in Python:

{% highlight python %}

#!/bin/env python3
import requests
import xmltodict
import json

def get_top_bdii():
    data = requests.get(
        "https://goc.egi.eu/gocdbpi/public/?method=get_service_endpoint&service_type=Top-BDII"
    )
    xpars = xmltodict.parse(data.text, strip_whitespace=True)
    j = json.dumps(xpars["results"], allow_nan=False)

    r = {"output": str(j)}
    print(json.dumps(r))

if __name__ == "__main__":
    get_top_bdii()
{% endhighlight %}

This queries the GOCDB to get the `Top-BDII` service endpoints.
The HTTP call is unauthenticated and returns an XML response which we parse into JSON.
The JSON is added to an `output` dict which is returned to Terraform via `stdin` as required.

#### External Consul Node

We now create single external node (`EGI`) to keep things simple, where we can register all of the declared services[^ExternalNode].
We declare it as "external" using the node metadata, so that the monitor can discover it and monitor services on it:

{% highlight hcl %}
resource "consul_node" "egi" {
  name    = "EGI"
  address = "https://egi.eu"
  meta = {
    "external-node" = "true"
  }
}
{% endhighlight %}

#### Consul Service

Now the meaty bit - we need to parse the result of the GOCDB lookup to create the services, and register them on the external node.

In order to do this, I first declare a `local` variable `service_endpoints` which filters out the result of the GOCDB data:

{% highlight hcl %}
locals {
  service_endpoints = { for v in jsondecode(data.external.bdiis.result.output).SERVICE_ENDPOINT : v["@PRIMARY_KEY"] => {
    key                = v["@PRIMARY_KEY"]
    configuration_item = v.GOCDB_PORTAL_URL
    hostname           = v.HOSTNAME
    sitename           = v.SITENAME
    in_production      = v.IN_PRODUCTION
    scopes             = v.SCOPES
    monitored          = v.NODE_MONITORED
    notifications      = v.NOTIFICATIONS
    country            = v.COUNTRY_NAME
    roc                = v.ROC_NAME
    scopes             = [v.SCOPES.SCOPE]
    }
  }
}
{% endhighlight %}

Now we have a nice variable of type map which we can loop over the keys of in order to register the service declared there in our Consul catalog.

Since there is a 1-to-many mapping between sites and service instances, we need to register the service and check with unique names.
We will use a combination of service name and primary key as defined in GOCDB for this.

In order to avoid repeating ourselves, we will use the `for_each` keyword for the `consul_service` resource and loop over the `local.service_endpoints` keys.

{% highlight hcl %}
for_each   = { for i in local.service_endpoints : "${i.sitename}-${i.key}" => i }
  node       = consul_node.egi.name
  address    = each.value.hostname
  port       = 2170
  name       = "top-bdii"
  service_id = "top-bdii_${each.key}"
  tags = concat(
    [each.value.sitename],
    [each.value.roc],
    flatten([each.value.scopes])
  )
  check {
    tls_skip_verify                   = true
    check_id                          = "service:${each.value.sitename}-${each.value.key}"
    name                              = "${each.value.sitename} top bdii check"
    interval                          = "1m0s"
    timeout                           = "20s"
    tcp                               = "${each.value.hostname}:2170"
    notes                             = "${each.value.sitename} TCP check"
    deregister_critical_service_after = "720h0m0s"
  }
  meta = {
    primary_key   = each.value.key
    ci            = each.value.configuration_item
    hostname      = each.value.hostname
    sitename      = each.value.sitename
    in_production = each.value.in_production
    monitored     = each.value.monitored
    notifications = each.value.notifications
    country       = each.value.country
    roc           = each.value.roc
  }
}
{% endhighlight %}

We've added a series of key-value metadata to reproduce the kind of information that one would find in the GOCDB.
As we'll see, many of the instances registered there are not alive, so their service checks will immediately fail and the service will soon be deregistered.
We only register one health check for now, which is a TCP check on the LDAP server.
It would be _far_ better and more accurate to add an actual LDAP search script check[^ARGOdoesthis], but we'll get into that later in the discussion below.

#### External Service Monitor

Finally, we need to run an [external service monitor](https://developer.hashicorp.com/consul/tutorials/developer-discovery/service-registration-external-services#monitor-the-external-service-with-consul-esm).

I am obviously doing this with something you might not have: a beautiful Nomad cluster.
The external service monitor can be run next to any Consul agent, so in principle you could run it as a systemd unit on one of your Consul agents.
I'm using Nomad so that I can easily manage the deployment and lifecycle.
The final resource is therefore a `nomad_job`:
{% highlight hcl %}
resource "nomad_job" "consul_esm" {
  jobspec = templatefile("${path.module}/consul-esm.jobspec.hcl", {
    consul_esm_version = var.consul_esm_version,
    # spread over available nodes
    count = 5
  })
  rerun_if_dead = true
}
{% endhighlight %}

with associated Job Specification.

## Results

Now, let's deploy this monstruosity and discuss the result.
The Terraform plan looks sane, with a bunch of

{% highlight hcl %}
{% raw %}#{% endraw %} module.example.consul_service.top-bdii["AEGIS01-IPB-SCL-1183G0"] will be created
  + resource "consul_service" "top-bdii" {
      + address    = "bdii.ipb.ac.rs"
      + datacenter = (known after apply)
      + id         = (known after apply)
      + meta       = {
          + "ci"            = "https://goc.egi.eu/portal/index.php?Page_Type=Service&id=1183"
          + "country"       = "Serbia"
          + "hostname"      = "bdii.ipb.ac.rs"
          + "in_production" = "Y"
          + "monitored"     = "Y"
          + "notifications" = "N"
          + "primary_key"   = "1183G0"
          + "roc"           = "NGI_AEGIS"
          + "sitename"      = "AEGIS01-IPB-SCL"
        }
      + name       = "top-bdii"
      + node       = "EGI"
      + port       = 2170
      + service_id = "top-bdii_AEGIS01-IPB-SCL-1183G0"
      + tags       = [
          + "AEGIS01-IPB-SCL",
          + "NGI_AEGIS",
          + "EGI",
        ]

      + check {
          + check_id                          = "service:AEGIS01-IPB-SCL-1183G0"
          + deregister_critical_service_after = "720h0m0s"
          + interval                          = "1m0s"
          + method                            = "GET"
          + name                              = "AEGIS01-IPB-SCL top bdii check"
          + notes                             = "AEGIS01-IPB-SCL TCP check"
          + status                            = (known after apply)
          + tcp                               = "bdii.ipb.ac.rs:2170"
          + timeout                           = "20s"
          + tls_skip_verify                   = true
        }
    }
{% endhighlight %}

The apply operation added 83 resources in just under 18s.
Below is a screencast of the events in Consul while the services are registered, and eventually become healthy.

<div class="video">
<video width="100%" controls>
<source src="{{ site.url }}/images/service-registration-in-consul.webm">
</video>
</div>

As you can see, they are tagged as registered by Terraform in the EGI external node.
The Nomad job running the external monitor takes a few seconds to come up and perform the monitoring checks which eventually makes healthy service instances go green.

<div class="video">
<video width="100%" controls>
<source src="{{ site.url }}/images/consul-esm-in-nomad.webm">
</video>
</div>

So, in a few seconds, we have both registered the external services, and are monitoring them with a basic tcp liveness check.
Let's see if this makes any difference to a user.

### I promised you UX improvements

Now, imagine I'm member of the alice VO and I want to find a top-bdii:

{% highlight console %}
host alice.top-bdii.service.consul
alice.top-bdii.service.consul is an alias for topbdii.grif.fr.
topbdii.grif.fr is an alias for lpnhe-topbdii.in2p3.fr.
lpnhe-topbdii.in2p3.fr is an alias for lpnhe-gs9013.in2p3.fr.
lpnhe-gs9013.in2p3.fr has address 134.158.159.13
lpnhe-gs9013.in2p3.fr has IPv6 address 2001:660:3036:197:134:158:159:13
{% endhighlight %}

Consul creates DNS entries for all services and their tags in its catalogue, and only returns healthy instances.
Since it's DNS, there's automatically round-robin so we don't risk hitting a given instance too hard.

Since we've tagged services with their site name as well as NGI and ROC names, we can also find local, national or regional instances:

{% highlight console %}
host ngi_france.top-bdii.service.consul
ngi_france.top-bdii.service.consul is an alias for lapp-bdii01.in2p3.fr.
lapp-bdii01.in2p3.fr has address 134.158.84.162
lapp-bdii01.in2p3.fr has IPv6 address 2001:660:5310:420:7::1
{% endhighlight %}

### Discussion

Of course, I'm hiding a few details from you here, dear reader[^agent], but bear with me -- we are looking at this from the user's point of view.
If a community decided that they wanted to use a modern stack, but use some of EGI's federated services, they could use this approach to discover infrastructure services.
This could greatly improve the performance of workflow engines for example which need to keep an up-to-date list of healthy compute endpoints.
The last time I touched this problem, it was done using either a hardcoded list of endpoints or an unweidly and unreliable GIIS lookup.
Being able to find things just by using DNS seems to me a much better approach.

I'm also looking only at Top-BDII services here.
I decided to start there because it was easy to write a health check for it and I'm pretty much guaranteed that these will be open to the world.
It seems a bit redundant to put service discovery systems (top-bdiis) into another service discovery system (Consul).
We could replace the GOCDB query and find UIs in the same way though, which might be a bit more useful eventually.

Another point is the combination of service discovery and service availability checks.
I mentioned above that the GOCDB only declares the desired state of the world, but Consul adds to that by including a current state check.
There is no historical and statistical  information in this, so it's by no means a replacement for something like ARGO.
However, if you have a piece of infrastructure which needs to query a service topology in order to configure itself, this is a better way to go.

In conclusion, the whole concept of external services is really useful here.
I can easily envisage a scenario where a community comes up with a set of applications which gets deployed into its platform, but needs to augment them with infrastgructure or compute and data services from the federation.
This fun little experiment shows just how easy it is to terraform the federation into your environment.

I'm not proposing anything radical here, but I am intrigued by the idea of including the _entire_ federation into a set of peered Consul datacentres, replacing the entire bdii infrastructure with a combination of Consul's service mesh and key-value store.
I have a sneaking suspicion it would be quite handy in creating the controls which are required to satisfy the FitSM CONFM process requirements.
Consul's documentation says it should be able to scale effectively... but I'm more interested in entirely eliminating the need for site information services and using it as a distributed source of truth for configuration items.

For now it's just a thought experiment, but I really want to scratch that itch.-- I look forward to extending the approach to see what else we can do :star:

---

[^FitSMCONFM]: Taken from the FitsSM standard, section on Configuration Management Process
[^TFNote]: Note that we will be using environment variables to configure the Consul provider (address and token). The backend is not declared here, but in the actual instantiation, the backend is also Consul.
[^ExternalNode]: I thought about binding these declared external services to actual exsting nodes, and then requesting that the monitor run on those nodes, but there is currently no operator loop between terraforming the services and the node state. Nodes could therefore potentially fail, taking the services registered on them with them, so I  decided on the "fake" external node.
[^ARGOdoesthis]: This kind of script check is exactly what the Nagios-based ARGO monitor does.
[^agent]: First of all, I've got my environment set up to be able to use the Consul DNS by having an agent running locally.
