---
title: Patroni for postgres clusters on prem
layout: post
description: Postgres clusters for platform customers.
category: lab-notes
toc: true
tags:
  - blog
  - platform-engineering
  - data-plane
  - packer
  - nomad
  - consul
  - postgres
  - patroni
mermaid: true
---

A common pattern in deploying applications is ensuring they have a reliable persistence layer.
This often takes the form of a database.
If an application, or rather the service owner which has to deliver that application, can assume that it can connect to a database via configuration parameters, this reduces the scope of the application deployment.

The problem then gets shifted to the platform owner to ensure that there is indeed a database server for the application ready when it's deployed.
In a scenario where there are several applications deployed into the platform, the platform owner now has a set of design choices to make:

* Do we have a single database server serving all applications in a given scope, or does each application get its own databse server?
* Do we deploy database servers in the same scope as the application, or outside of the application scope?
* Does database management lie with the service owner or with the platform owner?

The correct choices will probably depend from platform to platform.
From my point of view, the important thing is to have options available to be able to even pose the questions in the first place.

## Managed databases on prem

The platforms I own now are all custom-built, on-premise.
There are no platform functions other than those I build into them, which reveals many of the ways a platform engineer needs to make design choices. 
Coming from cloud-native environments where databases are available as a service, I had to go back to the drawing board.

First, I had to recognise that the only storage I have available is the block storage attached to virtual machines which constitute the Nomad agents.
In some cases, these attached disks could not even be requested at agent launch time, but had to be added when the VM itself was provisioned - either you had an attached disk or you didn't and you couldn't be sure across the fleet whether was the case.
However, one has to start with _some_ basic assumptions.
I assumed that I could request certain agents which would have attached block storage, and that block storage would be reliable: it would be snapshotted, backed up, _etc_.

I wanted a way to bring up a reliable database service, as a Nomad job.
It should be able to have servers spread across several nodes to protect against single node failures, and provide the ability to do rolling updates across nodes, taking them out of the cluster temporarily, then re-joining.

All of this is possible of course, and probably well-trodden paths for a database administrator, but I am not the DBA!
I wanted something that DBAs have refined over the course of their collective years of experience into something which would allow me to perform their magic without necessarily understanding the minutae.

Some research lead me to Patroni.

# Patroni fits into my platform

Since these are intended as lab notes, I will skip the various alternatives which were discarded.
Patroni is a management layer for Postgres.
These are my words; they call it a "template for HA Postgres".
It's been around for more than 10 years at the time of writing, and is used by a few big names _e.g._ EnterpriseDB -- let's just say that's a good signal in my book!
However Patroni had one key design consideration which fits into my idea of a platform -  distributed configuration.
It uses a DCS (distributed configuration store) to managed postgres deployments _externally_.
The DCS is the source of truth for the database.
This in turn is kept in a kv store which is replicated across a Raft cluster.
This KV store is used not only to write configuration data, but also agree on the leader or primary, since it is the one with a lock on the configuration.
If the primary fails, it loses the lock on the configuration, and an election is triggered, resulting in one of the replicas being promoted to the new primary (leader).

It should be noted that what sealed the deal for me was the built-in ability to use Consul as a DCS.

## A little bit of engineering

Patroni is a python library; it is the application which manages Postgres.
I consider it to be a kind of postgres harness -- but you still need the postgres binary.

In order to deploy it in my platforms, it would run as a Nomad job, where the task driver used would inevitably be one of the container runtimes, Docker or Podman.

### Images

I first needed to have containers that actually had patroni in them.
I considered using pre-start tasks in the default postgres containers to install patroni, but opted instead of building my own containers with patroni pre-installed.
I wrote a [Packer template](https://github.com/brucellino/packer-patroni-postgres) to do this, with essentially only two changes:

1. Add the dependencies to the image, according to the [Patroni installation documentation](https://patroni.readthedocs.io/en/latest/installation.html)
2. Change the entrypoint of the container from its defalt to `/patroni/bin/patroni`.

The Packer template has two builders for the most common architectures in my platforms - AMD64 and ARM64.

### Deployment

We have all of the ingredients necessary to deploy as many clusters as we like:

* Consul cluster for DCS
* Patroni-ready postgres images
* Nomad orchestrator

The final step would be to write the Nomad job and deploy.
I took the case where we would need a default database server as a fundamental dependency for platform services themslves.
This would serve several databases to platform service rather than platform customers, which would probably merit having their own deployments within their namespace, scope, _etc_.

The first nice thing is that we can simply declare a capacity using the [Nomad group `count`](https://developer.hashicorp.com/nomad/docs/job-specification/group#count) paramter; this opens up possibilities of scaling later.

The group also has its own network, where we declare the Patroni API port as well as the postgres port of course:

{% highlight hcl %}
network {
      # Expose the patroni API. This listens on 8008 on the container
      port "patroni_api" {
        to = 8008
      }
      # A dynamic port is reserved for postgres
      port "postgres" {}

    }
{% endhighlight %}

Each instance of the task in the group should get its own service registered in Consul.
I made the mistake in initial attempts to declare the service at the level of the group rather than the task, which ended up giving routing errors.
This allowed each instance of Patroni in the group to have a unique service ID, and was necessary to get the DCS leader election to work.

We declare the patroni service with its healthcheck and the postgres service in the `group.task.service` block:

{% highlight hcl %}
service {
  name     = "patroni-api-${var.tenant}"
  port     = "patroni_api"
  provider = "consul"

  check {
    name     = "alive"
    type     = "tcp"
    interval = "10s"
    timeout  = "2s"
  }
  check {
    name     = "healthy"
    type     = "http"
    interval = "10s"
    timeout  = "2s"
    path     = "/healthy"
  }
}

service {
  name     = "postgres-${var.tenant}"
  port     = "postgres"
  provider = "consul"

  check {
    name     = "alive"
    type     = "tcp"
    interval = "10s"
    timeout  = "2s"
  }
}
{% endhighlight %}

Here `${var.tenant}` is a Nomad variable to distinguish who this postgres cluster instance belongs to.

The next tricky part was writing the `patroni.yml` configuration template.
Each patroni instance needed to be unique within a given scope in order to perform leadership elections.
For example, in a 3  node cluster (1 primary, 2 replicas), they all need a `patroni.yml` file which differs by the name of the instance, but has the same namespace and scope, as well as allows each instance to communicate with each other.

I decided to configure them with the scope as the name of the tenant, a namespace selected by hand (`operations`) and using the Nomad allocation runtime variables as identifiers:

{% highlight yml %}
scope: ${var.tenant}
namespace: operations
{% raw %}name: {{ env "NOMAD_ALLOC_NAME" }}-{{ env "NOMAD_ALLOC_ID" }}{% endraw %}

{% endhighlight %}

That would end up being templated as 

{% highlight yml %}
scope: default
namespace: operations
name: patroni-postgres.patroni[0]-8ae3416a-2a73-a3e0-ea10-9d50596a442d
{% endhighlight %}

for the first (`[0]`) instance of the group.

We configure the Patroni REST API to be served at the task's specific URL:

{% highlight yaml %}
{% raw %}
restapi:
  connect_address: {{ env "NOMAD_ADDR_patroni_api" }}
  listen: {{ env "NOMAD_ADDR_patroni_api" }}
{% endraw %}
{% endhighlight %}

Then we come to the tricky part: the postgres configuration.
This was tricky because I know almost nothing about configuring postgres servers.
There were however only two configurations I needed to actually think about.
The first was the users and their authentication when bootstrapping and managing the clusters.
Patroni defines 3 users which it uses to manage postgres behind the scenes: `superuser`, `rewind` and `replication`.
These should all have different credentials.

I had stored them already in the platform Vault as secure KV pairs, so that was used in the template with a Vault lookup: `{{- with secret "hashiatho.me-v2/data_plane" }}`.
Again the Nomad runtime variables gave me the port which postgres was served on in the deployment: `{{ env "NOMAD_ADDR_postgres" }}`

{% highlight yaml %}
{% raw %}
postgresql:
  authentication:
{{- with secret "hashiatho.me-v2/data_plane" }}
    replication:
      password: "{{ .Data.data.postgres_replication_password }}"
      username: {{ .Data.data.postgres_replication_user }}
      sslmode: allow
    rewind:
      password: "{{ .Data.data.postgres_rewind_password}}"
      username: {{ .Data.data.postgres_rewind_user }}
      sslmode: allow
    superuser:
      password: "{{ .Data.data.postgres_root_password }}"
      username: "{{ .Data.data.postgres_root_user }}"
      sslmode: allow
  bin_dir: ''
  connect_address: {{ env "NOMAD_ADDR_postgres" }}
  data_dir: /var/lib/postgresql/17/data
  listen: {{ env "NOMAD_ADDR_postgres" }}
  parameters:
    password_encryption: md5
  pg_hba:
    - host all all 0.0.0.0/0 md5
    - host replication {{ .Data.data.postgres_replication_user }} 127.0.0.1/32 md5
    - host replication {{ .Data.data.postgres_replication_user }} 127.0.0.1/32 md5
    - host replication {{ .Data.data.postgres_replication_user }} 192.168.1.0/24 md5
{{- end }}
{% endraw %}
{% endhighlight %}

To be honest the `pg_hba` key was the result of trial and error, as well as a bit of good old searching on errors, but essentially, I needed to add the local network CIDRs which the replication user would communicate on.

# Operations

So, now we can deploy postgres clusters with reliable replication, using Consul to manage leadership election.
We can scale them with a single parameter (`count`) and manage all of the configuration centrally from the Consul KV.

There are many interesting aspects to look into next, especially from an operations point of view:

* Can I update recovery and replication configuration just by updating entries in Consul?
* Can I perform blue-green deployments and ensure that all data is replicated during the deployment? Will this obviate the need for persistent disks in certain cases?
* How can I use the Vault database secrets provider to provision one-time credentials for database users?

I have also already had a few failures of database nodes in prod on one of our platforms, where essentially replicas were not rebuilt after a downtime due to divergence in history.
Whatever that means - I have no idea really.

There is also the need to actually back up databases and define point-in-time snapshots so that we can reproduce deployments elsewhere, e.g. during CI/CD or testing procedures.

I now feel like I understand Patroni well enough that I can use it to solve these needs, perhaps adding a few other tools like Barman.
The data plane of the platform is taking fine shape!
