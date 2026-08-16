---
title: 'Building Hermes: a self-hosted AI agent on a cheap VPS'
description: 'I wanted an AI assistant I fully control — reachable from Telegram, running on my own box. So I built one.'
pubDate: 'Aug 16 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

> Draft — fill in the "what broke" section and hit publish.

I wanted an AI assistant I fully control: reachable from Telegram, running on my
own hardware, that never refuses a legitimate security question. So I built one.

## What I actually built

- A Telegram bot wired to a Python bridge running on a Hetzner VPS
- Model routing with a fallback chain, so it stays up when one provider rate-limits
- A `/code` command that lets the agent edit and restart its own code
- External health monitoring from a separate Raspberry Pi, so it alerts me if the VPS goes dark

## The bit I got stuck on

*(This is the part recruiters actually read — tell the real story.)*

- What broke — the crash loop? A self-update that wiped my patches?
- How I diagnosed it
- The fix, and why it worked

## What I'd tell someone starting this

- One or two honest lessons.

*Tech: Python, Telegram Bot API, systemd, Hetzner, Tailscale.*
