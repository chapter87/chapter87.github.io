---
title: 'A handheld RF research tool, and the day my Faraday box failed the test'
description: 'I built up an open-source ESP32 multi-radio device for learning about wireless security — legally, in a shielded setup. The most valuable result was a negative one: my shielded enclosure gave zero attenuation, and rigorous testing is the only reason I know that.'
pubDate: 'Aug 04 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

Wireless security is one of those subjects you can't really learn from a book —
you have to watch radios behave. So I put together an open-source ESP32-based
multi-radio device: one small handheld with Wi-Fi, Bluetooth, several 2.4 GHz
radios, and a sub-GHz radio, meant for hands-on RF research.

Before anything else, the legal part, because it's the whole frame: in the UK,
*receiving* and scanning is fine, but actively transmitting on many of these
features is restricted, and RF jamming specifically is illegal to operate at all —
there's no "it's my own device" exemption. So every transmit-side test in this
post happened, or was supposed to happen, inside a shielded enclosure with the
device on battery, isolated from the airwaves. If you take up this hobby, that
constraint isn't optional and it isn't a formality.

What I got out of it wasn't a cool demo. It was a lesson in measurement honesty.

## Trust nothing you haven't measured, part one: the radio that lied

Early on I hit a puzzle: the device claimed it was transmitting Wi-Fi frames —
the firmware reported success, zero errors — but I couldn't confirm anything was
actually reaching the air. So I set up an independent witness: a known-good
Wi-Fi adapter sitting inches away, listening.

The result was clear and useful. **Bluetooth transmission worked** — the witness
caught it, and so did a separate monitoring device across the house. But
**Wi-Fi transmission was completely dead**: five different transmit tests, the
witness six inches away, and not a single frame ever arrived, while that same
witness happily heard neighbours' networks *through walls*. The firmware was
reporting success into the void.

That's a genuinely important habit: a device telling you it did something is not
evidence it did. Only an independent measurement is. Without the reference adapter
I'd have believed the firmware and been completely wrong.

## Trust nothing you haven't measured, part two: the box that wasn't a box

Here's the one that actually mattered for safety. My whole plan for doing
transmit-side tests legally rested on a shielded enclosure containing the signal.
So before running anything at power, I measured the enclosure itself: how much
2.4 GHz energy from the outside world was getting *in*?

The answer was: **all of it.** The number of occupied channels inside the closed
box was essentially the same as out in the open room. External Wi-Fi passed
straight through. My "Faraday box" provided no measurable shielding whatsoever.

Which means any transmit test I'd have trusted it for would have been radiating
into the open air — exactly the thing the box was supposed to prevent. I stopped
and shelved the powered tests until the enclosure can be fixed and *verified* with
a simple check (a phone that can't receive a call inside it, and a baseline that
reads near-zero). The assumption I was about to rely on for both legality and
safety was flatly false, and the only reason I know is that I tested it instead of
trusting it.

## The takeaway

The flashy version of this hobby is "look what this gadget can do." The version
worth putting your name to is the discipline underneath it: know the law before
you transmit, verify your containment before you trust it, and never accept a
device's own report as proof it did anything. My best result from this build was
learning that two things I was about to rely on — a transmit path and a shielded
box — didn't work. Negative results, rigorously established, are still results.

*Tech: ESP32-S3, nRF24L01, CC1101 (sub-GHz), 802.11 monitor mode, RF measurement, and a healthy respect for the Wireless Telegraphy Act.*
