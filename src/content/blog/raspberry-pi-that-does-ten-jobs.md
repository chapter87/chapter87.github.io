---
title: 'The Raspberry Pi doing ten jobs on my network'
description: 'One small, cheap, low-power computer runs my DNS, blocks ads for the whole house, acts as a hardened jump host, bridges my network segments, and doubles as a wireless-audit node. Here is how I set it up and locked it down.'
pubDate: 'Aug 10 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

The most useful machine on my network cost about sixty pounds and sips a couple of
watts. It's a Raspberry Pi 4, and over time it has quietly become the Swiss army
knife of my home lab — running a handful of genuinely important jobs at once. This is
what it does and, just as importantly, how I keep something that central actually
secure.

## Job one: DNS and ad-blocking for the whole house

The Pi runs an ad-blocking DNS resolver that every device on the network points at.
That means trackers and ad domains get blocked at the network level, before an app
or a smart TV ever loads them — no per-device software required. Behind that sits a
recursive resolver that does the actual lookups itself, so my browsing history isn't
handed to a third-party DNS provider. When my internet provider turned out to be
[quietly intercepting DNS](/blog/isp-was-hijacking-my-dns/), this same Pi was where I
fixed it, by routing lookups out over an encrypted path they couldn't touch.

## Job two: a hardened jump host

I never expose SSH to the open internet. Instead the Pi is reachable over a private
mesh VPN, and it acts as a jump host to reach other things on the network when I'm
away. Because it's a doorway, I hardened it hard: SSH is key-only with passwords
completely disabled, and an intrusion-prevention service jails any address that
fails to log in. That combination is worth internalising — key-only auth means there's
no password to guess, and the jail means an attacker doesn't even get to keep trying.
(I learned the jail works a little *too* well the day I fat-fingered my own username
and locked myself out for a while. A good problem to have.)

## Job three: bridging my network segments

Here's a subtle, powerful role. The Pi has both a wired and a wireless connection,
which means it can reach parts of my segmented network that other devices can't. That
makes it the safe, controlled bridge for specific tasks that need to cross a
boundary — a deliberate, monitored crossing point rather than a hole in the wall.

## Job four: a wireless-audit node

With a capable USB Wi-Fi adapter attached, the Pi becomes a permanent
[wireless monitoring station](/blog/wifi-6e-adapter-rf-survey-own-airspace/),
capturing and analysing what's happening in my own airspace over days at a time. A
tiny always-on box is exactly the right host for that kind of long-running,
low-intensity job — it just sits there and watches.

## Keeping something this central alive

When one small machine does this much, its reliability matters. Two lessons stood out.

First, **give it a way to save itself.** Low-level work with the USB adapter could
occasionally hang the whole Pi, and because it isn't powered over Ethernet there was
no remote way to reboot it — I'd have to physically pull the plug. The fix was to
enable the hardware watchdog, so a lockup now self-recovers in seconds instead of
waiting for me to come home. Anything unattended and important needs a dead-man's
switch like that.

Second, **keep it patched, carefully.** I keep it current with regular updates, but
I've learned to watch them land rather than assume — an update once left a package
half-configured because of a driver quirk, and a service I depended on didn't come
back until I looked. On an always-on box you trust with real jobs, "did the update
actually finish cleanly?" is a question worth asking every time.

## Why I love this thing

A Raspberry Pi is the perfect teacher precisely because it's cheap and constrained.
Every one of these roles — DNS, hardened remote access, network bridging, wireless
monitoring — is a real production concept in miniature, running on hardware I can hold
in my hand and rebuild from scratch in twenty minutes if I break it. For the price of
a takeaway, it's the most educational machine I own.

*Tech: Raspberry Pi 4, AdGuard Home + a recursive resolver, key-only SSH + Fail2ban, a mesh VPN, a USB Wi-Fi adapter, hardware watchdog.*
