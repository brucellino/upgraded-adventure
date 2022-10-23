---
title: Issuing host certificates from Vault with Ansible
description: A few remarks on perfecting part of a configuration layer.
category: Practice
tags:
  - Ansible
  - ContinuousDelivery
  - Vault
---

- [Overview](#overview)
- [The Vault CA](#the-vault-ca)
- [Issueing certificates](#issueing-certificates)
- [Deciding on whether to issue a certificate](#deciding-on-whether-to-issue-a-certificate)
- [Declaratively determining `issue_cert`](#declaratively-determining-issue_cert)
- [Further enhancement](#further-enhancement)
- [Footnotes and References](#footnotes-and-references)

## Overview

I recently released a new [Ansible role](https://github.com/brucellino/ansible-role-base-platform-pi/releases/tag/v1.0.0) which is responsible for providing a base layer of configuration machines in a cluster.

This layer is responsible for preparing the machine to host other services, but without depending explicitly on them and as such, it is a somewhat _abstract_ component.
It can be applied to any machine by a controller which has access to the
In particular, this release deals with the issuing of a X.509 certificate to a machine in order to allow it to communicate securely with infrastructure services such as the service mesh or orchestration services[^ConsulNomad].

## The Vault CA

The certificate chain we are provisioning is managed by [Vault](https://vaultproject.io)[^VaultBlog].
We have configured our Vault instance with an Intermediate CA, following the [Vault "Build your own CA" tutorial](https://developer.hashicorp.com/vault/tutorials/secrets-management/pki-engine).
I had previously encoded this as a [Terraform module](https://registry.terraform.io/modules/brucellino/ca/vault/1.1.0), more as an exercise than anything else.

The real state is currently defined in [`github.com/brucellino/vaultatho.me`](https://github.com/brucellino/vaultatho.me/blob/main/hashiatho.me-pki.tf). Most importantly, the Vault PKI secret backend role resource is defined there:

{% highlight hcl %}
{% raw %}

# H@H Intermediate CA role

resource "vault_pki_secret_backend_role" "hah_int_role" {
  backend = vault_mount.hah_pki_int.path
  name    = "hah_int_role"
  key_usage = [
    "DigitalSignature",
    "KeyEncipherment",
    "KeyAgreement"
  ]
  allowed_domains = [
    "*.service.consul",
    "*.node.consul",
    "*.node.dc1.consul",
    "*.hashiatho.me",
    "*.station"
  ]

  allow_bare_domains = true
  allow_subdomains   = true
  allow_glob_domains = true
  allow_ip_sans      = true
}

{% endraw %}
{% endhighlight %}

Calls to this endpoint with a valid token would result in the issueing of a certificate to the caller.

## Issueing certificates

Before describing exactly how we manage to deliver the certifiate to the host, let's consider some details.
We have a situation where a _controller_ is applying a configuration to an entity in our inventory.
The controller has access to a secret which allows it to authenticate to Vault, particularly the token allows calls to issue new certificates, _i.e._ the [`/pki/issue/:name` endpoint](https://developer.hashicorp.com/vault/api-docs/secret/pki).

Our controller is actually a machine running an Ansible playbook.
I had the choice of one of two Ansible modules:

- [`ansible.builtin.uri`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html#ansible-collections-ansible-builtin-uri-module)
- [`community.hashi_vault.vault_pki_generate_certificate`](https://docs.ansible.com/ansible/latest/collections/community/hashi_vault/vault_pki_generate_certificate_module.html#ansible-collections-community-hashi-vault-vault-pki-generate-certificate-module)

In the first case, we would have to call the API endpoint directly and pass several parameters in the header to request the certificate, whilst in the second, we could call the wrapper module with a similar set of parameters.

In this case, I opted for the latter, with something like this:

{% highlight yaml %}
{% raw %}
- name: Issue certificate to host
  when: (issue_cert | bool)
  block:
    - name: Issue cert from Vault
      community.hashi_vault.vault_pki_generate_certificate: # noqa syntax-check
        role_name: hah_int_role
        common_name: "{{ ansible_fqdn }}.node.consul"
        engine_mount_point: "pki_hah_int"
        url: "{{ lookup('env', 'VAULT_ADDR') }}"
        token: "{{ lookup('env', 'VAULT_TOKEN') }}"
        alt_names:
          - "{{ ansible_hostname }}.node.consul"
          - "{{ ansible_hostname }}.hashiatho.me"
      register: cert_data
{% endraw %}
{% endhighlight %}

## Deciding on whether to issue a certificate

The astute reader will not the the task is conditional: `when: (issue_cert | bool)`. This is a check on a boolean fact which we use to determine whether we should issue a certificate (`true`) or not (`false`).

How do we know when to set this value to `true`?
Surely we should issue a new certificate _when a new certificate is required_.
Decomposing this statement, a new certificate is required when:

1. There is no private key present
1. The public certificate is invalid
    1. issued to incorrect host or hostname changed
    1. corrupted file
    1. expired

In principle, if the private key is present, but the certificate is not valid for some reason, we could look up the public key and CA data in Vault, as long as we knew the serial number of the certificate, but simply revoking the cert and issueing a new one is far simpler.

### Declaratively determining `issue_cert`

There is a lot of logic in the process of determining whether to issue the certificate anew.
The first few runs of the playbook which tests this role resulted in hundreds of new certificates, one for each run, since the Vault PKI endpoint isn't a stateful service.

On the other hand I could have implemented the logic in a big script which did all the computing... but this didn't feel like the right thing to do and would nevertheless result in a lot of extra work.

In the end, I settled on an approach using a few invokations of the `stat` module and setting facts on the fly:

First, we check all of the certificate files:

{% highlight yaml %}

- name: Stat cert files
  ansible.builtin.stat:
    path: "/etc/tls/hashi@home/{{ item }}.pem"
  register: stat
  loop:
    - certificate
    - private_key
    - issuing_ca
{% endhighlight %}

We get back a large dictionary which we can query to see when the stat on files shows that they are not present:

{%- highlight yaml -%}
{% raw %}
- name: Set issue_cert
  delegate_to: localhost
  ansible.builtin.set_fact:
    issue_cert: "{{false in (stat.results | community.general.json_query('[*].stat.exists')) }}"
{% endraw %}
{%- endhighlight -%}

If the certificate files are all present, we can enter the decision branch where we check decide to issue the certificate based on the validity of the existing certificate.
We perform a `x509_certificate_info` and get back the cert info.
If it's valid, `issue_cert` is set to `false`.
If not, not only is `issue_cert` set to `true`, but we also remove the corrupt or invalid files:

{% highlight yaml %}
{% raw %}

- name: Check Certs
  when: false not in (stat.results | community.general.json_query('[*].stat.exists'))
  block:
    # If this fails -- either if the cert is not present or if it is not a valid cert
    # Then the rescue is invoked
    - name: Get cert facts
      community.crypto.x509_certificate_info:
        path: /etc/tls/hashi@home/certificate.pem
      register: cert_info
    - name: Set expired fact
      ansible.builtin.set_fact:
        issue_cert: cert_info.expired
  rescue:
    - name: Remove Corrupt Cert
        ansible.builtin.file:
          path: "/etc/tls/hashi@home/{{ item }}.pem"
          state: absent
      loop:
        - certificate
        - private_key
        - issuing_ca
    - name: Set issue_cert fact
        ansible.builtin.set_fact:
          issue_cert: true
{% endraw %}
{% endhighlight %}

After all of that, we finally have a good value for `issue_cert`, which is used, as shown above, to determine whether or not to issue a new certificate and deliver it to the host:

{% highlight yaml %}
{% raw %}

- name: Issue certificate to host
  when: (issue_cert | bool)
  block:
    - name: Issue cert from Vault
        community.hashi_vault.vault_pki_generate_certificate: # noqa syntax-check
          role_name: hah_int_role
          common_name: "{{ ansible_fqdn }}.node.consul"
          engine_mount_point: "pki_hah_int"
          url: "{{ lookup('env', 'VAULT_ADDR') }}"
          token: "{{ lookup('env', 'VAULT_TOKEN') }}"
          alt_names:
            - "{{ ansible_hostname }}.node.consul"
            - "{{ ansible_hostname }}.hashiatho.me"
        register: cert_data

    - name: Deliver certs
      ansible.builtin.copy:
        dest: "/etc/tls/hashi@home/{{ item }}.pem"
        content: "{{ cert_data.data.data[item] }}"
        mode: 0644
        owner: root
        group: root
      loop:
        - certificate
        - issuing_ca
        - private_key

{% endraw %}
{% endhighlight %}

## Further enhancement

This approach can safely and reliably issue an x.509 credential chain to a host from the Vault CA via the controller.
It takes care of checking whether a certificate needs to be issued before actually calling the CA for a new cert, but there are a few aspects can be further improved in later versions.

First of all, we can call the [PKI endpoint to tidy the certificate store](https://developer.hashicorp.com/vault/api-docs/secret/pki#tidy) when a certificate is determined to be invalid.
Even more correctly, we should probably [_revoke_](https://developer.hashicorp.com/vault/api-docs/secret/pki#revoke-certificate) a certificate if we are issueing  new one, so that we can update the CRLs.
This can probably be quite easily implemented as an Ansible handler, but the cert serial number is required in order to call the revokation function.
This seems like a good case to use the Consul KV store (or indeed the Vault KV store) to keep host names (keys) and cert serial numbers (values) easily retrievable, rather than having to do a filter on a big list of serial numbers returned by [`/pki/certs`](https://developer.hashicorp.com/vault/api-docs/secret/pki#list-certificates).

For the full role, see [@brucellino/ansible-role-base-platform-pi](https://github.com/brucellino/ansible-role-base-platform-pi), and open an issue if you would like to discuss!

## Footnotes and References

[^ConsulNomad]: In my case, these are Consul and Nomad.
[^VaultBlog]: See the [Vault blog](https://www.hashicorp.com/blog/certificate-management-with-vault)
