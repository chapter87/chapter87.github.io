---
title: 'A little flight radar for my son, built on a $5 clock'
description: 'My son wanted the tiny desk clock to show planes flying overhead like a radar screen. So I wrote custom firmware for its ESP8266 that pulls live flight data and draws a real scope. The bugs along the way were a great tour of embedded graphics.'
pubDate: 'Jul 09 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

My son pointed at a cheap little desk clock — one of those tiny ESP-powered gadgets
with a square colour screen — and asked if it could show the planes flying over our
house, "like a radar." That's the best kind of project brief: small, concrete, and
genuinely delightful if you pull it off. So I did.

The result is custom firmware running *on the clock itself*: it pulls live aircraft
positions from a free flight-tracking API, and draws a proper radar scope — home in
the centre, aircraft as blips on distance rings, a sweep line, callsigns, altitude
colours, and an alert when something passes directly overhead.

## Fitting it on an ESP8266

The clock runs an **ESP8266** — even more constrained than the ESP32s I usually
reach for. The big risk was whether it could even make an *encrypted* API call
without running out of memory, since flight data comes over HTTPS. It could, but
only with care: a generous receive buffer, streaming the JSON through a filter
instead of loading it whole, and fetching just the aircraft within a 30-mile radius
of home every few seconds. Memory stayed stable across hours of running.

## Three bugs that were really lessons

**The black screen.** My first firmware drew nothing at all. The display controller
needed a specific SPI mode that the obvious library defaults got wrong; the panel
was also mounted upside down and needed its colours inverted. None of that is
guessable — the fix came from finding a known-good configuration for that exact
panel and matching it. Embedded displays are fussy, and "it's the wiring" is often
"it's one config flag."

**The planes that wouldn't move.** Some blips sat frozen near the centre and
confused my son (and me). I pulled the raw data and found the culprit: they were
real aircraft, but ones *parked on the ground* at the airport a few miles away,
being drawn as if they were flying. The airport was close enough to be inside our
radar range. Filtering out anything marked on-the-ground cleaned it up — and it was
a nice reminder to always check your data against reality before blaming your code.

**The flickering sweep.** The rotating radar sweep flickered horribly, because the
little screen has no back buffer — every full redraw flashes. The fix was to stop
redrawing the whole screen each frame and instead only erase and repaint the thin
sweep line, restoring the rings and blips behind it. Change only the pixels that
change. That single idea is behind a huge amount of smooth graphics, and it's
satisfying to implement it by hand on hardware primitive enough to force you to.

## Why I love this one

No career angle, no security lesson — just a dad writing firmware so his kid can
watch planes on a five-pound clock. But it quietly exercised real skills:
constrained networking, embedded graphics, reading an API, and debugging by
checking assumptions against ground truth. The best learning projects are the ones
someone you love actually wants to exist.

*Tech: ESP8266, ST7789 display, Arduino, a free flight-tracking API over HTTPS, over-the-air updates.*
