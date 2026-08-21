---
title: 'Detecting DCSync means not alerting on the account that does it every two minutes'
description: 'The hard part of catching a DCSync attack is not spotting the replication request — it is that your cloud-sync account makes the exact same request, legitimately, all day long. Here is how I built a detection that tells the two apart, tested against real traffic.'
pubDate: 'Aug 25 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

DCSync is the attack where someone with the right permissions asks a domain controller to
replicate its secrets to them — every account's password hash, including the `krbtgt` key
that signs every Kerberos ticket in the domain. It is a devastating move, and it has a famous,
almost unmistakable fingerprint: Windows event 4662, exercising the *Replicating Directory
Changes All* right, identified by a specific GUID.

So detection looks trivial. Alert on 4662 with that GUID. Done.

I built exactly that rule, pointed it at my own lab domain, and watched it for ten minutes
before firing a single attack. In those ten minutes it matched the replication GUID eight
times. None of them was an attack.

## The account that DCSyncs all day, on purpose

If you run hybrid identity, your directory contains a sync account — the one that keeps your
on-premises accounts mirrored into the cloud. To synchronise password hashes, that account
needs to *read* password hashes, which it does by replicating the directory. In other words:
your cloud-sync account performs a DCSync, legitimately, every couple of minutes, forever.

Every one of those eight matches in my log was that account doing its job. The replication GUID
was present. The event was real. And it was completely benign.

This is the trap in "just alert on the fingerprint." The naive rule doesn't fire once when an
attacker strikes. It fires every two minutes, all day, on normal operation. Within a week
someone mutes it. The month after that, the real DCSync arrives, lands in a muted rule, and
nobody sees a thing. A detection that cries wolf is worse than no detection, because it
manufactures the false confidence of a green dashboard.

## Alert on the anomaly, not the action

The fix is to stop asking "did a DCSync happen?" and start asking "did a DCSync happen *from
something that has no business doing one?*"

The legitimate principals are a short, knowable list: the domain controllers themselves (they
replicate to each other constantly), and the identity-sync accounts. Everything on that list is
expected to hold replication rights and use them. Nothing else is.

So I split the detection in two. The first rule matches the replication GUID and fires high
severity. The second rule sits behind it and *downgrades* the alert to an informational
baseline whenever the requesting account is one of the known-good principals — the DC machine
accounts, the sync account, the managed service account behind it. The high-severity alert
survives only when the account doing the replicating is none of those.

```
rule A (level 12): 4662 + Replicating-Directory-Changes GUID  ->  "DCSync by <account>"
rule B (level 3):  rule A  AND  account is a DC$ / sync identity  ->  "expected, baseline"
```

The list of exclusions is not a convenience. It *is* the detection. The entire security value
sits in knowing precisely which non-human identities are supposed to hold this power — which,
if you have read anything else on this blog, is a question I think most organisations cannot
answer. You cannot write rule B until you have actually enumerated who can replicate your
directory. Most teams discover the sync account has this right for the first time while writing
this exact rule.

## What the ten minutes proved

Watching real traffic before running any attack turned out to be the most useful test I could
have run. The sync account fired the baseline rule eight times, exactly as designed — matched,
recognised, downgraded, silent. The high-severity rule stayed at zero. That is the hard
90 percent working: the detection is quiet during normal operation, so when it does speak, it
means something.

The rule is now armed to fire high severity the moment a principal outside that known-good list
exercises replication rights — a compromised admin, a freshly-granted attacker account, anything
that is not the handful of identities that legitimately sync. That is the behaviour by design,
and the tuning that makes it trustworthy is the part I could verify against live traffic.

## What this costs, and where it can still fail

The honest limitations, because a detection you trust blindly is its own risk:

The exclusion list is a maintenance burden with teeth. Add a second sync tool, a new
replication partner, a DR domain controller, and you must add it to rule B — or you get a
false positive on the next legitimate change. Forget to *remove* one when you retire it, and
you have handed an attacker a name to impersonate: compromise the account you excluded, and your
own tuning waves the attack straight through. The exclusion that suppresses noise is also an
allowlist an attacker would love to land on. It has to be reviewed like the privileged list it
actually is.

And this is one detection for one technique. It catches the replication request. It does nothing
about the attacker recovering the sync account's password off the sync server in the first place,
or forging a Golden Ticket after the hashes are out. Detection engineering is not a wall; it is a
series of trip-wires, and this is one wire across one path.

But it is the right wire, in the right place, tuned so it will still be switched on when it
matters. The measure of a good detection is not that it fires. It is that it stays silent until
it shouldn't — and that you tested the silence.
