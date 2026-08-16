---
title: 'Building Hermes: a self-hosted AI agent that runs my digital life'
description: 'A personal AI agent living on my own VPS, reachable from my phone, that reads the live web, remembers context, survives provider outages, checks on my home network, and even edits its own code — built with security as the first constraint, not an afterthought.'
pubDate: 'Aug 16 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

I use AI assistants all day, and I kept wanting one that was *mine* — running on
hardware I control, reachable from my pocket, able to actually do things rather than
just chat, and built so that its considerable reach can't be turned against me. So I
built one. I call it Hermes. What started as a chat bot has grown into a genuine
personal operations agent, and the interesting story is as much about the security
architecture as the features.

## The shape of it

Hermes is a Python service on a small VPS, and I talk to it through a messaging app
from anywhere. Underneath that simple front door sits a real agent: it can call
tools, hold a conversation with memory, reach out to the live internet, and take
actions on my own infrastructure. None of it is exposed to the public internet —
which, given what it can do, is the whole point.

## Making it reliable: provider failover

A single AI provider is a single point of failure. Rate-limit it or have it go down
and the assistant is dead. So Hermes routes each request down a chain of models and
providers, falling back automatically when one stalls or refuses, ending at a model
that will always answer. I put an AI-gateway service in front of the whole thing to
manage that routing and failover cleanly. The result is an assistant that stays up
and useful even when a provider has a bad day. Cold starts on the fallback used to
make it feel broken; a small scheduled warm-ping keeps it awake and ready.

## Giving it memory and the live web

Two upgrades turned it from a toy into something I rely on. First, **memory** — a
local memory engine so it remembers earlier context in a conversation and can recall
facts across sessions, all stored on my own box rather than a cloud. Second, **eyes
on the live internet** — a set of tools that let it fetch and read web pages, pull
articles and video transcripts, and read public feeds, so its answers aren't frozen
at some training cutoff. An agent that can't read today's web is guessing; one that
can is genuinely useful.

## Reaching my own world — carefully

This is the capability I'm proudest of and most careful about. Hermes can reach into
my own infrastructure: check the health and status of my home network, see how my
own devices are doing, and act on machines I own — all from a message sent while I'm
miles away. That's powerful, and power pointed at your own home is exactly the thing
you have to secure properly or not build at all.

So the reach is wrapped in defense in depth, and this is the part worth copying:

- **Nothing is exposed to the internet.** The agent's services sit behind a
  default-deny firewall. The only way in is the messaging front door, and the only
  way it reaches home is over an authenticated, encrypted private mesh — never an
  open port, never the public network.
- **It only obeys me.** The bot answers a single authorised identity. A stranger who
  finds it gets nothing.
- **Least privilege, everywhere.** It can do the specific things I've granted and
  nothing broader, and the sensitive paths run through a hardened jump point rather
  than exposing the network directly.

The lesson that shaped all of it: **the more capable you make an agent, the more its
security is the actual project.** The features are the easy half. Making sure a thing
that can act on your home network can't be hijacked into acting *against* it is the
half that takes the real engineering.

## Letting it fix itself — behind a safety gate

Once it was stable I gave Hermes the ability to edit its own code, compile-check the
change, and restart itself cleanly. That sounds reckless, and it would be without the
compile gate — that check is the whole safety net. If the change doesn't build, it
doesn't ship, and the running version stays up.

There was a hard lesson buried here too. The agent can also update itself from
upstream, and early on those updates silently overwrote my local customisations — I'd
fix something, and days later an update would quietly revert it. The fix was to stop
treating my changes as edits to upstream files and layer them so an update can't
clobber them. Anything that can rewrite your system needs to know what it's not
allowed to touch.

## Watching the watcher

The last piece: Hermes can't be trusted to report its own death. So a second,
completely separate little machine at home watches it from the outside and messages
me if it goes quiet, plus a daily heartbeat so I know the monitoring itself is alive.
Out-of-band monitoring is a lesson you learn once, usually the hard way.

## What it taught me

The chat loop was a weekend. Everything that makes Hermes genuinely useful *and*
safe — the failover, the memory, the careful reach into my own network, the
self-editing behind a gate, the external watchdog — is where the real work lives. And
the throughline is the one I most want to carry into security work: build the
capability, then spend most of your time making sure it can only ever be used by the
person it's meant for.

*Tech: Python, a messaging bot front end, an AI-gateway for provider failover, a local memory engine, web-reading tools, a private mesh VPN, systemd, a Raspberry Pi for out-of-band monitoring.*
