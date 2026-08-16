---
title: 'Building Hermes: a self-hosted AI agent that never leaves me stuck'
description: 'I wanted an AI assistant I fully control — reachable from my phone, running on my own box, that answers security questions instead of refusing them. Here is what it took, and what broke along the way.'
pubDate: 'Aug 16 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

I use AI assistants all day for security work, and I kept hitting the same wall:
the moment a question got genuinely technical, the answer turned into a refusal.
Fair enough for a public service — but I wanted something I own, that runs on my
own hardware, and that treats me like an adult. So I built one. I called it Hermes.

## What it actually is

Hermes is a small Python service on a cheap VPS, wired to a chat bot I message
from my phone. I can ask it anything, hand it a task, and get a real answer back
in a few seconds. Nothing about my usage leaves infrastructure I control.

The interesting engineering isn't the chat loop — that part is easy. It's
everything I had to add to make it *reliable* and *useful* when I'm not watching it.

## Reliability: a model fallback chain

A single model provider is a single point of failure. Rate-limit it, or have it
refuse, and the whole assistant is dead. So Hermes routes each request down a
chain: a fast hosted model first, and if that fails or stalls, it falls back —
ending at a model running locally that will always answer. The effect is an
assistant that stays up and stays useful even when the primary is unavailable.

Getting that chain right took more tuning than I expected. Cold starts on the
fallback were brutal — the first request after an idle period could take long
enough that it felt broken. The fix was a warm-ping: a tiny scheduled request
that keeps the fallback awake, so it's ready when I actually need it.

## The bug that kept coming back: self-update ate my patches

Here's the one I'd put on the whiteboard. Hermes can update itself. Great —
except the update process pulled a clean copy and quietly overwrote the local
patches that made it behave the way I wanted. I'd fix something, it would work
for days, then an update would silently revert it and I'd be debugging a
"regression" that was really my own automation undoing my own work.

The lesson was one I keep re-learning: **anything that rewrites your code needs
to know what it's not allowed to touch.** The fix was to stop treating my
customisations as edits to upstream files and instead layer them so an update
can't clobber them. Since then, updates are boring — which is exactly what you
want from an update.

## Letting it fix itself

Once it was stable, I gave Hermes a command that lets it edit its own code,
compile-check the change, and restart itself cleanly in the background. It sounds
reckless, and it would be without the compile gate — that check is the whole
safety net. If the change doesn't build, it doesn't ship, and the running version
stays up. This is the feature I'm proudest of, and also the one I test the most
carefully.

## Watching the watcher

The last piece: Hermes can't be trusted to report its own death. If the VPS
falls over, a service on the VPS is in no position to tell me. So a second,
completely separate little machine at home polls it from the outside every few
minutes and messages me if it goes quiet, plus a daily heartbeat so I know the
monitoring itself is alive. Out-of-band monitoring is one of those lessons you
only learn once, usually the hard way.

## What I'd tell someone starting this

- The chat loop is a weekend. The reliability is the project.
- Assume every external dependency will fail, and design the fallback first.
- Anything that can rewrite your system needs a hard boundary and a safety gate.
- Monitor from outside the thing you're monitoring.

*Tech: Python, a chat bot API, systemd, a VPS, a Raspberry Pi for out-of-band monitoring.*
