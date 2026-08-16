---
title: 'I built a camera that looks at things and tells you what it sees'
description: 'A tiny ESP32-S3 with a camera and a speaker, wired up to a vision model, that you point at an object and it describes it out loud. Getting it to actually talk was the hard part — the memory management on a chip this small is unforgiving.'
pubDate: 'Jun 15 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

I wanted to build the simplest possible "AI in the physical world" gadget: point
it at something, and it tells you what it is, out loud. No phone, no screen — just
a little box with a camera and a speaker. The hardware is an **ESP32-S3 AI camera
board** with a small I2S amplifier. The whole thing fits in your palm.

The idea is easy. Making a microcontroller with a few hundred kilobytes of working
memory both *see* and *speak* over an encrypted connection is where it got
interesting.

## Getting it to talk

The camera part came together quickly — capture an image, send it to a vision
model, get back a description. The **speaker** fought me for a while. The audio
library reported that it was happily playing sound, and produced silence. Total,
confident silence.

The fix was to stop trusting the higher-level audio library and feed the speaker
raw audio directly: request the speech from the text-to-speech service in a raw
PCM format and stream those 16-bit samples straight to the I2S output, at the
speaker's native sample rate with no resampling in between. The moment I stopped
letting a layer "helpfully" convert the audio and just handed the hardware exactly
what it wanted, it spoke.

## The real enemy: memory

On a chip this small, the encrypted (HTTPS) connections needed to reach the vision
and speech services are *expensive* — each one wants a big chunk of memory for the
TLS handshake. Do vision and speech naively, both holding secure connections at
once, and you run out of heap and crash.

Two things made it stable:

- **Route big allocations to the external PSRAM.** Setting a threshold so anything
  over a kilobyte comes from the spare memory chip instead of the tiny internal
  heap took the pressure off the part of memory the network stack needs.
- **Never hold both secure connections open at once.** I made the device finish and
  *tear down* the vision request — freeing all that TLS memory — before it opened
  the connection to speak. Buffer the "what to say," then say it. Sequential, not
  simultaneous.

That's the whole embedded-systems mindset in one project: on a big computer you'd
never think about any of this; on a microcontroller, *when* you allocate and free
memory is the difference between a product and a crash loop.

## What I took from it

The AI part is almost a commodity now — anyone can call a vision model. The
engineering that's still hard, and still satisfying, is squeezing it onto real,
constrained hardware and making it reliable. A talking camera is a toy. The
lessons underneath it — give hardware exactly what it expects, and respect the
memory budget — are not.

*Tech: ESP32-S3, I2S audio (MAX98357), PSRAM, TLS on embedded, PlatformIO, a vision + TTS model.*
