---
title: 'Every wireless attack I tested on one handheld RF multitool'
description: 'Deauth detection, BLE spam, sub-GHz replay, 2.4 GHz spectrum analysis, packet capture, and jamming research — all on a single open-source ESP32 device, all in my own lab. A field guide to what actually worked, what did not, and what the law says.'
pubDate: 'Aug 04 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

I've written before about the discipline side of this device — [verifying my
shielded box, trusting measurements over firmware claims](/blog/handheld-rf-tool-and-the-faraday-box/).
This post is the other half: the actual **techniques** I put through it. One
open-source ESP32 handheld, with Wi-Fi, Bluetooth, several 2.4 GHz radios, a
sub-GHz radio, and an IR transceiver, is basically a portable wireless-security
lab. Here's what I tested on each band and what I learned.

**First, the law, because it's the frame for everything below.** In the UK,
*receiving* and scanning is fine. *Transmitting* on many of these features is
restricted under the Computer Misuse Act and the Wireless Telegraphy Act, and RF
jamming is illegal to operate outright — no "own device" exemption. Everything
transmit-side here was against my own equipment, on battery, in a controlled
setup. Do not do the TX half of this list on anything you don't own.

## Wi-Fi

- **Monitor mode + packet capture.** Put the radio in promiscuous mode, hop the
  channels, and write real `.pcap` files to the SD card — thousands of management,
  data, and beacon frames in a sweep, opening cleanly in Wireshark afterwards. This
  is the foundation of all Wi-Fi analysis, and it's the *defensive* skill: you
  can't understand an attack you can't see on the wire.
- **Deauthentication detection.** The device flags deauth and disassociation frames
  in the air. Interesting real-world result: a quiet sweep at home still turned up
  dozens of deauth/disassoc frames — almost certainly my own access points doing
  band-steering and kicking weak clients, not an attack. Which is the whole lesson
  of a detector: it tells you frames are flying; *you* have to work out whether
  it's hostile or just your network being normal.
- **Beacon spam / SoftAP / injection.** These are the transmit-side Wi-Fi features
  — flooding fake network names, standing up a rogue access point, injecting raw
  frames. On this particular unit I proved (with an independent listening adapter)
  that the Wi-Fi *transmit path was actually dead* despite the firmware reporting
  success — a good reminder that "the tool says it worked" is never proof.
- **A regulatory gotcha worth knowing:** the chip defaults to a region that only
  scans channels 1–11, so it silently *misses* the channels 12–13 that are legal
  and used in the UK. If you survey Wi-Fi with a device like this and don't fix the
  country setting, you have a blind spot and won't know it.

## Bluetooth Low Energy

- **BLE scanning and sniffing** — enumerate nearby devices, watch advertisements.
- **BLE advertisement spam** (the "Sour Apple"-style flood of fake device popups)
  transmitted and was caught two ways: on a phone, and independently by a separate
  monitoring device. The genuinely useful part was the **detection fingerprint**:
  the flood showed up as dozens of brand-new random MAC addresses, each appearing
  for one to four packets and then vanishing forever — whereas real devices
  advertise steadily and accumulate packets over time. Even the randomiser wasn't
  uniform; the addresses clustered in a way that's itself a tell. **That's the
  defender's takeaway:** this attack is noisy and has a recognisable signature.

## Sub-GHz (the 433 MHz world)

This is the band of garage remotes, cheap doorbells, weather sensors, and a lot of
"IoT" that predates anyone thinking about security. The device can **capture and
replay** sub-GHz signals. The lesson here is sobering: an enormous amount of
real-world kit uses *static* codes with no rolling security, meaning a captured
signal replayed verbatim just works. Testing this on my *own* remotes is the
fastest way to understand why fixed-code RF is a bad idea and why rolling-code
matters.

## 2.4 GHz spectrum + the jamming research

- **Spectrum analysis.** The 2.4 GHz radios can scan the band channel by channel
  and show you what's busy — you can literally see your own Wi-Fi and Bluetooth
  occupying the spectrum. This is also how I discovered my "shielded" enclosure
  wasn't shielding anything: the in-box spectrum looked identical to the open room.
- **Jamming — researched, contained, and mostly a lesson in containment.** In a
  controlled bench test a narrowband interferer parked on a victim channel drove
  100% packet loss on a test link; a broadband sweep was far less effective per
  target (it only sits on any one channel briefly). But the headline finding was
  the *safety* one: because my enclosure gave no real attenuation, I stopped short
  of trusting any powered run as "contained." Jamming is illegal to operate in the
  open in the UK, full stop, and this is exactly why proving your containment comes
  *before* the interesting part.

## Infrared

The IR transceiver does capture-and-replay of remote-control codes — the TV-B-Gone
class of trick. Lower stakes, same principle as sub-GHz: a lot of everyday signals
are unauthenticated and trivially repeatable.

## What the whole exercise taught me

Cheap, capable RF hardware is here and anyone can buy it. That's the real security
message. But the value of owning one isn't the "attacks" — half of which are
illegal to actually transmit and several of which didn't even work on my unit. The
value is that every offensive feature has a **defensive mirror**: deauth *detection*,
BLE-spam *fingerprinting*, understanding why static sub-GHz codes and
non-PMF Wi-Fi are weak. You learn to defend a band by watching how it breaks — in
your own lab, within the law, trusting measurements over marketing.

*Tech: ESP32-S3, nRF24L01 ×3, CC1101 sub-GHz, 802.11 monitor mode + PCAP, BLE, IR, and a lot of reading of the Wireless Telegraphy Act.*
