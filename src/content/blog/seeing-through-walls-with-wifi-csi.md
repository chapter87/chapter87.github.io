---
title: 'Seeing movement through walls with a $10 Wi-Fi chip'
description: 'Wi-Fi signals bounce off and pass through everything in a room, including people. With an ESP32 and some open-source software, you can read those distortions and sense human presence and pose — no camera involved. I built a working node and dashboard.'
pubDate: 'May 12 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

Here's a fact that sounds like science fiction but is just physics: the Wi-Fi
signal filling your home is constantly being distorted by everything it passes
through and bounces off — walls, furniture, and people. If you can read those
distortions precisely enough, you can detect where a person is, and even roughly
what pose they're in, **without a camera**. It's called Wi-Fi sensing, and the
raw material is something called Channel State Information (CSI).

I wanted to see it work with my own hands, on cheap hardware, so I built a sensing
node.

## The build

The sensor is an **ESP32-S3** — a microcontroller that costs about the price of a
coffee. Flashed with open-source CSI firmware, it does one job very well: for
every Wi-Fi packet it hears, it captures the fine-grained channel measurements and
streams them out over the network to a machine that does the heavy analysis. On
the receiving side runs an open-source Wi-Fi "DensePose" pipeline in a container,
turning that stream of raw radio measurements into a live view of what the signal
is telling us about the space.

End to end, it works: the node pushes CSI frames at around 46 per second, the
dashboard picks them up, and it flips from its fallback simulation mode to
**"real hardware connected."** Watching a graph respond to someone moving in a
room it can't see is genuinely uncanny.

## The bug that taught me to read the fallback logic

The first time I wired it up, the dashboard insisted it was in *simulation* mode
even though my hardware was clearly streaming real data. I could see the packets
leaving the ESP32.

The cause was a subtle one worth remembering. The software had an "auto-detect"
mode that waited a short window to decide whether real hardware was present — and
my ESP32's first packet arrived *just after* that window closed. So it gave up,
declared "no hardware," and fell back to simulated data, ignoring the real stream
that showed up a moment later. Forcing it to expect a hardware source instead of
auto-detecting fixed it instantly.

The lesson: when a system has an automatic "is the real thing there?" check and it
guesses wrong, don't fight the symptom — find the timing assumption behind the
guess. Auto-detection is convenience that quietly makes decisions for you, and
when it's wrong it's wrong silently.

## Why this matters

Wi-Fi sensing is a real and slightly unsettling capability: presence and motion
detection that needs no camera, works in the dark, and sees through walls, built
from hardware anyone can buy. For a security-minded person that cuts both ways —
it's a privacy consideration worth understanding, and it's a fascinating,
low-cost way to get hands-on with the physical layer of the networks we otherwise
treat as invisible plumbing.

*Tech: ESP32-S3, Wi-Fi CSI, ESP-IDF, Docker, an open-source Wi-Fi DensePose pipeline.*
