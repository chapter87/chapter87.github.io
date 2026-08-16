---
title: 'Building a verified Tails USB — an amnesic computer in your pocket'
description: 'Tails is an operating system that runs entirely from a USB stick, routes everything through Tor, and forgets everything when you unplug it. I built one properly — including the verification step most people skip, and the reason it matters.'
pubDate: 'Jul 28 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

Tails is one of the most quietly remarkable pieces of software out there. It's a
complete operating system that boots from a USB stick, routes all its internet
traffic through the Tor network, and — this is the clever part — leaves no trace on
the computer it ran on. Unplug it and everything is gone, as if it never happened.
It's the tool journalists, activists, and privacy-conscious people reach for when
they need a clean, anonymous computing environment. I built one, and the build itself
is a good lesson in doing security tasks *properly* rather than just quickly.

## What Tails actually is

Two ideas make Tails special. It's **amnesic**: by default it runs entirely in
memory, so when you shut down, nothing you did is written to the host machine's disk.
And it's **anonymous-by-design**: every connection is forced through Tor, so your real
network identity stays hidden. You can optionally carve out an encrypted "persistent
storage" area on the same stick for files you *do* want to keep between sessions — I
left room for that — but the default posture is to remember nothing.

The point of all this is a threat model most software ignores: what if the computer
you're using isn't yours, or isn't trustworthy? Tails lets you bring your own clean,
private environment to any machine and take it away with you.

## The step everyone skips: verification

Here's the part that turns "burned a USB stick" into "did this properly." When you
download an operating system whose entire purpose is protecting you, you have to
answer one question before you trust it: **did I actually get the real thing, or
something tampered with in transit?**

So before writing anything, I verified the download's cryptographic signature against
the Tails developers' signing key. That check confirms two things at once — that the
image hasn't been altered, and that it genuinely came from the Tails project and not
an impostor. Skipping this is the single most common mistake people make with
security tools: they download the thing whose job is trust, and then trust it blind.
A privacy OS you got from a compromised mirror is worse than useless.

Then, after writing it to the stick, I did the second verification people skip: I
**read the data back off the USB and compared its fingerprint to the original.** A
write command reporting "success" does *not* prove the bytes actually landed
correctly — flash media lies, cables flake. The only proof is to read it back and
check it matches. It did.

## The little hardware gotchas

A couple of practical notes from the build, because the physical layer always has
surprises. The write speed doubled once I moved the stick to a faster USB port — worth
checking before you blame a "slow" drive. And booting it needed the right key to
reach the boot menu on my machine, plus an awareness that a modern dedicated graphics
card can leave you staring at a black screen unless you nudge the boot options, since
a privacy-focused OS ships only open-source graphics drivers. None of that is
difficult; all of it wastes an evening if you don't expect it.

## Why bother

I don't need to disappear. But building a Tails stick taught me the discipline that
matters everywhere in security: **verify before you trust, and verify that the thing
you built is actually the thing you intended.** Those two habits — checking a signature
before running code, and checking that a write really wrote — are exactly the muscles
you want for any serious security work. And there's something genuinely reassuring
about having a clean, private, forget-everything computer that fits on a keyring,
ready whenever a situation calls for one.

*Tech: Tails, Tor, GPG signature verification, checksum read-back verification, encrypted persistent storage.*
