---
title: Managing security and risk in a complex system
layout: post
description: Tools and methods for maintaining risk posture and managing exposure in a system with thousands of dependencies.
category: methods
toc: true
tags:
  - blog
  - platform-engineering
  - security-plane
  - dependency-management
  - risk-management
mermaid: true
---

* TOC
{:toc}

## Problem Statement

In this post, we'll tackle a common problem faced by those responsible for shipping software for a client: ensuring that software supply chain is kept secure and compliant with requirements of the contract.


## Setting the stage

Let's say you're building a set of software services for a client, which takes the form of a product composed of multiple services, managed on behalf of the client.
The goal of this set of services is to provide access to a defined set of cloud services, data sets and other APIs, all of which are also part of your product.
Now, let's say that the client intends to release this product for use by third parties and in order to do that, they need to state with certainty that the product is secure and compliant with all relevant regulations and standards.

It is safe to assume that each of these services is implemented directly using one or another programming language, which inevitably depend on a set of libraries or frameworks.
These direct dependencies may themselves also depend on other libraries, packages or modules.
In reality then, the system is composed not just of the specific implementation of services, but also all of this graph of dependencies.

Given this understanding of the composition of the system, of known, assumed and unknown dependencies, let us consider what it would take to keep the system safe.
This would, at a minimum, require the ability to observe the full composition of the system, as well as the presence of known vulnerabilities in the dependencies.

## Seeing Risk

Let's assume we can actually see all of the dependencies: how would we achieve that?
It would involve some tool which inspected the system and extracted the graph of all of the dependencies.
This is only feasible when the system _declares_ what it depends on.
This varies from case to case:
  - In the case of an application, this will come in the form of a language-specific manifest file, such as a `requirements.txt` for Python, a `go.mod` for Go, or a `package.json` for Node.js.
  - In the case of a service, this will come in the form of a service-specific file, such as a `Dockerfile` a Docker Compose file for Docker, a manifest file or Helm chart for Kubernetes, or the providers section of a Terraform configuration.
  - In the case of a virtual or bare-metal machine, this will come from interrogating the filesystem for installed packages or modules.

Scanning these various endpoints, whether they be package managers, deployment manifests, or actual deployed environments, provides us with a list of discovered versions and their versions.
This is the first step in being able to see the risk of the system: the simple ability to generate an inventory.
The key capability however is being able to take **appropriate action** based on whether a given component is known to contain a given vulnerability.
Vulnerabilities do not inherently carry risk, however, since they must be exploitable in order to pose a threat, and whether they are exploitable or not depends on several factors, including the configuration and deployment of that component in the actual system.

Nonetheless, the ability to map CVEs[^cve] to components allows us to locate points of investigation, and perhaps even keep an audit trail of the system's posture.

## Owning Risk

The system is not a flat, amorphous blob of components.
There is a structure to the system which comes from the way in which humans have decided to organise their work around building, maintaining and deploying it.
In some cases, there will be a single system owner, perhaps an enterprise architect, who has the final responsibility for the system's security posture.
In others, each service in the system will have self-contained ownership, and agree to a common set of policies in order to guarantee a consistent security posture in the system even though they take action on known vulnerabilities autonomously.
The point is that it in order to ensure that the system as a whole is secure, it is necessary to know who should take action on which component.
**Ownership** is thus the ability to map vulnerabilities to components, _via_ services to actual people who take responsibility for action.

## Where have I seen this before

Way back in before 2020 some time, I forget exactly when, I was working in the SANREN group at the Council for Scientific and Industrial Research (CSIR) in Pretoria, South Africa.
SANREN stands for[ "South African National Research and Education Network"](https://www.sanren.ac.za), and aside from provisioning high-capacity fibre connections throughout South African research institutions and universities, it also had the mandate to deliver services to these customers.
One of these services was the South African National Grid, which I was responsible for, which coordinated the distributed computing infrastructure offered by the universities and laboratories connected to the SANREN network.
However, SANREN was very active in providing [end-user services](https://www.sanren.ac.za/services/) such as the identity federation, on-demand file transfer between individuals, _etc_.
Most of these were simply instantiations of a product commonly-used by peers in the research networking community.
These surely provided some value to the communities served by the infrastructure, but they were too generic to be considered invaluable.
The real wins were where we could give actionable advice to customers.

### Providing value to customers closes the feedback loop

I will permit myself a brief digression here to consider how infrastructures are perceived and how they become sustainable.

The question can be posed as such:

> How can invisible infrastructures keep proving their value to customers?

An infrastructure works well when it becomes invisible; nobody sits and thinks about how the network provides value to them, they just use the things which the network enables.
This invisibility is something which often breaks the feedback loop between customer and provider, since when things are working well, the value is invisible, and only when the infrastructure fails to do its job does it become visible.
By its very nature, it is only perceived negatively.

A second consideration is whether there is a direct feedback loop between the end users of the services offered by the infrastructure and those who actually pay for it (the customer).
In the absence of such a loop, the sustainability of the infrastructure, depending on its continued viability, requires arduous measurement and reporting of the customer satisfaction and usage metrics, which inevitably drives up the cost and reduces the efficiency.
Instead, what if the infrastructure provider itself provided services _directly to the customer_?
In SANREN's case, these are typically the Information Technology departments of the institutes served by the network[^asaudit].
These IT organisations are comprised of IT professionals, with their budget directly allocated to IT services _for the institute_.
Part of their mission is usually to protect the network and IT resources associated with them, either within their institute or across the network in the case of research and scientific collaborations.

Assessing and owning risks was one of those activities which simply could not be performed by the respective IT departments, given their limited resources and vast set of users.
If SANREN could provide them with a service to assess security risks and provide specific actionable recommendations assigned to specific people this would have gone a long way towards helping them perform their overall mission.
This would have been a service provided directly to the customer, providing a strong feedback loop between the invisible infrastructure and those who ultimately depend on it.

It was my buddy Schalk Peach way back around 2017-2018 who helped me understand how powerful this feedback loop between vulnerability, component and person would be.
He was a postgrad student at the CSIR SANREN team during my last few years there, and we had almost daily conversations about the workflows and entity models that would be involved in providing a service like this.
At the time, we were focussed on perimeter security and directing customer network administrators to specific mitigation actions based on the unintelligible vulnerability reports from [Nessus](https://www.tenable.com/products/nessus)[^nessus].

Every now again I think about him and how far ahead of the curve he was.

## Putting Dependency Track to use

Now I'm part of the team delivering a managed service to researchers on behalf of the European Commission, and we are facing the same challenges as a decade ago. 
A customer like the EC does not mess around when it comes to compliance and quality, and the system we were building is delivered by a great number of independent contractors, so I knew that assuring all interested parties that the system was secure and up to date would require specific tooling.
Indeed at the outset my mind flew right back to Schalk and his magical vulnerability-remedy system... if only we had something like that to observe and track our dependencies!

Well, it turns out that in the decade or so since I last bothered myself with platform security, the OWASP foundation has brought a great many projects to maturity, including one called ["Dependency Track"](https://dependencytrack.org).

### Tracking dependencies, measuring risk, ensuring compliance

Dependency Track's strength comes from its ability to keep a detailed database of Software Bill of Materials (SBOM), as well as a local database of all known vulnerabilities.
Vulnerabilities are almost always published with an associated component, so all we needed to do was declare the components of our system, and their associated SBOMs.
Dependency Track also has an overlay system which allows one to declare that certain projects are owned by certain teams, recreating that all-important feedback loop we mentioned repeatedly before.

This functionality allowed security teams at a glance to know which vulnerabilities were present where, and indeed what the relevant service owner was doing about.

We ended up bolting on a custom notification handler to Dependency Track to allow us to accurately communicate the discovery of vulnerabilities both with the client and the security team.

### Services in the Security Plane of the platform

In the larger context of providing a [platform providing support functions for the delivery of a wide set of services](https://platformengineering.org/platform-tooling), Dependency Track falls cleanly into the "Security Plane".
The final goal is not measurement and visibility, but actually taking remedial action, we should envision the connection of task and issue tracking services with the dependency tracker.
However, dependencies of software components are not the only source of risk and security vulnerability in the system.
Misconfiguration and insufficient resources can also contribute to security risks, or risks of denial of service.
These are detected with other tools, such as penetration and load testing tools.
These findings too are associated with specific components, owned by specific people in the organisation.

This could be done with [DefectDojo](https://defectdojo.com/), another tool in the OWASP stable.

These of course only address the system design and architecture.
There are of course also the operational aspects, the day-to-day events happening in the system, and the inevitable attacks it will face if connected to the internet.
To this end a security information and event management (SIEM) system is required, where alerts could also be passed to the overall view of the security posture, potentially correlated with known vulnerabilities and outdated components.

## Conclusion

We have gone to great lengths to find a set of tools which would allow teams delivering software to declare their composition and dependencies to a central service, and in turn obtain visibility and actionable advice on how to respond to known vulnerabilities.
This could be deployed in the context of platform engineering within the "security plane", offering a platform-level internal service to internal customers wishing to deploy in the platform.
All of the tools discussed here are open-source and deployable on-prem.
Indeed I have deployed them in a Nomad cluster built for the purposes of providing the platform-level functions to our client, as well as a similar deployment in EGI internally.
We gain insight into component-level risk, the ability to map that risk to actual people, and the ability to provide them with specific concrete advice on how to mitigate the risk.
With the addition of a SIEM, we would be able to have real-time observability into security events, and thus ensure not only the architectural security of the system, but also its operational security.

Total visibility and collaboration between platform engineers, SRE and service owners is the goal, and we are within its reach.


---

# Footnotes and References  

[^cve]: Common Vulnerabilities and Exposures (CVE) is a dictionary of publicly known information security vulnerabilities and exposures. CVE is maintained by the MITRE Corporation.
[^asaudit]: The Association of South African University Directors of IT (ASAUDIT) was in my understanding the interface between the infrastructure provider and the institutes -- it has apparently transformed into [Higher Education IT South Africa](https://heitsa.ac.za/)
[^nessus]: I think it was Nessus, but it was a while ago. I do remember that it was open source at the time, whatever the tool was.
