---
title: Modelling onboarding
layout: post
description: To seek perfection, you must see the big picture
category: egi
toc: true
tags:
  - blog
  - rpa
  - aai
  - signup
mermaid: true
---

<small>_This is a discussion article for work done in the context of EGI user support._</small>

## So, you want to run a training event

There are some events which you need to run repeatedly, where the workflow is more or less constant, even though the specific details of each instance may change.
One of the north stars of engineering in our line of work is to **eliminate toil**[^eliminate_toil]: work that is

- **manual**
- **repetitive**
- **automatable**
- **tactical**
- **no enduring value**

In this case, we'll take a look at preparation for training events from a perspective of eliminating toil, and see what the effects might be on participant experience, as well as overhead in running the event.

### Treating Training Environments as Products

When a user comes to a training event, whether we like it or not, whether we realise it or not, we are selling them an experience, before they actually use the service.
If that experience is a good one, and if they perceive that the service they are using actually provides _utility_, then they will come back and do whatever is necessary

A lot of ink has been spilled regarding _developer experience_ (DX), and how to improve it, culiminating perhaps in the practice of _platform engineering_.
One of the main lessons that has been learned is that in order to be successful, the platform needs to be designed _as a product_, making it appealing and engaging for the end users, and creating _"golden paths"_ for them.

With this as backdrop, let's see if we can apply these ideas to a training event where we are exposing potential users to a service for the first time.
A traditional training event might start at "zero" and walk the user through all of the initial steps necessary to obtain access to the platform, eventually getting to the point where they have access to the actual service they want to use.

Let's take a look at what might be considered a "good" user journey -- one which attempts to demonstrate the value (utility) of the service to the user as soon as possible:

<div class="mermaid">
---
title:  Demonstrating service utility during training
config:
  journey:
    actorColors:  ["#59c9a5", "#465775"]
---
journey
  section Awareness
    Discovers service via EGI page:  5:  User
    Registers to event:  6:  User
  section Access
    Has pre-prepared credentials:  4: Operator
    Pre-registered in group / VO: 4:  Operator
    Accesses service:  5:  User
</div>

Here, we split the journey into two sections -- `Awareness` and `Access`, showing the user emotion from positive (high) to negative (low) as they complete their journey to accessing the service[^User_Research].
The goal is to get the user in front of the service as fast as possible, with as little distraction as possible.
In order to achieve this, however, we have created an artificial scenario where the user is _assuming an identity_ which we have already created and approved in the group.
We are hiding complexity from the user in order to use the little time we have in contact with them to greatest benefit.

**Our goal is to deliver value to them, their goal is to conduct their research**.

We need to spend every moment of the time they have dedicated to spend with us towards convincing them that our service can help them reach their goal.

As you can see though, this requires some intervention on the part of the operator in that they need to provision the users and the training environment which the attendees of the tutorial will use.
As it turns out there is already a good recipe for creating the training environment, but what about provisioning the users?

### The invisible barrier

This has until now been a source of toil -- or rather the toil has been rejected and implicitly pushed onto the users.
We know that there is actually a huge barrier for them to accessing our services, the **Authentication** and **Authorisation** of the user.
This barrier is there by design in order to place access controls and service level quotas on the services we provide.
The main problem is that it creates a huge mental separation between _us_ and the _attendees_ since we have already been granted access to the service, while they have not.
We don't feel the disillusionment and pain of having to overcome that barrier, while they do -- and often it is the first emotion they have when trying our services.
Let's take a closer look at that Access section, as it stands for the average first-time user.

<div class="mermaid">
journey
  section Access
    Requests service:  8: User
    Selects IdP: 6:  User
    Logs in IdP:  5:  User
    Info Release IdP: 4:  User
    Info Release SP: 3: User
    Access Denied:  1: User
</div>

Since the trainer doesn't know who the users are, or which IdP they will use to log in, they can't pre-approve _actual real people_, so the user journey that starts at the service login page is a **dead-end**.

In fact, the user needs to first be enrolled in a group or [Virtual Organisation (VO)](https://confluence.egi.eu/display/EGIG/Virtual+organisation), and thus authorised to access the service.
What is more, they need to be authorised to access that service _in a specific context_ -- _i.e._ as a member of a specific group or VO.

Taking stock of this means understanding that there is actually a whole other journey for the user to take -- entering a specific group.

### Golden paths for first time users

Understanding that we now actually have **two** journeys for the user to complete, we can keep their goals clearer, rather than sending them on a winding path with dead ends:

1. First, we create a paved road[^aka_golden_path] for **using the service**. Outcome: They want to use the service.
1. Then, we show them the paved road for **accessing the service**. Outcome: they understand the trust and community aspects of the service.

Separating these two experiences can be done in different sessions of a tutorial, and will help the attendees remain focussed on one goal at a time.

## Eliminating Toil

So much for the product development and retention side of things -- we previously mentioned that pre-populating the training users would be a source of toil, even if it made the user experience much better.
What if we could eliminate that toil by encoding the process?

The user pre-registration workflow looks a bit like this:

<div class="mermaid">
---
title: Preparation for User Training
config:
  useMaxWidth: true
  theme: base
  themeVariables:
    primaryColor: "#00ff00"
  flowchart:
    fontSize: 32px
---
sequenceDiagram
  autoNumber
  actor A as IdP Admin
  participant i as IDP

  V->>A: We need users

  A->>i: Create usernames and passwords
  A->>O: We have users
  actor O as Operator
  participant r as RPA
  O->>r: Invoke Process

  activate r
  participant c as Check-In
  r->>c: Sign Up
  c->>c: Create New User
  r->>c: Petition VO
  deactivate r
  actor V as VO Manager
  c->>V: Notify Request
  V->>c: Approve Request

  c->>c: Add to VO

  V->>U: Here are your credentials for the training
  U->>s: Request service
  s->>c: Authentication
  c->>i: Authenticate\nto IdP
  i->>c: Return\nAttributes
  c->>s: Authorize
  s->>U: Grant\nAccess

  actor U as User
  participant s as Service
</div>

Here[^arg_mermaid] we can see that User only really needs to interact with the Service, and they experience the login procedure _as it should be_, thanks to the pre-registration of the identities in Check-In.

We introduce a new actor here called "RPA", which stands for [Robotic Process Automation](https://en.wikipedia.org/wiki/Robotic_process_automation).
This is actually some code which executes the process of first signup on behalf of the users -- this is the "toily" bit which we would like to eliminate from human actors (the Operator in this case).

### Making the signup robot

The process has been implemented using [Robot Framework](https://robotframework.org), by impersonating the user itself, and driving a browser to complete the tasks the user would have done.
This is broken down into tasks shown in the sequence diagram above:

- `Signup`: Sign up a new user to Check-In
- `Join VO`: Petition to join the VO
- `Clean Up`: Collect created EPUIDs and send them to Check-In admin for deletion.

The first two are mentioned in the sequence diagram above, while the last is a process which happens once the training event is completed.
In principle, we could have a multi-actor RPA, which means that we could invoke the roles of VO Manager or IDP Admin as well, but for now, we are only focussing on the part in the middle: registering users and requesting to join the VO.

Let's take a closer look at the code.

{% highlight robot %}
*** Settings ***
Library             Browser
Library             DataDriver    users.csv
Resource            login_resources.robot

Suite Teardown      Close the Browser
Task Setup          Open the Browser
Task Template       Signup

*** Tasks ***
# Sign with user    Default    UserData
Signup    Default    UserData
{% endhighlight %}

Here we can see the single task referred to above.
It is parametrised to use an input file of users, containing their usernames and passwords.

We had to write the task keyword `Signup` ourselves -- it's contained in the `login_resources.robot` file you see declared as a `Resource` in the settings part:

{% highlight robot %}
*** Keywords ***
Signup
    [Arguments]    ${username}    ${password}
    # Click on EGI SSO where the users have been created
    Click    selector=${EGI_SSO_SELECTOR}

    Wait For Navigation    url=https://sso.egi.eu/egissoidp/profile/SAML2/Redirect/SSO?execution=e1s2
    # fill in the login form
    Provide Credentials    username=${username}    password=${password}

    # Accept info release
    Click    css=.grid-item > button:nth-child(1)
    Wait For Load State    domcontentloaded    timeout=30s

    # Follow the signup flow and accept info release and Terms and Conditions
    Click    css=div.grid-item:nth-child(1) > button:nth-child(1)
    Wait For Navigation    url=https://aai.egi.eu/registry/co_petitions/start/coef:2

    Click    selector=a[href='/registry/co_petitions/start/coef:2/done:core']
    Click    css=.checkbutton
    Click    css=div.ui-dialog-buttonpane:nth-child(11) > div:nth-child(1) > button:nth-child(1)
    Check Checkbox    selector=id=CoTermsAndConditions1

    # Submit the request and wait for the server to log us out
    Click    css=div.submit:nth-child(1) > input:nth-child(1)
    Wait For Navigation    https://aai.egi.eu/registry/pages/public/loggedout

Provide Credentials
    [Arguments]    ${username}    ${password}
    Fill Text    id=username    txt=${username}
    Fill Text    id=password    txt=${password}
{% endhighlight %}

The various keywords you see there such as `Click`, `Wait For Navigation` are all in the [Robot Framwork Browser library](https://marketsquare.github.io/robotframework-browser/) which is a python wrapper around [Playwright](https://playwright.dev/).
It drives an actual browser to complete the workflow.

Once the user has created a unique ID in Check-In, we can complete the registration by petitioning to join a VO.

{% highlight robot %}
*** Tasks ***
Join VO    Default    UserData

*** Keywords ***
Join VO
    [Arguments]    ${username}    ${password}
    Click    selector=${EGI_SSO_SELECTOR}
    Fill Text    id=username    txt=${USERNAME}
    Fill Text    id=password    txt=${PASSWORD}
    # Click Login button
    Click    css=.grid-item > button:nth-child(1)
    Click    css=div.grid-item:nth-child(1) > button:nth-child(1)
    # This does not redirect me to the VO enrollment page
    Go To    ${VO_SIGNUP_URL}
    # We still have the cookie, so select the favourite
    Click    css=#favouritesubmit
    # Review AUP
    Click    selector=.checkbutton
    Click    css=div.ui-dialog-buttonpane:nth-child(11) > div:nth-child(1) > button:nth-child(1)
    Check Checkbox    css=#CoTermsAndConditions114
    Click    css=div.submit:nth-child(1) > input:nth-child(1)

    Wait For Navigation    https://aai.egi.eu/registry/
    # Pending acknowledgement notification
{% endhighlight %}

The cleanup task is similar, but logs into Check-In and populates a file with EPUIDs to be sent to the Check-In admin for deletion.

## Discussion: have we really eliminated toil

We now have two simple tasks that have been codified: a repeatable procedure which can be reliably performed by a computer!
**We have removed the work from the Operator**, eliminating part of the toil in this procedure.

## Footnotes and References
[^eliminate_toil]: See [The Google SRE book](https://sre.google/sre-book/eliminating-toil/) for more.
[^User_Research]: I pulled these emotions graded from 0-10 straight out of the air, I have no data to back this up. It would be really interesting to measure user happiness along their journey as part of the typical training activity.
[^aka_golden_path]: The "Paved Road" or "Golden Path" are used interchangeably here, referring to an easy, intuitive, straightforward path from start to goal. Read about some of the differences [on the Octopus blog](https://octopus.com/blog/paved-versus-golden-paths-platform-engineering)
[^arg_mermaid]: I keep getting kicked in the teeth for my optimistic view of mermaidjs. Man, creating this diagram was a pain, and the box and destroy features just straight up didn't work. Oh well. Next time.
