---
title: 'Do Wi-Fi deauth attacks still work? I tested it on my own network'
description: 'Everyone demos the deauthentication attack on YouTube. I wanted to know whether it still works against a modern, correctly configured access point. So I set up a closed lab with my own kit and found out.'
pubDate: 'Aug 14 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

The Wi-Fi deauthentication attack is the one everybody demonstrates: knock a
device off its access point, force it to reconnect, capture the handshake. It's a
staple of every wireless security tutorial. But most of those tutorials are years
old, and Wi-Fi has moved on. I wanted a straight answer to a simple question:
**does this still work against a modern access point that's set up correctly?**

So I built a closed lab — my own access point, my own client devices, my own
adapter — and tested it properly. Short version: mostly no. And *why* it fails is
the genuinely interesting part.

## The kit

I used an Alfa AWUS036AXML, a Wi-Fi 6E adapter on the `mt7921u` driver. Before
testing anything, I confirmed it could actually inject: pushing frames at my own
access point, I logged thousands of injected frames landing successfully across
both bands. So when an attack failed later, I knew it was the *target's* defences,
not a broken adapter — which is exactly the kind of thing you have to rule out
before you trust a negative result.

## The finding: Protected Management Frames change everything

The classic deauth attack works by spoofing a management frame — an unprotected,
unauthenticated "you're disconnected now" message that the client obediently
believes. That's the flaw the whole attack rests on.

Modern APs support **Protected Management Frames (802.11w, "PMF")**, and when
it's set to *required*, those management frames are authenticated. My spoofed
deauth frames simply got ignored. The client stayed connected. The attack that
every tutorial shows as a sure thing just... did nothing.

To still capture a handshake in the lab, I had to fall back to waiting for a
*legitimate* reconnect — toggling the client's Wi-Fi and catching the handshake
as it naturally re-joined. Which rather makes the point: with PMF required, the
easy remote attack is gone and you're reduced to needing physical or social
access to the client.

## The practical takeaway

This is a rare case where the defensive advice is genuinely cheap and genuinely
effective: **turn PMF on, set it to required.** It closes the single most
demonstrated Wi-Fi attack in existence, and most modern APs support it. If you
run a network and it isn't on, that's a five-minute win.

## One for the notes: the driver wedges

A practical gotcha for anyone doing this: after repeatedly switching the adapter
between modes, the `mt7921u` driver would lock up and stop cooperating. The fix
was to fully reload the kernel module — unload and reload it — to get a clean
state. Small thing, but it wasted enough of my evening that it's worth writing
down so it doesn't waste yours.

## The line I won't cross

Everything here happened against my own equipment, on my own network, in a closed
lab. That's not a disclaimer bolted on the end — it's the whole point. The value
of security work is being able to put your name on it, and you can only do that
when the target is yours. Testing the defences on your own network to understand
how they hold up is exactly the kind of hands-on learning that makes the theory stick.

*Tech: Linux, aircrack-ng / hcxdumptool, 802.11w (PMF), Alfa AWUS036AXML, mt7921u.*
