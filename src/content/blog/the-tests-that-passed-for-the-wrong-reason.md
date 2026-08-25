---
title: 'The tests that passed for the wrong reason'
description: 'I built a sealed lab to detonate malware in, and wrote the tests first so I would know it was safe. Then the tests told me it was safe when it was not. Six times. Every one of them the same shape, and the fix turned out to be a single idea.'
pubDate: 'Aug 26 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

Last week I wrote about building a malware detonation lab test-first. Seven gates, all red to
begin with, build until they go green. That post ended honestly: most of it was still unbuilt.

It is built now. Two victims, a sealed network, telemetry reaching my monitoring stack, and a
drill where simulated ransomware encrypted twenty files while the alerts landed in the SIEM. All
of that worked.

This post is about the other thing that happened, which I think is more useful. Six times, a test
told me something was fine when it was not. They were all the same shape underneath, and one of
them would have been genuinely dangerous.

## The shape

Here is the pattern, in the abstract. A test wants to prove a negative — that something is
absent, blocked, unreachable, gone. The only evidence available is that a command produced no
output, or exited non-zero.

But a command produces no output for two very different reasons. Either the thing genuinely is
not there, or **the command never ran properly in the first place**. From outside, those look
identical.

Every single one of my false passes was a version of that.

## The one that mattered

The lab has a sealed network segment. A victim machine sits on it and must not be able to reach
anything — not the gateway, not the internet, not my monitoring server. Three tests, one per
target, each running a ping inside the victim and requiring it to fail.

All three passed. Green, green, green. Isolation proven.

Except the probes were running Windows ping syntax against a Linux victim. Every one errored out
on bad arguments before a single packet left the machine. A command that fails to start returns
non-zero, and my test read non-zero as "could not reach it".

Those three passes would have looked exactly the same on a victim wired directly into my home
network.

That is the one that would have hurt. I would have read "isolation proven", moved to the next
phase, and eventually run real ransomware on a machine I believed was sealed on the evidence of
three tests that had never tested anything.

## What actually fixed it

Not better ping syntax. The fix is structural, and it is the single most useful thing I have
learned this month.

**Before testing that something is unreachable, prove you can reach something.**

There is exactly one host the victim is allowed to talk to. So the suite now checks that first.
If the victim can reach the collector, the probe mechanism demonstrably works, and the failures
that follow mean something. If it cannot, the whole isolation suite refuses to run and reports
itself blocked rather than passing.

A positive control. Every lab scientist learns this at about age nineteen and I had to rediscover
it by nearly detonating malware on my own network.

The general form: **any test that infers absence from a failed command needs a positive control,
or it will eventually pass for free.**

## The other five, briefly

**Grep called my alert file binary.** Searching the SIEM's alert log returned nothing, and I
wrote the words "0 alerts matched" before checking. The alerts were there. Grep had decided the
file was binary, printed "binary file matches" instead of the line, and my parser discarded that
as noise. One flag fixed it. Nothing errored, nothing warned.

**A permission-denied read looked like an empty disk.** I checked whether a drive had been
wiped by reading its first sector and finding no non-zero bytes. There were no bytes at all —
I did not have permission to read the device. Empty output, read as empty disk.

**"Wiped" meant "signatures cleared".** Wiping a drive removes the partition table in about three
seconds, then spends hours overwriting the rest. My verification checked the first gigabyte,
which is cleared in those first three seconds. A wipe I killed at seven percent still reported
verified. I changed it to sample the start, middle and **end** of the disk — the end is the only
part that proves the write actually traversed the whole surface. That change then caught a real
partial wipe within the hour.

**An error message that matched no keyword read as success.** Checking whether a virtual machine's
guest agent was responding, I grepped its output for "error" or "not running". The actual message
was "No QEMU guest agent configured". No keyword matched, so the test went green — on a machine
that was powered off. Now it judges the exit code, because guessing at error wording is guessing.

**A read-only mount reported as writable.** Mounting a Windows disk to edit a file offline, the
mount command succeeded, so I said it was mounted read-write. It had silently fallen back to
read-only, because a hard power-off leaves the filesystem dirty. A successful mount is not proof
of write access. I only found out when the write failed.

## Why I am writing this down

The obvious lesson is "test your tests", which is true and useless.

The one I actually take from it is narrower and I think more transferable. **Silence is not
evidence.** Whenever a check concludes something is absent, blocked or clean because nothing came
back, ask what else produces nothing coming back. Usually the answer is: a broken check.

And in security work that failure mode is the dangerous direction. A test that wrongly fails
wastes an afternoon. A test that wrongly passes tells you a thing is contained when it is not.

My suite is at fifty-seven checks now and all of them pass. I trust that number more than I would
have a week ago, and specifically because five of those checks exist only to catch the ways the
other fifty-two could lie to me.
