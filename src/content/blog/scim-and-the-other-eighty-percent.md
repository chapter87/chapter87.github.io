---
title: 'The 80% your IdP never touches'
description: 'Your IdP provisions the ~20% of apps that speak SCIM; the offboarding gaps that fail audits live in the 80% it never reaches.'
pubDate: 'Aug 22 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

I ran an offboarding audit last spring against a flat list from HR: everyone who had left in the previous ninety days. The IdP was confident. Every one of them showed as deprovisioned, every lifecycle state green. Then I logged into the finance app directly, the one that approves purchase orders, and four of those people could still authenticate. Nothing was misconfigured. That app doesn't speak SCIM. Its offboarding was a ticket in a queue, and the ticket had been closed months earlier when someone deactivated a similarly named account.

That gap is the whole problem. Not a SCIM bug — the 80% of your estate SCIM was never going to reach.

## What SCIM actually does well

For an app that implements RFC 7644 properly, provisioning is close to hands-off. A joiner fires a POST /Users with the core schema — userName, name, emails, externalId — and the enterprise extension carries manager and department. A leaver is a PATCH: {"op":"replace","path":"active","value":false}. A mover syncs changed attributes. The IdP owns the record, the app follows, and you stop thinking about it.

This is what every provisioning demo shows. Create a user in the directory, watch it appear in a SaaS app thirty seconds later. It works.

## The number nobody puts on a slide

Count your apps. Actually count them: pull the SSO logs, the expense reports, the browser-extension inventory. Most mid-size orgs land in the low hundreds. Now count the ones your IdP provisions over SCIM. Every environment I've measured comes back at roughly a fifth.

Two reasons it stays that low. First, SCIM is usually gated behind an "enterprise" or "identity" pricing tier, the SSO tax with provisioning bolted on top. Second, "supports SCIM" means very little on its own. Plenty of implementations create but never deprovision. Some ignore PATCH. Some have no group support, so every entitlement decision falls back to manual. SCIM-capable and automatable end to end are different sets.

## Where it falls apart

**Custom entitlements.** The core schema defines entitlements and roles attributes, but almost nobody implements them the same way, so in practice you push group membership and let the app map groups to internal roles. That holds until the business needs "AP approver, but only for the EU entity." SCIM has no vocabulary for fine-grained, scoped authorization. It was never meant to.

**The mover.** Provisioning is additive. Adding a group on a role change is a clean PATCH. Removing the access the person no longer needs is the step nobody automates, because it requires knowing the delta between the old role and the new one. So movers accumulate. The clearest audit finding I file, every time, is someone who moved from AP to AR and kept both — a separation-of-duties break created entirely by successful provisioning.

**The deprovision race.** active:false is asynchronous. The push can succeed while the app's live session stays alive and its OAuth refresh token keeps minting new access tokens for another hour. Worse, push failures are often silent: the IdP marks the user deprovisioned because it sent the request, not because the app confirmed the account is dead. That is exactly how my four finance-app users stayed live under a green dashboard.

## The patterns that cover the rest

**Event-driven glue off the IdP.** Okta event hooks, Entra ID lifecycle workflows, any IdP that emits a user.lifecycle.deactivate webhook. Point it at a small function that calls the app's native API. Yes, this is building a connector, but a narrow one you own, for the handful of high-risk non-SCIM apps that justify the effort.

**HR-triggered runbooks for the long tail.** Make the HRIS the source of truth. A termination drives the workflow: SCIM apps deprovision automatically, and every non-SCIM app generates a ticket with a named assignee, an SLA, and a specific account identifier — not "remove John's access," but the exact login to kill.

Then quarterly access reviews are supposed to catch whatever slipped through. Be honest about how those go. Most recertification gets rubber-stamped: managers approve every line without reading it, and the one account that matters hides in a page of green. A review reconciles nothing unless you force the reviewer to justify each keep instead of clicking approve. It's the control I've watched fail most, and I still run it, because even a sloppy review sometimes surfaces the account nobody can explain.

## Why "just build a connector" is a trap

The build is the cheap part, and it's the part everyone budgets for. Each app's API brings its own auth, pagination, rate limits, error semantics, and idempotency rules. Deprovision 500 users in a batch and you'll hit a rate limit you never tested. Then the expensive part arrives: the app ships an API change, your connector fails quietly, and you have a deprovisioning gap you can't see until an audit finds it. Multiply by every integration. You didn't write a script — you're operating a fleet of fragile distributed systems, on call, forever.

## What actually fixes it, and what it costs

Nothing makes the 80% disappear. Accept that and the strategy gets simple. Pick one source of truth and let the HRIS hold it. Spend real automation budget only on the apps that earn it, the ones with the highest headcount and the widest blast radius, with event-driven glue you're prepared to maintain. Put everything else on tracked tickets with owners and SLAs, and backstop the lot with reviews you actually enforce.

The cost isn't a license. It's headcount: an identity engineer who owns the glue and runs the reviews, plus a standing acceptance that offboarding keeps a human in the loop across most of your estate. Anyone selling you fully automatic joiner-mover-leaver hasn't counted their apps. Count yours before you believe them.
