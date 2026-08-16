---
title: 'A round smartwatch-style sensor dashboard on an ESP32'
description: 'A 1.28-inch round touchscreen, a 6-axis motion sensor, and an ESP32 — turned into a live gauge that reads real movement and temperature and renders it without a flicker. Small screen, surprisingly deep lessons.'
pubDate: 'Jun 05 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

I had a little round 1.28-inch touchscreen module built around an ESP32-S3, and I
wanted to turn it into something that felt like a real product: a smartwatch-style
dashboard with a sweeping gauge, a needle, and live readings from actual sensors on
the board. Cheap hardware, but getting it to look *smooth* taught me more about
embedded graphics and sensors than a bigger project would have.

## What it senses

The board carries a **6-axis inertial measurement unit** — an accelerometer and a
gyroscope in one chip — plus a **capacitive touch** controller under the round
glass. That IMU is the sensory heart of the thing: it reports orientation and
motion in three axes, and it exposes a temperature reading too. So the dashboard
isn't showing fake demo numbers; the needle moves because the sensor genuinely feels
the device tilt and move, and the temperature tile reads the chip's real environment.

Reading a sensor like this is its own small lesson. It talks over a two-wire bus,
and you don't just "get a number" — you configure its ranges, read raw values, and
turn those into something meaningful. Getting the first sane reading out of an IMU,
watching the value track as you tip the board in your hand, is one of those moments
where the abstract "sensor" becomes a real, responsive thing.

## The hard part was making it smooth

Here's what surprised me: the sensor was the easy bit. Making a needle sweep around
a colour gauge *without flickering* was the real challenge, and it's the same
problem I keep meeting on small displays that have no back buffer.

If you draw a moving needle the naive way — erase the old frame, draw the new one —
the screen flashes on every update, because for a split second it's half-drawn. The
professional fix is to render each frame **off-screen first**, into a memory buffer
(a "sprite"), and then push the finished image to the display in one shot. The screen
only ever shows complete frames, so the motion is buttery. On a round display it's
extra satisfying, because you also have to think in polar coordinates — the gauge arc
and the needle are angles and radii, not the rectangles you're used to.

The result is a gradient arc that runs blue through green to red, a needle that
tracks the live sensor value, a big centre number, and a row of little status tiles
along the bottom for connectivity, battery, and temperature — all redrawn many times
a second with no flicker.

## Why a tiny screen was worth it

None of this is going to change the world — it's a gauge on a two-inch circle. But it
packed in a genuinely useful stack of skills: reading a real motion sensor over a
hardware bus, turning raw sensor data into something a human can read at a glance,
and the off-screen-buffer trick that separates smooth embedded graphics from
flickery ones. Constrained hardware is the best teacher precisely because it won't
let you paper over any of it — if your rendering is lazy, the little screen shows it
immediately.

*Tech: ESP32-S3, 1.28-inch round GC9A01 display, QMI8658 6-axis IMU, CST816S touch, TFT_eSPI sprites.*
