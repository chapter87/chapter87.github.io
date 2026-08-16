---
title: 'I host my own password manager now'
description: 'Every password I own lived in someone else’s cloud. I moved them to a Bitwarden-compatible server running on my own NAS, reachable only over my private mesh — never exposed to the internet — and kept the polished apps I already liked.'
pubDate: 'Jul 21 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

A password manager is the most security-critical app you run. It holds the keys to
everything else. So there's an obvious tension in trusting all of it to a company's
cloud: one breach, one policy change, one account lockout, and your entire digital
life is someone else's problem. I decided to keep the excellent Bitwarden apps and
move the *server* under my own roof.

## The setup

I run **Vaultwarden** — a lightweight, open-source server that speaks the same
protocol as Bitwarden — on my home NAS. The clever part is that I didn't have to
give up anything: the official Bitwarden apps on my phone and browser talk to *my*
server instead of the company's, just by pointing them at a custom server URL.
Same polished autofill, same interface, but the vault itself lives on hardware I
control, holding a few hundred entries that are now mine in every sense.

## The security decision that makes or breaks it

Self-hosting the thing that holds all your passwords is only an *upgrade* if you
don't then expose it to the internet. A password server on a public IP is a giant
target with a bullseye on it. So mine is **never reachable from the public
internet at all** — the only way to reach it is over my private mesh VPN.

That gives me the best of both: my phone syncs the vault whether I'm at home or out
on mobile data, because the mesh works everywhere — but to anyone scanning the
internet, the server simply doesn't exist. Reachability without exposure. (Getting
my phone onto the mesh was, amusingly, the only fiddly step — it had to be on the
right account before it could see the server. Once it was, the vault connected
first try.)

## Why do this

It's not that the big password managers are bad — they're generally excellent. It's
that for the single most sensitive service I run, I wanted the trust boundary to
end at my own front door. If it's going to be catastrophic when it's breached, I'd
rather the attack surface be a box on my shelf that isn't even visible to the
internet, than a household-name cloud service that thousands of attackers probe
every day.

It's also just a genuinely good exercise. Standing this up teaches you how a
password manager actually works, why the vault is encrypted client-side before it
ever reaches the server, and how to think about exposure versus reachability — which
is the same lesson underneath half the security decisions I make.

*Tech: Vaultwarden, the Bitwarden apps, a NAS, a mesh VPN, client-side encryption.*
