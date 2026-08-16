---
title: 'I built the identity infrastructure a real company runs — in my homelab'
description: 'Standing up Active Directory, Keycloak, Authentik and OpenLDAP on my own hypervisor, then automating the joiner/mover/leaver lifecycle the way an IAM team actually does it.'
pubDate: 'Jun 22 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

Every IAM job spec asks for the same things: directory services, single sign-on, least privilege, and lifecycle management. Most people learn those words from a slide deck. I decided to build the real thing at home instead — the same identity plumbing a mid-sized company quietly depends on — and run it until it broke, then fix it.

Here is what I stood up on my own two-node hypervisor cluster.

## The directory is the source of truth

The centre of any corporate identity story is Active Directory, so I built a fresh domain controller from scratch. Not clicking through a wizard once and walking away — I promoted a clean Windows Server into a new forest, set the domain functional level, confirmed replication and DNS health with `dcdiag`, and verified SYSVOL and NETLOGON were shared and healthy. A domain controller that "looks up" and one that actually passes its own diagnostics are two different things, and knowing the difference is half the job.

Around that I ran three self-hosted identity providers in containers, each one restarting automatically on boot so the lab survives a host reboot without me:

- **Keycloak** as the primary SSO broker, speaking both OIDC and SAML.
- **Authentik** as a second modern IdP to compare federation models.
- **OpenLDAP**, with a web admin front-end, as a standalone directory to practise the raw LDAP layer that sits underneath everything else.

To prove the SSO layer wasn't just running but actually working, I federated a real application to Keycloak over both OIDC and SAML. That meant creating a realm, registering a confidential client, wiring up the discovery endpoint, and mapping user attributes like username and email into the assertion. When the app's login page finally rendered a "Sign in with Keycloak" button and the redirect carried a signed token back, that was the moment the whole thing became real single sign-on rather than three services sharing a network.

## Joiner, Mover, Leaver — the part that actually matters

SSO gets the attention, but the lifecycle is where identity teams live. So I built the joiner/mover/leaver process end to end.

First I gave the directory a shape a real org would recognise: a top-level organisational unit for the company, sub-OUs for active users, disabled accounts and service accounts, and department security groups for IT, Finance, HR, Sales and Engineering. Then I wrote three PowerShell scripts to drive the lifecycle:

- **Joiner** creates the account, generates a unique logon name, drops the user into the right OU, and grants the department group that gives them least-privilege access on day one.
- **Mover** handles the internal transfer — swapping group membership so that changing teams also changes access, instead of the classic "permissions pile up forever" problem.
- **Leaver** disables the account, strips every group, moves it to the disabled OU, and scrambles the password so the identity is dead but auditable.

Every script has parameter validation, a `-WhatIf` dry run, and logging, because an IAM script that silently does the wrong thing to a production directory is worse than no script at all. I ran the full lifecycle against test users — a joiner, a team move, and a full offboard — and watched each stage land correctly in the directory.

## The one problem that taught me the most

The leaver script failed the first time, and the error was a good one. My original version tried to hide the departing user from the address list using an Exchange attribute. On a real corporate DC that attribute exists because Exchange extends the AD schema — but my domain was vanilla, so the attribute simply wasn't there, and the script threw.

The lesson stuck with me more than any successful run: **automation is only as portable as its assumptions.** Code that works against one directory can quietly depend on a schema extension you didn't know was there. I stripped the Exchange-specific attribute, and the offboarding became something that works on any Active Directory, not just a lucky one. That is exactly the kind of thing you want to learn in a lab and never in production.

## Why this is on my CV

I can now talk about directory services, SSO federation over OIDC and SAML, least-privilege group design, and a working JML lifecycle — not as vocabulary, but as things I have built, broken, and fixed with my own hands. That is the difference I was after.
