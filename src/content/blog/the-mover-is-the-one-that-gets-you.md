---
title: 'Nobody deprovisions a promotion'
description: 'Joiners get provisioned and leavers get deprovisioned, but the mover accumulates access from every role they''ve held, because the HR transfer event almost never triggers a revoke.'
pubDate: 'Aug 19 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

Pull the group memberships on anyone who has been at the company eight years and changed teams a few times. I did this last quarter, during an access review. One account, a manager in finance, sat in forty-one security groups. She could tell me what six of them were for. The rest was sediment: a distribution list from a support-desk role in 2019, write access to a shared drive from a stint in marketing, an approver seat in a procurement app she touched for one project and never opened again. Nobody had removed any of it, because nobody ever offboarded her. She kept getting promoted.

That account is the most common audit finding I see, and it sits in the middle of a model everyone can recite: Joiner, Mover, Leaver. The joiner and the leaver get budget. The mover gets a slide and no owner, and the mover is what quietly wrecks least privilege.

## A joiner is an event. A mover is a diff.

A hire fires provisioning. A termination fires deprovisioning. Both are discrete, both have an owner. HR flips a status and downstream systems react, and nobody argues about whether a termination should remove access.

A move is not an event in that sense. It is a change to attributes on an identity that already exists. Department goes from A to B. Adding B's access is easy, and everyone wants it fast, because the person starts the new job Monday. Removing A's access is the other half, and it has no owner and no deadline. So it doesn't happen.

## Why the transfer never reaches the revoke

Workday raises a Change Job or Transfer business process. Your provisioning layer receives a SCIM PATCH: `op: replace` on department, title, manager. That is all SCIM (RFC 7644) is built to carry. It updates attributes. It has no concept of "and revoke whatever the old attributes entitled."

For the revoke to fire, your IGA tool needs a lifecycle event you configured on purpose. In SailPoint that's a lifecycle event watching the department attribute; when the value changes, it recomputes entitlements and deprovisions the delta. Out of the box that rule does not exist. Most integrations wire hire and terminate and stop there, because those were the two events the project was scoped around.

## Birthright recomputes. The one-off never does.

Even with the trigger configured, it can only remove what it can attribute to the old role. That splits your access into two piles, and they behave nothing alike.

Birthright access, the stuff granted by rule, cleans up if and only if it was written as membership rather than a one-time assignment. An Entra dynamic group with `user.department -eq "Finance"` drops the mover the moment the attribute flips. A static group, or an on-prem AD group, does not: `memberOf` has no expiry, so it only ever grows.

Then there is requested access. The one-off approvals. The "can you add me to this share" over Slack that someone actioned in thirty seconds. None of it is tagged to a role, so nothing in the system knows it went stale the day she moved. This is the honest limit of the remove step: it assumes you know what to remove, and for requested access you structurally don't. You cannot revoke a mapping you never recorded.

## The control that's supposed to catch this doesn't

The standard answer is periodic access reviews. Recertification. Send each manager a list, ask them to confirm their reports still need what they hold.

In practice they hit approve-all. The incentive is lopsided: revoke something that turns out to be needed and you've created a ticket, blocked an employee, and left a trail back to you. Approve something unneeded and nothing visible happens. So reviewers certify the sediment quarter after quarter, and the review becomes the paper trail that proves the over-privilege was signed off. That's the part most write-ups leave out.

Accumulation isn't just clutter. Raise a purchase order in procurement, move to finance where you approve payment, and now one person does both. That's a segregation-of-duties violation, the classic SOX finding, and it shows up at movers. A joiner gets one role. A mover collects conflicting ones.

## What actually fixes it, and what it costs

Three things, each with a real price.

Treat the transfer as a first-class trigger, equal to hire and terminate. Configure the lifecycle event; lean on dynamic membership so birthright self-corrects. The cost is that someone has to own the role-to-entitlement mapping and keep it current as the org changes. That's a standing job, not a one-off project, and it's the one nobody wants to fund.

Grant-then-remove, with a hard deadline on the remove. You'll grant the new access first, for continuity, which opens a window where the person holds both roles' rights. Fine, but the removal has to be scheduled and enforced, say fourteen days, then revoked automatically with a re-request path if it was wrong. You can't have zero downtime and zero combined-access at once. The window is the cost; the deadline is what stops it becoming permanent.

Time-box anything you can't map. Access with an expiry beats access someone has to remember to pull. Entra PIM does this for privileged roles: eligible instead of active, activate with a TTL. The catch is that PIM is aimed at admin roles, and creep actually lives in the boring app entitlements nobody thinks to put behind it. Expiry means re-request friction, and users resent it. That friction is the price of not finding someone in forty-one groups.

The account with access from three roles ago isn't a mistake anyone made. It's the absence of a step nobody was assigned. Give the transfer an owner and a deadline, or keep meeting her in every audit.
