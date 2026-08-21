---
title: 'The identity kill chain: one credential in memory to the whole cloud tenant'
description: 'How an attacker turns a single stolen credential sitting in a machine’s memory into total control of an Active Directory domain — and then the cloud tenant synced behind it. Six moves, each with the exact signal that betrays it and the control that ends it. The cloud pivot at the end is the part most write-ups miss.'
pubDate: 'Aug 23 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

Most identity attacks are not clever. They are a chain of ordinary moves, each one
unremarkable on its own, that add up to somebody owning your entire directory — and, if you
run hybrid identity, the cloud tenant behind it.

I mapped the whole chain against a controlled Active Directory domain in my home lab. What
follows is each link, the real technique behind it, and — because knowing the attack is only
half a defender's job — the exact signal that gives it away and the control that breaks it.

The important thing to hold onto before we start: **every link depends on the one before it.**
A defender does not have to win at every stage. Break any single link and the whole cascade
stops. The cheapest places to break it are near the beginning.

## 1. Foothold

The attacker lands on one domain-joined workstation with local administrator rights. A phished
user, a reused password, an unpatched service — the specific door does not matter. What matters
is that this is the *only* door they have to force from the outside. Everything after this
happens inside your walls.

- **Detect:** endpoint telemetry and Sysmon process-creation events; new-device or
  impossible-travel sign-ins.
- **Defend:** least privilege, application allowlisting, phishing-resistant MFA at the edge.

## 2. Credential dump — Mimikatz

Here is the move everyone has heard of, and the reason it works is a design decision, not a bug.
Windows caches your secrets in the memory of a process called LSASS so that you do not have to
retype your password every time something needs it. Mimikatz reads that memory and lifts
plaintext passwords, **NTLM password hashes**, and Kerberos tickets straight out of it.

No cracking. No brute force. The credentials are just *sitting there* in RAM, and local admin is
enough to read them.

- **Detect:** this is one of the highest-signal events in all of Windows security. Sysmon Event 10
  records a process opening a handle to `lsass.exe`; the specific access rights Mimikatz needs
  show up as a recognisable value. Almost nothing legitimate reads LSASS that way.
- **Defend:** Credential Guard puts those secrets behind virtualisation-based security so the
  memory is unreadable. LSA Protection (`RunAsPPL`) hardens the process. Disable legacy WDigest so
  plaintext is never cached in the first place.

## 3. Lateral movement — pass-the-hash

This is the step that surprises people. The attacker does not need your actual password. **The
hash *is* the credential.** Windows authentication will happily accept the hash directly, so the
attacker moves across the network as that user without ever knowing or cracking the plaintext.

The goal of the movement is specific: find a machine where a **domain administrator** is currently
logged in, land there, and dump memory again — because now they will lift a domain admin's
credential instead of an ordinary user's.

- **Detect:** network logon events (Type 3) using NTLM in host pairs that never normally talk to
  each other; workstation-to-workstation administrative authentication.
- **Defend:** LAPS gives every machine a unique local admin password, so one dumped workstation
  does not unlock the next. Tiered administration means domain admins never log in to
  workstations, so their credentials are never in that memory to steal.

## 4. DCSync

Once the attacker holds a domain admin credential, they do something audacious: they **pretend to
be a domain controller.** Active Directory controllers replicate changes to each other constantly,
over a protocol designed for exactly that. The attacker invokes that protocol and asks a real
controller to replicate the directory's secrets to them — every account's hash, including the one
key that matters most: **krbtgt.**

No malware is dropped on the controller. Its disk is never touched. To the DC, it looks like a
peer asking for a routine sync.

- **Detect:** this one has an almost unambiguous fingerprint. Windows Event 4662 records the
  replication right being exercised — and if the principal exercising it is *not* a domain
  controller, that is DCSync, essentially every time. It is one of the most reliable detections in
  the entire kill chain.
- **Defend:** restrict directory replication rights to actual domain controllers, and alert the
  first time anything else attempts it.

## 5. Golden ticket

The `krbtgt` account's key signs every Kerberos ticket in the domain. It is the master key to how
everyone proves who they are. Once the attacker has it, they stop stealing tickets and start
**forging** them — a ticket for any user, in any group, valid for years.

They are now, in effect, a domain administrator that the directory has no record of ever creating.
This is why DCSync of `krbtgt` is the point of no return: it converts a break-in into permanent,
invisible ownership.

- **Detect:** anomalies in ticket-granting requests — outdated RC4 encryption, lifetimes that
  exceed your domain policy, service tickets that arrive with no preceding authentication request.
- **Defend:** once `krbtgt` has leaked, there is only one real cure: rotate it **twice** (the double
  rotation is required to fully invalidate existing tickets). Everything else is theatre.

## 6. The cloud pivot — the part most write-ups skip

Here is where this stops being a 2015 Active Directory story and becomes a 2026 one.

If you run hybrid identity — and most organisations now do — your directory contains the accounts
that **synchronise it to the cloud.** The Entra Connect sync account (you will recognise it by its
`MSOL_` prefix), an `OktaService` connector account, and their equivalents. These are non-human
identities: highly privileged, created once during setup, and then almost never looked at again.

They are the bridge. Own them, and the on-premises breach you have been reading about becomes an
**Okta or Entra tenant breach.** The attacker crosses from your server room into your cloud, using
a credential nobody was watching, because everyone was watching the humans.

This is the thread that runs through everything I have been working on lately. The dangerous
identities are increasingly the ones that are not people — the service accounts, the sync
connectors, the automation credentials. They have enormous privilege and almost no scrutiny, and
they are exactly where a domain compromise turns into a cloud compromise.

- **Detect:** cloud sign-in logs for the sync principal appearing from new locations; credentials
  being added to service principals; impossible travel on a non-human account.
- **Defend:** treat sync accounts as tier-0 — the same protection class as domain admins. Keep
  cloud administrators separate from on-prem ones. Enforce Conditional Access. Rotate the sync
  credential on a schedule, not never.

## Where the chain breaks

Read back through the defences and a pattern appears: you do not need all of them. You need *one,
early.*

- **Break it at stage 2:** Credential Guard and LSA Protection make LSASS memory unreadable, so
  Mimikatz lifts nothing, and there is no hash to pass in stage 3.
- **Break it at stage 3:** LAPS and tiered admin mean a dumped workstation never surrenders a domain
  admin credential, so the attacker never reaches DCSync.
- **Break it at stage 4:** a single alert on non-DC replication catches DCSync the first time it
  fires — before `krbtgt` ever leaves the building.
- **Break it at stage 6:** a watched, tier-0 sync account keeps an on-premises incident from ever
  becoming a cloud one.

The attacker has to complete every link. You only have to sever one. That asymmetry is the whole
reason defence is possible at all — and it is why mapping the chain end to end, instead of chasing
each technique in isolation, is what tells you where to spend your effort.

*I built an interactive, ATT&CK-mapped version of this chain as a visual — each stage as a
holographic node with its technique, detection, and defence. This written version is the reference;
the diagram is the map.*
