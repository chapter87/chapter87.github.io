---
title: 'Hybrid identity in a homelab: syncing on-prem AD to Okta and Entra'
description: 'IAM job specs ask for hybrid identity and directory sync. Instead of just reading about it, I built the whole thing in a homelab — AD on-prem, synced up to both Entra ID and Okta, managed as code.'
pubDate: 'Aug 15 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

Almost every Identity & Access Management job ad asks for the same things:
directory synchronisation, single sign-on, joiner/mover/leaver lifecycle,
least privilege. You can read about all of it. But reading about directory sync
and having actually watched a password hash flow from on-prem Active Directory up
into a cloud tenant are two very different lines on a CV. So I built the real thing.

## The setup

- **On-prem Active Directory** running on a Proxmox host — the authoritative
  source of truth for identities, the way it works in most real organisations.
- **Microsoft Entra ID** connected via directory sync, with Password Hash Sync,
  so my on-prem users can authenticate against the cloud tenant.
- **Okta**, importing and reconciling the same users from AD.
- A **joiner / mover / leaver** flow across all of it, because provisioning an
  account is the easy half — the hard, and more important, half is making sure
  access *leaves* when a person does.

## A design decision I had to make: who owns what?

The moment you have the same users represented in Terraform, in AD, and in two
identity providers, you have to answer one question or you'll be fighting drift
forever: **for any given thing, which system is the source of truth?**

The rule I settled on: **Terraform owns the group *definitions*, and Active
Directory owns the *membership*.** Terraform declares that a group exists and
what it's for; AD decides who's in it. That split matters because if both systems
think they own membership, every sync becomes a tug-of-war — Terraform reverts a
change an admin made in AD, someone re-does it, the next `apply` reverts it again.
Drawing that boundary once, clearly, is what makes the whole thing stable. It's
also a genuinely useful thing to be able to talk through in an interview.

The Terraform side runs through CI, so a change to the identity config is a pull
request that gets checked before it lands — the same discipline you'd want
managing identity for a real org, not click-ops in a portal.

## The bit that kept biting me

Cloud security defaults. Every so often the tenant's security defaults would
quietly switch back on and demand interactive multi-factor auth on an account I
was using for *automated* sync — which promptly broke the headless flow with an
error that, on the surface, looked like a credential or permissions problem. I
lost real time chasing it as an auth bug before realising the platform had simply
re-asserted a policy I'd relaxed.

The lesson: when an automated identity flow suddenly starts failing auth, check
what the *platform* changed before you assume you broke something. Security
defaults protecting you from yourself is a feature — but it'll happily break your
service account at 11pm if you're not expecting it.

## Why this was worth doing

This project maps, almost line for line, onto the language in IAM job specs:
directory synchronisation, SSO, identity lifecycle, infrastructure as code,
least privilege. But more than that, building it taught me the questions that
only show up when the systems are actually running against each other — source
of truth, drift, and what happens when a platform quietly overrules you.

*Tech: Active Directory, Microsoft Entra ID, Okta, Terraform, Proxmox, CI.*
