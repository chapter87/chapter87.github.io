---
title: 'Playing IoT test engineer: evaluating a smart alarm through three iterations'
description: 'A university brief cast me as the test engineer for an IoT security alarm. Instead of just building one, I designed, built, and evaluated three progressively smarter versions — and learned that testing an IoT device is a discipline of its own.'
pubDate: 'May 24 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

One of my university units handed me a role rather than a task. The brief put me in
a fictional security company as an **IoT Test Engineer**, building and evaluating a
smart alarm for security guards. That framing changed everything: the deliverable
wasn't "make an alarm work," it was "prove, through testing, which design is
actually fit for the job." That's a genuinely different skill, and it's closer to
real engineering than most coursework gets.

## Three iterations, each testing an idea

Rather than build one alarm and call it done, I worked through three versions, each
adding capability and each raising a new question to test.

- **Iteration 1 — the basic alarm.** An ultrasonic sensor and a buzzer: something
  gets too close, it sounds. The thing worth testing here isn't "does it beep" — it's
  *responsiveness*. I made the alarm's urgency scale with distance, so the beeping
  gets faster as the threat gets nearer, and then tested where the thresholds should
  actually sit for a human to react in time.
- **Iteration 2 — zones and sensitivity.** I added a row of LEDs and a rising tone,
  turning a binary "safe/alarm" into a graded proximity scale with distinct zones.
  Now the test question was about *false alarms versus missed detections* — the
  fundamental trade-off in any sensor system. Too sensitive and it cries wolf; too
  relaxed and it misses the real intrusion. Finding that balance is exactly the kind
  of tuning a real security product lives or dies on.
- **Iteration 3 — armed / disarmed, with logging.** The final version added state:
  the alarm could be armed or disarmed, showed its status on an LCD with graded risk
  levels, and logged events. This is the jump from a gadget to a *system* — one that
  has modes, remembers what happened, and can be audited afterwards.

## Testing as the actual deliverable

The part that reframed the whole unit for me was writing it up as a test engineer
would. That meant a structured evaluation: what each iteration was supposed to do,
how I verified it, where it fell short, and a comparison across the three. It's the
difference between "I built a thing" and "I can tell you, with evidence, which
design meets the requirement and why."

That's a habit I've carried into everything since. A demo proves a thing can work
once. A test proves it works under the conditions that matter — the edge cases, the
false triggers, the failure modes — and documents it so someone else can trust your
conclusion without repeating your work.

## The IoT-in-security angle

Underneath the alarm, the unit was really about emerging technology in security:
what it means to put a network-connected sensor into a safety-critical role. That's
a serious question. An IoT alarm that can be jammed, spoofed, or simply left in the
wrong state is worse than a dumb one, because people *trust* it. Working through the
iterations gave me a concrete feel for why IoT security devices need the same rigour
as any other safety system — sensible defaults, clear state, logging you can audit,
and testing that actively tries to make them fail.

It's an undergraduate project built on a few pounds of parts. But the mindset it
drilled — build to a requirement, test to prove it, document so others can trust it —
is exactly the mindset I want to bring to security work for real.

*Tech: Arduino Uno, HC-SR04 ultrasonic sensor, LEDs, buzzer, 16×2 LCD, iterative test methodology.*
