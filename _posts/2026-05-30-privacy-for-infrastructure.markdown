---
layout: post
title:  "Privacy for Infrastructure: Addressing Problems at the Root"
date:   2026-05-30 12:00:00 -0800
categories: jekyll update
---

(This post was adapted from a [talk](https://fpf.org/wp-content/uploads/2021/06/Session-1-1-Privacy-for-Infostruction-by-OMadadhain-Young.pdf) that I [presented](https://youtu.be/lenVTvDMWxM) at [PEPR 2021](https://fpf.org/fpf-event/pepr-2021-conference-on-privacy-engineering-practice-and-respect-2/) with [Gary Young](https://www.linkedin.com/in/gary-young-bb9b28/).)

One should generally solve problems as far down in the stack as possible; privacy is no exception.  This post explores what shifting left looks like in privacy, why it’s a good idea, how to do it, and where some of the pain points may be.

## Terminology

* **system**: application, product, or service
* **infrastructure**: a system that provides other systems with capabilities.  ([“Infrastructure: If Anything Exciting Happens, We’ve Done It Wrong”](https://www.youtube.com/watch?v=Wpzvaqypav8&t=1247s).)  
    * examples: storage, networking, data processing systems, server frameworks, libraries, system integrations
* **product**: a user-facing system (e.g. application)

Infrastructure has **clients** (the systems that use it); products have **users** (the people that use it).

**Data-agnostic**[^1] infrastructure is not aware of the kinds of data it handles, that is, it doesn’t handle privacy-relevant data differently from other data.)  The data-agnostic approach tends to be simpler and more general; however, sometimes infrastructure is made data-agnostic to avoid responsibility for solving privacy problems (“we just handle data, it’s the client’s job to do it right”).[^2]

## Why should privacy be infrastructure’s problem?

First, because the infrastructure can be part of the problem independent of whatever its clients may be using it for.  Leaving infrastructure out of the privacy design of a system makes about as much sense as leaving it out of the security design.  (And while security provides essential tools for solving privacy problems, not all privacy problems are security problems.)

Second, because **solving problems in infrastructure is how you solve them at scale**; this is just as true for privacy problems as for any other sort.

For this reason, data-agnostic infrastructure should receive extra scrutiny to ensure that it’s not missing opportunities to fix problems once rather than requiring each of their clients to do so.

Finally, because infrastructure that solves engineering problems at scale **may create privacy problems at scale** if it doesn’t make a point of solving them.  Privacy problems that aren’t handled systematically across an organization will at best be re-solved repeatedly, inconsistently, and expensively.  Infrastructure and automation are a much better way to solve those problems than spreadsheets and a lot of human attention.

**Exception**: if only one client has a particular problem, that may indicate that the client should be solving that problem.

## Infrastructure privacy design review

Privacy design reviews for both infrastructure and products each need to consider the following factors:

* what privacy-relevant data are handled?
    * infrastructure often generates its own logs, error messages, etc. based on clients’ activity; these may therefore include privacy-relevant data
* what are the relevant purposes for handling this data?
* data minimization: did you need to collect that?  do you need to keep it?
* access control: who has access, and how is that controlled?
* retention: are data being properly removed by request and on schedule?
* dependencies: what does this system depend on?

However, infrastructure review needs to be at a **different level of abstraction** than product review: it needs to answer the question _”how does the infrastructure help its clients to meet **their** data handling needs?”_

As a result, infrastructure review may need to approach some topics differently.  Not all of these topics apply to all kinds of infrastructure; for instance, infrastructure that doesn’t store anything won’t have retention risks.

### use cases/purpose limitations

Infrastructure’s purpose for handling privacy-relevant data is usually only to provide the service.  However, if it has other purposes (such as building predictive models that incorporate data from multiple clients) those must be compatible with declared purpose limitations, and documented, just as they are for products.

Infrastructure may also perform data governance functions on behalf of its clients, including:

* annotation management: parsing, generation/inference, propagation
* governance: purpose limitations, access control, join restrictions

It’s important for infrastructure to document any data governance roles it may have, so that clients know what they can, and can’t, depend on infrastructure to handle for them.

**what can possibly go wrong?**

* Serving infrastructure builds a model from client data to make serving more efficient; as a side effect, it predicts user behavior for clients that have user data.  The ads team gets access to this model, and starts using it for ads targeting, in contravention of the purpose limitations for this data.  Lawyers ensue.
* Data processing infrastructure does not propagate annotations of data that pass through it from its inputs to its outputs.  This creates a hole in the affected clients’ data governance that they can’t easily patch.


### access control

Infrastructure generally should not allow clients to access each others’ data; setting up per-client access controls should be part of onboarding.

Infrastructure that stores or serves client data may have internal access paths for operational or debugging purposes.  Historically many infrastructure system owners have built their systems so that any member of their team has free access to those internal paths.  Infrastructure review should consider whether:

* limitations on such access (such as multi-party authorization, access logging, structured justification, or redaction by default) are appropriate
* client team members should be given access to those internal paths

Infrastructure that’s responsible for the fundamental building blocks of access control (e.g. cryptography, authentication/authorization checks, identity and access management) should be among the most carefully scrutinized.  Security review should catch most problems in this area, but their evaluation of risks may be different than privacy’s, and any integrations with data governance are likely to be more in privacy’s wheelhouse.

**what can possibly go wrong?**

* Infrastructure team leaves access to their internal logs open to all of their clients. One client snoops on other clients’ logs, uses this for business advantage, and then a whistleblower in that client’s organization tells the media what happened and where the data came from.  Lawyers ensue.

### retention/deletion, data export, data updating

Storage infrastructure that handles privacy-relevant data should either support these functions automatically, or provide client APIs for them.  Otherwise the infrastructure team would need to handle such requests manually.

Since these functions (especially full deletion) can take time, infrastructure should document any propagation delays that their system may introduce.

All of this applies to primary data stores, caches, and backups.

**what can possibly go wrong?**

* Infrastructure neither handles data export for its clients, nor empowers its clients to handle it.  All export requests (whether from users or in response to legal requests) have to be handled manually by the infrastructure team.
* Infrastructure adds a new backup system that covers all client data, and keeps backups around indefinitely because storage is cheap and they want to help their clients recover their data.  This places them out of compliance with retention deadlines for clients that handle privacy-relevant data.  Lawyers ensue.

### notice, consent, control

These don’t generally apply directly to infrastructure directly, but the infrastructure may be able to empower its clients to set, or inspect, the status of:

* notices: has a specified user seen the most recent version of this notice?
* consent moments: what has the user consented to?
* other controls, such as user-to-user sharing or blocking actions

**what can possibly go wrong?**

* Infrastructure does not support reading and writing consent state; each client implements its own consent storage and retrieval.
    * An organization has multiple products that each allow customers to opt **out** of using their data for advertising and marketing, and records their decision.  Since each product has its own records, customers wishing to opt out of all such uses must opt out for each product.  This causes customer confusion and frustration.
        * In response, the organization requires each product to respect "opt out" requests collected by other products (but does not mandate common storage); this increases maintenance costs for each product.
    * Later, the organization allows customers to opt **in** to using their data to build AI models, adding further complexity to the consent state.  Some products misinterpret others' consent records, and build AI models using data from customers that have not opted in.  Lawyers ensue.

### aggregating and obscuring data

This includes anonymization, pseudonymization, redaction, deidentification, and generating summary statistics.  As with access control mechanisms, these are both important to do consistently across your organization, and easy to do in a way that introduces subtle vulnerabilities, so ideally infrastructure will provide common solutions.

**what can possibly go wrong?**

* Infrastructure does not provide an anonymization solution; clients use a mixture of strategies (mostly redaction) in an attempt to obscure the identities of their users.
    * This makes data governance difficult and inconsistent across the organization.
    * A data scientist pulls together insufficiently anonymized data from different clients and uses it to reidentify users.

## Configuration

The amount and complexity of per-client configuration needed has direct operational effects on clients, and indirect effects on user protections.  

Where configuration is possible, or necessary, there are two critical elements:
* responsibility: who does the configuration, infrastructure team or client team?
* complexity: how is the configuration performed?
    * simple UI
    * creating/editing configuration files
    * writing code 

In descending order of preference, this is what infrastructure per-client configuration should look like:

1. Zero configuration
1. Good stance by default
1. Good stance requires per-client configuration
1. Good stance requires client-side implementation
1. Good stance not possible

We use a storage system as a motivating example in the sections below.

### zero configuration

Infrastructure automatically performs all relevant privacy functions for all clients; no configuration is possible.

* all data are encrypted with client-specific keys that are generated at onboarding and managed via IAM integrations
* retention timelines are enforced for all data
* all aspects of data export are managed by infrastructure

### good stance by default

Infrastructure does something conservatively appropriate for all clients, but provides configuration options.

* data are encrypted by default; different encryption options available; client may choose to manage their own keys
* clients can override default retention timelines
* clients can customize data export formats

### good stance requires per-client configuration

Infrastructure can be configured to satisfy client needs.  

* clients must specify that they want their data encrypted, and manage their own keys
* clients must opt into specify a deletion timeline, and describe the data to be deleted
* clients must configure data export mechanisms and protocols

### good stance requires client-side implementation

* clients must encrypt their own data
* clients are responsible for keeping track of their data’s age and deleting it to meet retention requirements
* clients must implement data export themselves

### good stance not possible

The infrastructure’s design does not support the client’s privacy goals.  This is not a value judgement, but it is critical that infrastructure documents the privacy goals it does not, or cannot, support so that prospective clients can make informed choices.

* infrastructure does not encrypt data, and required data format does not support client-side encryption
* infrastructure does not provide client APIs for data deletion or export

## Change management

Infrastructure systems sometimes need to make client-visible changes.  This can place a large burden on clients, especially if the changes affect per-client configuration or otherwise impact privacy guarantees in ways that clients have to compensate for.

Infrastructure with open/public APIs may not know who all of its clients are; this can limit its ability to understand and address its clients’ needs, and make change management very difficult.

Examples of changes that may impact privacy guarantees:

* changing retention implementation timelines or policies
* starting, or stopping, logging of access requests, or changing what information is in them
* creating new dependencies that have data governance implications

Options for infrastructure for helping clients navigate change include:
* provide stability: allow clients to stay with a given version of infrastructure as long as possible
* own client configurations: allow infrastructure owners to edit/maintain client configurations so that they can make some changes without clients’ intervention
    * this may create other risks
* make changes with plenty of notice, documentation, and support
    * this requires reasonably robust communication channels with clients

## Conclusion

Infrastructure teams generally want to help solve their clients’ problems.  Privacy can and should work directly with these teams to help them to understand where there are opportunities for their infrastructure to solve privacy problems--and thus create value--for all of their clients.

-----------

[^1]: This is related to, but not identical with, the GDPR concept of a [data processor (as opposed to a data controller)](https://commission.europa.eu/law/law-topic/data-protection/rules-business-and-organisations/obligations/controllerprocessor/what-data-controller-or-data-processor_en).

[^2]: A more subtle version of the “it’s the clients’ responsibility” problem arises in the very common situation in which there are multiple layers of infrastructure and no clear division of responsibility.
