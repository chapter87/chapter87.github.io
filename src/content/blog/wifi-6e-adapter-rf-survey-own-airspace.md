---
title: 'What a three-day RF survey of my own airspace turned up'
description: 'A Wi-Fi 6E adapter that can listen and inject, pointed at my own network for a proper audit. It found an open access point broadcasting inside my house for 54 hours, an IoT gadget on the wrong network, and a silent WPA3-to-WPA2 downgrade — all things I would never have seen without looking.'
pubDate: 'Aug 04 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

You can't secure a wireless network you've never actually looked at. So I set up a
proper Wi-Fi audit rig — a capable USB adapter on a Raspberry Pi — and pointed it at
my own airspace for three days. What came back was a genuinely humbling list of
things wrong with a network I thought I'd locked down, plus a clear-eyed picture of
what these tools can and can't do. Everything here was against my own equipment;
the neighbours' networks I only ever *listened* to, passively, and never touched.

## The rig, and its honest limits

The adapter is a Wi-Fi 6E, 2×2 MIMO device — it can put its radio into monitor mode
to capture raw frames, and it can inject frames for testing. The single most useful
thing I learned about it wasn't a feature, it was a *constraint*: on one radio, you
cannot transmit while you're associated to a network. You can listen while connected,
but the moment you need to actively test, you have to give up the connection and
dedicate the radio to monitoring. Injection on a dedicated monitor interface works
perfectly — thousands of frames, no drops — but trying to do it while also staying
online silently fails. Knowing that operational rule up front saves hours of chasing
a "broken" adapter that's actually just doing what the hardware allows.

I also learned to respect how easily this class of adapter can wedge or brown out a
Pi if you reset the driver carelessly — enough that I added a hardware watchdog so a
lockup self-recovers instead of needing me to physically pull the power. Small
detail, real lesson: give anything that can hang your box a way to reboot itself.

## What the survey found on my own network

I left a capture running for three days and then, crucially, actually *read* it —
which is the step most people skip. Three findings stood out, all on my own kit:

- **An open access point broadcasting inside my house for 54 hours.** One of my own
  IoT gadgets, unable to join its assigned network, had quietly fallen back to hosting
  its own *open, passwordless* setup portal — and left it running for over two days.
  Anyone in range could have joined it and repointed the device wherever they liked.
  It only closed when the gadget finally rebooted onto a network it could reach. That's
  the kind of thing that never shows up in a router's dashboard and only appears when
  you're actually watching the air.
- **An IoT device sitting on my trusted network instead of the quarantined one.** By
  correlating its Wi-Fi and Bluetooth signatures — which the adapter can watch at the
  same time — I identified a cheap smart device with an open local-control port sitting
  on the network it absolutely should not have been on. It belonged in IoT jail.
- **A silent security downgrade.** My main network advertises the modern WPA3 standard,
  but it runs in a "transition mode" that still accepts older WPA2 clients for
  compatibility. My adapter proved this by connecting as a plain WPA2 client even
  though WPA3 was on offer. That backwards-compatibility is convenient and it's also a
  weakness — it keeps the door open to the older attacks WPA3 was designed to close.

## The lesson that reframed a decision

There was a nice twist. I'd previously turned on Protected Management Frames set to
*required* on my IoT network, because PMF is the correct defence against the classic
deauthentication attack. But the survey showed the trade-off: my cheap IoT devices
don't support PMF at all, so "required" was locking the very gadgets that network
exists to contain *out* of it — which is exactly how that open portal ended up
broadcasting. Security controls have costs, and the only way to see them is to
measure the real behaviour of your real devices, not the theory.

## Attacking — only my own, only to learn the defence

I did prove the offensive side end to end, entirely against my own access point and
my own devices: a targeted deauthentication genuinely knocks a client off, and the
re-join that follows is the moment a handshake crosses the air. Seeing it work taught
me *why* it works — and, more importantly, why the defences matter. On the network
segment with PMF properly in place, the same attack simply did nothing. That's the
whole point of doing this: you learn to shut a door by opening it, on a lock you own.

The takeaway I keep coming back to: your network is doing things you don't know about
until you listen. An afternoon of reading your own airspace is one of the most
honest security exercises there is.

*Tech: Wi-Fi 6E USB adapter (monitor + injection), Raspberry Pi, Kismet, airodump-ng, 802.11w (PMF), WPA3 transition mode.*
