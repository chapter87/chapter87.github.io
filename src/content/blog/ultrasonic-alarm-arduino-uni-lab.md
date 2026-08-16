---
title: 'The sensor wasn''t dead: debugging an ultrasonic alarm on an Arduino'
description: 'A university electronics lab where I built a distance-based proximity alarm on an Arduino Uno, and spent most of my time learning that a "dead" HC-SR04 was really just wired backwards.'
pubDate: 'May 13 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

For one of my university lab units I had to build a small alarm system on an Arduino Uno. The idea was simple on paper: an ultrasonic sensor measures how far away something is, and the closer it gets, the more the board reacts. A row of LEDs lights up band by band as the distance shrinks, and a buzzer starts chirping when things get too close. Think of it as a very cheap parking sensor, or the "you're getting warmer" game turned into hardware.

The finished version has five LEDs, each mapped to a distance band, and a small buzzer that gives short soft chirps rather than one continuous screech. I dropped an earlier LCD screen from the design because it added wiring complexity without teaching me anything new, and the point of the lab was the sensing logic, not the display. What I want to write about, though, isn't the parts list. It's the two days I spent convinced my sensor was broken when it was working fine the whole time.

## The "dead sensor" that wasn't

The HC-SR04 ultrasonic module is one of the first sensors most people meet in electronics. It has four pins: power, ground, a trigger pin that fires the ultrasonic pulse, and an echo pin that reports back how long the pulse took to bounce off something. Wire those four correctly and it just works.

My problem was that I couldn't get a sane reading out of it. I'd wave my hand in front of it and get nothing, or garbage. I did what a lot of beginners do: I blamed the sensor. I assumed I'd been handed a faulty module, and I started planning how I'd demo the lab with fake distance values instead.

The real culprit was the pin order. Almost every tutorial online assumes the pins run in one particular order when you read the labels printed on the back of the board. But my specific module, once it was mounted on the breadboard with the two silver cylinders facing up, had its pins effectively mirrored. The pin I thought was ground was actually power, and vice versa. So I was feeding the thing back to front and then being surprised it didn't answer. The moment I trusted the physical orientation in front of me instead of the diagram in the tutorial, it read my hand at twelve to sixteen centimetres, first time, correctly.

## Is it the code or the wiring?

The bigger lesson underneath that saga is a question I now ask on every hardware project: is this the code, or is this the wiring?

When something doesn't work, those are two completely different worlds, and you waste enormous amounts of time if you keep tweaking one while the fault lives in the other. What saved me was isolating them. I wrote tiny throwaway sketches that each tested one thing. One sketch just turned all the LEDs on and left them on. Another blinked the onboard light to prove the board itself was running my code. Another sequenced through the LEDs one by one. Another did nothing but print the sensor distance so I could watch the raw numbers.

That "all LEDs on" test told me something important. When the onboard light behaved but a breadboard LED didn't, I knew the code was fine and the fault was in the jumper wires. Same logic for the sensor: once a bare distance-printing sketch still misbehaved, I knew to stop editing code and go stare at the physical connections. Each little test cut the problem in half.

## Tuning the bands

Once the sensor was honest, the last job was tuning. Deciding where one distance band ends and the next begins is not a maths problem you solve on paper; it's a feel problem you solve by moving your hand and watching. I ended up spreading the five bands across roughly an eighty centimetre range, and softening the buzzer to gentle chirps so it warned without being unbearable in a quiet lab room.

It's a beginner project and I'm not pretending otherwise. But it taught me a habit that scales far beyond a five-LED alarm: don't trust the diagram over the thing in front of you, and always separate "is it the code" from "is it the wiring" before you touch either.
