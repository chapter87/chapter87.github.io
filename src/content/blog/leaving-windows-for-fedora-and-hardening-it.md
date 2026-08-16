---
title: 'Leaving Windows for Fedora, and hardening it'
description: 'Why I wiped Windows for Fedora as my daily driver, and the security hardening I did afterward — the wins, and one lesson that stung.'
pubDate: 'Jul 30 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

I finally did the thing I had been circling for months. I wiped Windows off my main PC and installed Fedora Workstation as my only desktop operating system. No dual-boot, no safety net. If I needed Windows for the occasional Office document, it would live inside a virtual machine, not underneath everything I do.

## Why I left

The honest trigger was telemetry. Windows keeps a device identifier tied to your hardware, and I watched it quietly re-establish itself even after I tried to strip it out. That was the moment it clicked: I was fighting the operating system to make it respect a boundary it was designed to ignore. On a machine I actually own, that is the wrong relationship.

There was also a career reason. I am aiming at identity and access management work, and that world is heavily Red Hat native. SELinux, Podman, Keycloak, FreeIPA — these are the tools I want fluency in, and Fedora puts me a few commands away from all of them instead of behind a translation layer. Choosing Fedora was equal parts privacy decision and study decision.

Before I touched anything, I did the boring, important part: I backed up the roughly 28 kilobytes that were genuinely irreplaceable. Not a full disk clone — my SSH keys, GPG material, and cloud credentials, packed into a single archive, encrypted with a passphrase stored in my password manager, and pushed to a private repository. Everything else was either already in the cloud or re-installable. That discipline mattered, because a wipe is only terrifying when you are unsure what you are about to lose.

## Hardening the new box

A fresh Fedora install is already reasonable out of the box — SELinux runs in enforcing mode, the firewall is on, and the SSH server is not even listening. But "reasonable default" is not "hardened," so I wrote a single script that applies my whole baseline and can be re-run any time.

The changes that gave the most value for the least fragility were the ones at the network edge. I set the firewall's default zone to drop, which means unsolicited traffic gets silently discarded rather than politely refused. That one setting quietly contains badly-behaved applications. A music player on my machine, for example, insisted on binding to every network interface to advertise itself. With the drop zone in place, that port is simply invisible from anywhere else on the network. The app still works; the exposure does not exist.

On top of that I layered the usual suspects: automatic security updates applied daily, kernel-level tightening through sysctl, obscure networking modules blacklisted so they cannot be loaded, the root account locked so all administration goes through a normal user, and audit rules watching the files that matter — password and shadow files, the sudo configuration, scheduled tasks. I also added file-integrity checking so that if something on disk changes unexpectedly, I find out on a schedule instead of by accident.

To check my own work, I ran scans back against the machine from another host on the network and from a server out on the internet. From both vantage points the box looked effectively dead — no open ports, no useful fingerprint. That is exactly what I wanted to see.

## The lesson that stung

Here is the honest part. The most useful things you can turn on are also the ones most eager to lock you out.

Key-only SSH is the correct way to allow remote access. But if you enable it before you have actually placed a client's public key on the machine, you have just built a door that opens with a key that does not exist yet. I caught this while planning, not after, but the shape of the mistake is classic: hardening and self-lockout are the same action seen from two sides. USB device control has the same trap — flip it on before whitelisting your own keyboard and you can find yourself unable to type.

The takeaway I carry from this is simple. Every hardening step needs a rehearsed way back in before you commit to it. Write the rollback command down first. Test remote access from a second device while you still have a session open. Treat "how do I undo this" as part of the change, not an afterthought.

Fedora is now the machine I trust, precisely because I know what it does and does not do. That is the whole point of leaving.
