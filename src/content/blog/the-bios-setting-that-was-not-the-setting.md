---
title: 'Wake-on-LAN was enabled. The network card was switched off.'
description: 'Two servers refused to wake remotely. Wake-on-LAN was correctly enabled on both, and the driver confirmed it. The real culprit was a second power setting hidden behind a name that does not contain the word it describes, and the experiment that found it took four minutes.'
pubDate: 'Aug 20 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

I have two small servers running a Proxmox cluster in a cupboard. One is an old Dell
small-form-factor desktop, the other a ThinkCentre. Between them they run a domain
controller, an identity provider, a SIEM, and whatever lab machine I am breaking that
week. When I am not at home, I want to be able to turn them on without asking someone
to walk over and press a button.

Wake-on-LAN is supposed to make that trivial. You send a magic packet, the network card
sees its own MAC address in the payload, and it pokes the motherboard awake. It is
thirty-year-old technology and it works on almost anything.

It did not work on mine. I would send a packet and get nothing back.
No boot, no ARP reply, nothing. The box just sat there in the dark.

## The thing that made it hard was that everything looked right

This is what I kept seeing when I checked the card:

```
Supports Wake-on: pumbg
Wake-on: g
```

That `g` is the important letter. It means the card is set to wake on receiving a magic
packet. Which is exactly what I was sending it. The operating system was telling me,
clearly and repeatedly, that the feature was armed.

The BIOS agreed. Dell exposes firmware settings to the operating system through SMBIOS
tokens, so you can read and write BIOS options without standing in front of the machine.
I read the Wake-on-LAN token. It was enabled. It had been enabled the whole time, and
had survived every reboot.

So: feature enabled in firmware, feature enabled in the driver, packet demonstrably
being sent. And nothing happened. I spent a genuinely embarrassing amount of time on
the assumption that the packet was the problem — wrong broadcast address, wrong subnet,
the router eating the frame, something on the wire.

## The experiment that ended the guessing

Eventually I stopped theorising and ran a proper test.

The key was that I had a third machine on the same network segment that I had never
had trouble with. So instead of testing one box at a time, I fired magic packets at all
three **from the same sender, at the same instant, to the same broadcast address**, and
watched for three minutes.

```
node-b   ->  UP after 79s   <-- control
node-a   ->  no response, 180s
desktop  ->  no response, 180s
```

That is a controlled experiment with a working control, and it is worth more than every
previous attempt combined. One machine woke up. It woke through the same
sender, the same broadcast address, the same packet format, the same switch, at the same
second as the two that did not.

Which means none of those things can be the problem. Not the broadcast address. Not the
subnet. Not the router. Not the packet. Every network-side theory I had been chewing on
died at once, and the remaining explanation had to be something specific to the two
machines that failed.

If you take one thing from this post, take that. When something intermittently does not
work, the highest-value move is usually not another attempt at the broken thing. It is
finding a version that *does* work and running both at once, so the difference between
them is the only variable left.

## What was actually wrong

Dell's SMBIOS tokens have human-readable names, so I dumped all 248 of them and went
looking for anything power-related. Searching for "Deep Sleep" — the setting I suspected,
because it is the classic cause of this exact symptom — returned nothing. Not one token
on the entire machine contains the string "Deep".

But this did turn up:

```
Token: 0x00f5 - Low Power Mode (Enabled)     value: bool = true
Token: 0x00f6 - Low Power Mode (Disabled)    value: bool = false
```

"Low Power Mode" is what this firmware calls Deep Sleep Control. It was on.

Deep Sleep does what the name suggests: when the machine powers off, it cuts power to
things that would otherwise keep drawing a trickle from the PSU. One of those things is
the network card. So the card was configured perfectly, and told the truth about being
configured perfectly, and then had its power removed the moment the machine shut down.
An armed card with no electricity in it.

That is why every diagnostic looked healthy. `ethtool` reports what the *driver* is
configured to do. The driver has no idea whether the card will still have power in ten
seconds. It was never lying to me — I was asking it a question it could not answer.

The fix was one command, run remotely:

```
smbios-token-ctl -i 0x00f6 --activate
```

## The part that would have wasted another evening

I flipped the token, shut the machine down, sent a packet. Nothing.

Dell latches token writes at POST. Writing the setting puts it in NVRAM, but the firmware
only reads NVRAM and reconfigures the hardware when it actually boots. A shutdown is not
a boot. So the new setting sat there, correct and completely inert.

The sequence has to be: write the token, **reboot so the firmware picks it up**, and only
then test a cold wake.

I suspect this is what happened on my first attempt the day before, when I flipped the
Wake-on-LAN token and concluded from the failed test that it had not worked. It may well
have worked. I just never gave the firmware a chance to notice.

With the reboot in the right place:

```
node-a fully DOWN at +180s
settling 25s in S5...
=== FIRING wake node-a ===
node-a: magic packet sent
node-a: UP after 39s
```

Thirty-nine seconds from dark to booting, over the network, with no physical access.
The other node cold-tested at eighty seconds. Both came back, the cluster reached quorum
on its own, and every guest restarted automatically.

## Turning it off is harder than turning it on

There is a second half to remote power control that nobody writes about, and I ran
straight into it while testing.

Waking a machine is one packet. *Shutting one down cleanly* turned out to be the slow,
fragile half. Two of my Windows lab VMs ignore ACPI shutdown requests entirely. When I
told the host to power off, the host politely asked each guest to stop, the guests
ignored it, and the host's init system sat there waiting on them for over five minutes
before giving up and killing them anyway.

Worse, during that window the machine is in a genuinely ambiguous state — it has stopped
accepting SSH but still answers pings. My test script was watching for the machine to go
dark before sending the magic packet. It never went dark in time, so the script gave up
waiting and fired the packet at a machine that was still running, which cheerfully
reported "already up". A completely wasted test run that looked like a result.

The fix was to stop treating host shutdown as one atomic thing. Instead of asking the
host to sort out its guests, each guest gets shut down explicitly with a bounded timeout
and a forced stop if it will not cooperate:

```
qm shutdown <id> --timeout 90 --forceStop 1
```

Same end state as letting the init system time out, but bounded, per-guest, and
predictable — ninety seconds, not "somewhere between two and eight minutes". I wrapped
that into a `lab-down` script that mirrors the `wake` script, so the lab now has a clean
command in both directions.

Two smaller things worth passing on from writing that script:

**`ping -W` means different things on different systems.** On macOS it is milliseconds,
on Linux it is seconds. I wrote `-W1` intending a one-second timeout, and on the Mac it
was a one-millisecond timeout, which reports every live host on the network as dead. If
you write host-liveness checks that might run on both, detect the platform.

**Do not build your recovery tooling on top of a machine that might be off.** My first
instinct was to reuse some existing remote-management scripts to shut the Windows guests
down properly. Then I noticed those scripts live on my desktop — a third machine, not in
the cluster, frequently powered off. Building lab shutdown on top of it would mean
needing one machine awake in order to put two others to sleep. The `--forceStop` approach
needs nothing that is not already running on the host itself.

## What I would tell myself a day ago

The setting you are looking at is enabled. That is not the same as the feature working,
because features have prerequisites that no single setting screen shows you. A network
card needs configuration *and* power, and those are two different toggles in two
different places, one of which is named after a thing it does not sound like it does.

And when a tool tells you something is fine, be precise about what it is actually
reporting on. `ethtool` was telling me the truth about the driver. I was the one
extending that into a claim about the hardware.

Stop re-running the failing case. Go and find a working one, and run them side by side.
