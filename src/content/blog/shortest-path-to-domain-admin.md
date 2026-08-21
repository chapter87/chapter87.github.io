---
title: 'The shortest path to domain admin ran through the account nobody watches'
description: 'I ran attack-path analysis against my own Active Directory domain. The fastest route to owning everything did not go through an admin account — it went through the cloud-sync service account, which every membership audit calls harmless. Here is the finding, why group-based audits miss it entirely, and what actually fixes it.'
pubDate: 'Aug 24 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

I have a working Active Directory domain in my home lab — a domain controller, a member
server, a handful of accounts, hybrid-joined to the cloud the way a real small company runs
it. I pointed BloodHound-style attack-path analysis at it, read-only, expecting to confirm
the obvious: that the road to domain admin runs through the domain admin.

It doesn't. The shortest path runs through an account that every audit I know of would wave
through as harmless.

## The question audits actually ask

When someone reviews Active Directory for "who has the keys," they almost always ask one
question: **who is a member of the privileged groups?** Domain Admins, Enterprise Admins,
Administrators. It's the natural question, it's what the tooling makes easy, and it feels
complete.

In my domain, the answer to that question is short and reassuring. One account —
`Administrator` — sits in the tier-0 groups. Everything else looks ordinary. If group
membership were the whole story, this domain would be clean.

Group membership is not the whole story.

## What I actually found

I asked a different question: **who can replicate the directory?** Specifically, who holds
the right called *Replicating Directory Changes (All)* — the permission that lets a principal
ask a domain controller to hand over secrets, including every account's password hash. This is
the right that makes DCSync possible. Owning it is, functionally, owning the domain.

The list came back with the expected entries — the domain controllers themselves, and the
built-in administrators. And then two entries that stopped me:

- The **cloud directory-sync account** (the one hybrid identity creates during setup).
- Its associated **managed service account**.

Both of these are non-human identities. Both show `adminCount = 0`. Both belong to **zero
privileged groups**. Ask "who is a domain admin?" and neither ever appears. But each one holds
the DCSync right through a **direct access-control entry** on the domain object — a permission
granted straight to the account, invisible to any review that only looks at group membership.

So the real path to total compromise looks like this:

1. Compromise the one server where the sync account lives — a single ordinary host, not a
   domain controller, often patched and watched less carefully than one.
2. Recover the sync account's password. It is stored there, recoverably, by design — the sync
   service needs it to run unattended.
3. Use it to **DCSync** the domain: pull every hash, including `krbtgt`.
4. Forge a **Golden Ticket** and become a domain admin the directory has no record of creating.

Not one step of that touches an account anyone was watching.

## This is not a misconfiguration

The tempting reaction is "someone set this up wrong." They didn't. Hybrid identity sync
*requires* replication rights — that is how it reads password hashes to synchronise them to the
cloud. The setup wizard grants exactly this permission on purpose. It is working as designed.

That is what makes it interesting, not boring. **The most dangerous privilege in the domain is
held by an unwatched machine identity, granted automatically at install time, and never reviewed
again** — because the review everyone runs looks at groups, and this privilege isn't in a group.
The design is correct and the exposure is real at the same time.

It is the same theme I keep landing on from different directions: the identities that get you are
increasingly the ones that aren't people. A service account with a forgettable name and a
crown-jewel permission is worth more to an attacker than most of your staff's logins, and it has
a fraction of the scrutiny.

## Why the tooling matters

This is the entire reason attack-path analysis exists. A membership audit produces a *list* —
who is in what group. Attack-path analysis produces a *graph* — who can reach what, through any
edge, including permissions granted directly rather than through membership. The difference is
not cosmetic. The list said this domain was clean. The graph found a four-step path from one
compromised server to complete ownership, running entirely through accounts the list called safe.

If you only ever do one of the two, do the graph.

## What actually fixes it

None of the fixes are exotic. They are just aimed at the right target for once.

- **Treat the sync account as tier-0.** Same protection class as a domain admin, because that is
  effectively what it is. Not a service account you forget about — a crown jewel you guard.
- **Harden the sync server like a domain controller.** Its compromise *is* domain compromise.
  It should be one of the hardest hosts you run, not one of the softest.
- **Alert on replication from anything that isn't a domain controller.** Event 4662 with the
  replication right, from a principal that is not a DC, is DCSync nearly every time. That single
  detection catches the whole attack the first time it fires — including from the sync account.
- **Audit for permissions, not just membership.** Enumerate who holds replication rights
  *directly*. That is where the privilege you're missing actually lives.
- **Rotate the sync credential on a schedule.** It is long-lived by default, which is to say
  forever, which is to say one leak away from permanent.

## The takeaway

The uncomfortable part of this finding is not that my lab had a weakness. It is that the weakness
was sitting in plain sight, created by a trusted wizard, and made invisible by the exact review
process most people would call diligence.

Ask better questions of your directory. "Who is an admin" is the easy one. "Who can *become* one,
through any path, including the permissions nobody granted through a group" is the one that finds
the account nobody watches.
