---
title: 'I ran a security audit on my own home network. It was humbling.'
description: 'I do security as a discipline, so I assumed my own home network was in decent shape. Then I actually scanned it like an attacker would. A smart display was handing out root shells to anyone on the Wi-Fi, and my "isolated" IoT devices were not isolated at all.'
pubDate: 'Aug 16 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

It's easy to assume your own house is secure, especially when security is the
thing you study. So I decided to stop assuming and treat my home network the way
an attacker would: enumerate everything on it, and see what a foothold on the
Wi-Fi could actually reach. The results were a useful dose of humility, and a good
reminder that everyone's home network deserves this once.

*(General findings and lessons here — I've kept specifics out on purpose, and
everything called out has been or is being fixed.)*

## Finding 1: a smart gadget was a root shell for anyone on the Wi-Fi

The one that made me sit up: a cheap smart display was listening on a debugging
port that, by design, drops a connection straight into a **root shell with no
authentication whatsoever**. Not a password prompt. Not a confirmation. Any device
on the network — a guest's phone, a compromised laptop, a dodgy app — could
connect and own that gadget instantly, and from there pivot deeper.

This is the quiet reality of cheap IoT: the device works, the app is pretty, and
under the hood it left a wide-open developer door that nobody closed. **Assume
every cheap smart device on your network is hostile until proven otherwise.**

## Finding 2: my "isolated" IoT was not isolated

I'd told myself the IoT gadgets were on their own segment, walled off from my
laptop, my server, and my NAS. When I actually tested it, my laptop could talk
directly to those IoT devices with nothing in the way. It was all one big flat
network — a single space where any compromised device can reach every other one.

Segmentation you *believe in* but haven't *tested* is not segmentation. The whole
point of putting untrusted devices on their own VLAN is that a break-in on the
smart bulb can't become a break-in on the file server — and that only holds if the
firewall between them is real and verified from the untrusted side.

## Finding 3: management interfaces reachable from everywhere

The admin interface of a core piece of infrastructure was reachable from that same
flat network — including from the very IoT devices that turned out to be
trivially compromisable. That's the dangerous shape of a real breach chain: weak
device → flat network → management plane. Each link is minor on its own; together
they're a path from a smart clock to the keys of the kingdom.

## What I fixed, and what I'd tell anyone

- **Turn off debug interfaces on IoT devices**, or if you can't, quarantine those
  devices somewhere they can reach the internet and *nothing else on your LAN*.
- **Actually segment, then test from the untrusted side.** Put IoT, guests, and
  work on separate VLANs, default-deny between them, and verify the walls hold by
  probing *from* the untrusted segment — not from your trusted laptop, which will
  happily tell you everything is fine.
- **Never expose a management interface to your whole network.** Lock admin planes
  to a specific trusted host or a VPN, not "anything on the LAN."
- **Do this to your own house.** An hour of scanning your own network the way an
  attacker would is one of the highest-value security exercises there is, and it
  costs nothing but honesty about what you find.

The uncomfortable takeaway: the person who studies this for a living still had a
root-shell-on-request device sitting on a flat network. Defaults are not safe,
convenience quietly erodes isolation, and the only way to know is to look.

*Tech: nmap, VLAN segmentation, host firewalls, network enumeration methodology.*
