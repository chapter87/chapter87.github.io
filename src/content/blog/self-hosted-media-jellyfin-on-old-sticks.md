---
title: 'Running my own media server on hardware nobody wanted'
description: 'How I built a self-hosted Jellyfin media library on a small NAS and got it onto cheap, old, half-locked living-room streaming boxes.'
pubDate: 'Aug 06 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

I wanted one thing: my own films and music, on my own hardware, playing on the TVs I already own, without paying a monthly fee to stream files I already have. No cloud, no subscription, no "this title is no longer available in your region." A month later I have exactly that, and the interesting part was never the media itself. It was the plumbing.

## The server

The core is Jellyfin, a free and open-source media server, running as a container on my home NAS. The NAS is built around a low-power Intel chip with Quick Sync, which matters more than it sounds: it means the server can transcode video in hardware. If a client is too weak to play a file directly, the server re-encodes it on the fly at almost no CPU cost, instead of choking a general-purpose processor. The library lives on a dedicated storage pool, and Jellyfin reads it as read-only paths mounted into the container.

That container detail bit me almost immediately, and it turned into the most useful lesson of the whole project.

## The port that lied

Jellyfin listens on a well-known port *inside* its container. Every guide, every forum post, every default config references that number. So when my home network couldn't reach the server, I did the obvious thing and opened a firewall rule for that port. Nothing. Total silence, hours of it.

The mistake was trusting the documented port. My NAS platform doesn't publish containers on their internal ports at all. It maps each one to a different, much higher port on the host, and *that* is the only number the outside world ever sees. My firewall rule was allowing traffic to a port nothing was listening on. The fix was one line: allow the real published host port, not the app's advertised one. Traffic flowed instantly.

The lesson is general and it's the kind of thing that separates people who *use* containers from people who can *debug* them: the port an app documents is almost never the port it's actually reachable on once something orchestrates it. Always confirm what the host is really publishing before you blame the network.

## Getting it onto old, cheap boxes

Here's where it got genuinely fun. My living rooms run on two very different streaming sticks. One is a recent 4K model. The other is an ancient little box driving a small TV, old enough that its official app store gave up on it years ago.

The recent stick was easy: the app is right there in the store. The old box was the puzzle. Its app store won't offer Jellyfin, and even when I fed it the raw app package by hand, half my attempts failed outright with an "older SDK" error. The box runs an operating system so old that most modern apps refuse to install. Jellyfin's TV app, thankfully, still ships a build that targets it, so I sideloaded that specific version over a debugging connection and it ran.

The newer stick had its own trap. It looks like a powerful 4K device, but underneath it runs a 32-bit userspace. Hand it a 64-bit app package and the install silently fails. You have to check the processor architecture and pick the matching 32-bit build every single time. It's a small check that saves an hour of confusion.

## The network quirks nobody warns you about

Two more gremlins were pure home-network trivia that cost real time. First, one stick had quietly joined the wrong wireless network entirely, one that sat outside my managed setup, so it was double-routed and firewalled off from everything. Rejoining the right network fixed a problem that looked, for an hour, like a server fault. Second, modern browsers now force every address to its secure version by default. My NAS admin page and my password manager share a machine, so typing the NAS address kept silently redirecting me to the password manager instead. Maddening until you spot it; trivial once you do.

## The payoff

I looked at reaching all this remotely over a mesh VPN, and I built that path. But I measured it, and the relay was too slow for higher-bitrate films, so I fixed the local network route instead and got a sixfold speed jump. Measure, don't assume.

None of this is exotic. It's containers, firewalls, DNS, device architectures, and the patience to check each layer instead of guessing. That patience is the actual skill.
