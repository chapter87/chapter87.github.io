---
title: 'Testing Wi-Fi frame injection in my own lab (and why PMF ruined my day)'
description: 'Do the deauth attacks everyone demos on YouTube still work against a modern, correctly configured access point? I tested it — on my own network.'
pubDate: 'Aug 14 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

> Draft — hands-on wireless security, done ethically on my own kit.

I wanted to know whether the deauthentication attacks everyone demos on YouTube
still work against a modern, correctly configured access point. I tested it
against **my own** AP. Short answer: mostly no — and that's the interesting part.

## What I actually did

- Used an Alfa AWUS036AXML (Wi-Fi 6E) and verified frame injection on both bands
- Ran a controlled deauth test against my own access point
- Found that Protected Management Frames (**PMF = required**) blocks the classic attack

## The finding worth writing up

- Deauth fails against PMF-protected clients; a handshake capture needed a manual client reconnect instead
- What this means for real-world defence: **turn PMF on** — it's a cheap, effective control

## The ethics line

Everything here is my own equipment, on my own network, in a closed lab. That
distinction is the whole point of doing security work you can put your name on.

*Tech: Linux, aircrack-ng / hcxdumptool, 802.11w (PMF), mt7921u.*
