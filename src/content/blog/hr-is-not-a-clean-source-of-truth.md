---
title: 'HR is primary, never sole'
description: 'Making HR your only source of identity truth leaves contractors and off-book accounts alive because no HR event will ever remove them, while the obvious fix deletes the accounts you can least afford to lose; here is what reconciliation actually has to catch.'
pubDate: 'Aug 15 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

The account had read/write access to the finance app and it belonged to no one. The reconciliation job flagged it: active in the directory, entitlements attached, no matching worker in the HR feed. Not a service account. A person's name, a person's mailbox, last login three days earlier. Someone was using it, and HR had never heard of them.

This is what HR-as-source-of-truth looks like the day it breaks, and it breaks constantly. The design is sound and I still recommend it. Everyone just stops reading after the design.

## The theory, which is correct

HR is the authoritative source for who works here. Workday, SuccessFactors, BambooHR, pick one. A worker record is created, and that event (the joiner) triggers provisioning: an account, a mailbox, birthright groups derived from department and job code. A change to the record (the mover: new manager, new cost center) re-evaluates access. A termination (the leaver) disables the account. Joiner, mover, leaver. The HRIS holds the truth; IAM subscribes to it, usually through an IGA platform pulling a nightly worker report or catching events over an API, and pushes downstream over SCIM with `active: false` as the off switch.

I've built exactly this. It works for the population it was designed for: full-time employees hired, managed, and terminated through HR. In most orgs that's a minority of the accounts you're on the hook for.

## Where the drift comes from

Contractors are the first hole. The agency developer who's been shipping code for eight months was never a worker record. Someone created the account by hand, or filed a ticket, and no HR event will ever fire to remove it. When the contract ends, nothing happens. The account just stays.

Termination lag is the second, and it's worse because it's invisible. HR enters a leaver on the 15th, effective-dated to the 1st. Your sync only saw it on the 15th. For those two weeks the account was live, and if you reconcile against the effective date the report looks clean: it shows them terminated on the 1st. It lies about the fourteen days in between. In Workday, the effective date and the date the change was actually entered are different values, and if you key off the wrong one you will never see the gap.

Then the long tail. Dual employment: one human, two positions, two worker records, two identities that both believe they're authoritative. Name changes: someone gets married, the surname changes, and if your correlation key was ever built on name or email, the link snaps and you've manufactured an orphan out of a perfectly good employee. Re-hires: the boomerang returns, HR reuses the old employee ID, and last tenure's entitlements resurrect because a stale account correlated straight back. And the exec assistant who provisions for the C-suite off-book, because waiting for the JML process is not a thing that happens to executives.

Every one of these produces the same artifact: an account whose lifecycle HR does not control. The orphan I opened with was a contractor whose sponsoring manager had themselves left.

## Why "make HR the only source" doesn't work

The tempting fix is to declare HR the sole source and deprovision anything without a worker record. Do that and you delete the contractor mid-sprint, kill the shared service account that legitimately has no human behind it, and disable a break-glass account the night you need it. Sole-source is a deletion policy pointed at your own operations.

Real-time provisioning doesn't save you either. If the bottleneck is a person in HR entering the termination four days late, streaming events instead of a nightly batch just propagates the lateness faster. The pipe was never the problem. The data entry was.

## What actually reconciles

HR is primary, not only. You define authoritative sources per population: HR for employees, the vendor management system (or a dedicated contractor process with a hard expiry date) for contingent workers, a CMDB entry with a named human owner for service and machine accounts. Then reconciliation runs as a standing job, not a project. Aggregate every account from every downstream system and correlate it back to a source on a stable key. Not email. Not name. A generated internal identity ID you own and HR can't recycle. It survives a surname change, and it won't hand a re-hire the ghost of their last tenure. Same human, same identity, every time.

Reconciliation produces the one thing that matters: an exception queue. Accounts that correlate to nothing. Accounts whose owner is already terminated. Accounts still live past an expiry date nobody enforced. And you cross-signal for the termination-lag gap using data that doesn't come from HR: badge-system last swipe, VPN last login, manager attestation. The only way to catch a leaver HR hasn't recorded is a source that isn't HR.

## What fixes it, and what it costs

The fix is an authoritative-source matrix, a stable correlation key, and reconciliation feeding an exception queue a named human works every week. None of that is a product you buy and switch on.

The cost is that last clause. Reconciliation with nobody to work the exceptions is a dashboard nobody opens; I've watched an uncorrelated-accounts report sit at four hundred rows for a year. Service accounts need an attestation cadence with a real owner, because there is no leaver event for a token: the owner quits and the token stays. And catching the exec assistant's off-book account means either closing that path, which is political, or accepting you'll find it after the fact, which is detection you have to staff.

HR as the source of truth is the right design. It is not the whole system. The whole system is everything you build to handle the identities HR was never going to tell you about.
