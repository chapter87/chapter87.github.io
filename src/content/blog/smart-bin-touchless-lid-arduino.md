---
title: 'The smart bin: a touchless lid, and the small bugs that teach you electronics'
description: 'A university IoT project — an Arduino bin that opens its lid when you wave your hand near it. Simple on paper, but the servo browning out, the lid auto-cycling, and a flaky sensor each taught a real lesson about building things that touch the physical world.'
pubDate: 'Feb 26 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

For a university IoT unit I built a **smart bin**: wave your hand near it and the
lid opens on its own, holds for a second, and closes. No touching a grubby pedal.
It's the kind of project that looks trivial written down — an ultrasonic sensor, a
servo, a bit of code — and then teaches you three real lessons the moment you build
it, because the physical world doesn't care about your assumptions.

## The build

An **Arduino Uno**, an **HC-SR04 ultrasonic distance sensor** to see your hand, an
**SG90 servo** to lift the lid, and an **RGB LED** for status: green means ready,
cyan means the lid's moving or open. Wave your hand within about 20 cm and it
triggers. There's also a button to open it manually. Conceptually done in an
afternoon. Then reality showed up.

## Lesson 1: the servo that browned out

My first instinct was to drive the servo to a big opening angle so the lid swung
wide. Instead it *stalled* — the motor drew so much current at the extreme that it
dragged the supply voltage down and the whole board misbehaved. The fix was to
back off the open position to one the servo could reach cleanly (in servo terms,
capping the pulse width rather than pushing it to the limit). **A motor is a load,
not a line of code.** On a small supply, "just move it further" can crash the very
computer telling it to move.

I also had the servo *buzzing* and drifting even when the lid was supposedly still,
because a servo actively holds its position and jitters doing it. The fix was to
**detach** the servo once the lid finished closing — stop commanding it, let it
rest — which killed the jitter and saved power.

## Lesson 2: the lid that opened and closed forever

The first working version got stuck in a loop: open, close, open, close, with no
hand anywhere near it. The cause was obvious in hindsight — the sensor sees your
hand as "close," opens the lid... and the *open lid* is now the nearest object, so
it reads "close" again and never stops.

The fix was a tiny bit of state: the code now requires it to have seen the space
*clear* (a "far" reading) before it will count the next "near" reading as a
genuine, deliberate wave — plus a five-second cooldown after each close. That's the
difference between a sensor reading and an *intention*. A wave is "far, then near,"
not just "near." Encoding that turned a twitchy gadget into one that only responds
when you mean it.

## Lesson 3: never trust a single flaky reading

Ultrasonic sensors are noisy — the odd wild reading is normal. Acting on a single
ping made the bin jumpy. The fix was to take a few readings in quick succession and
average them, and to treat a no-echo timeout as "nothing there" rather than a
crash. I even built in a little pin auto-detection so a miswired sensor would try a
couple of common pin pairs before giving up — because on a breadboard project, "is
it the code or the wiring?" is the question you ask most, and anything that answers
it faster is worth the few lines.

## Why a bin was a good teacher

Nothing here is advanced. But every one of these bugs is a category you'll meet
again in anything physical: **power budgets** (the servo), **debouncing intent out
of noisy signals** (the wave logic), and **not trusting a single sensor reading**
(the averaging). A blinking LED on screen teaches you none of that. A lid that
won't stop flapping teaches you all three in one evening.

*Tech: Arduino Uno, HC-SR04 ultrasonic, SG90 servo, RGB LED, C++ (Arduino).*
