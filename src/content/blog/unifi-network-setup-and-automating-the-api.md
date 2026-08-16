---
title: 'Setting up a UniFi network — and driving it from its own API'
description: 'How I built and run my home network on UniFi kit: adopting the gateway, switch and access point into one controller, carving it into VLAN zones with default-deny firewalling, and then automating the whole thing through the controller API instead of clicking around a dashboard.'
pubDate: 'Jul 19 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

When I rebuilt my home network, I went with UniFi kit for one reason above all: it
puts an entire network — gateway, switch, access point — under a single controller,
managed like one system instead of three boxes with three web pages. This is how I
set it up, how I segmented it, and how I ended up managing it as code through its
API rather than clicking around the dashboard.

## Adoption: one brain for the whole network

The heart of UniFi is the controller. Each device — the security gateway that routes
and firewalls, the PoE switch, the Wi-Fi access point — gets *adopted* into it, and
from then on you manage the network as a whole. The switch powers the access point
over the same Ethernet run that carries its data, so there's one cable to each drop
instead of two. Everything's visible in one topology view: which client is on which
port, which band, how much it's pulling.

## Segmentation: zones that deny by default

The real work was carving the flat "everything can talk to everything" network into
zones. I created separate networks — each its own VLAN — for my trusted computers, my
IoT gadgets, and a dedicated zone for sensitive infrastructure like my NAS. Then I
built firewall rules between them on the principle that matters most: **deny by
default, allow on purpose.** My IoT devices can reach the internet and nothing on my
trusted network. My NAS zone accepts connections only from the specific machines
allowed to manage it, and drops everything else. A compromised smart plug can't see
my laptop, because there is no rule that lets it.

The trick with segmentation is that you have to *verify* it from the untrusted side.
It's easy to believe your VLANs are isolated; it's only true if, when you probe from
the IoT network, the trusted services genuinely don't answer. I tested exactly that,
and watched the firewall drop what it was supposed to.

## Wi-Fi and its security knobs

On the access point I tuned the things that actually matter: channel widths per band,
band-steering behaviour, and the wireless security settings. UniFi exposes modern
options like WPA3 and Protected Management Frames, and setting those correctly is
half of wireless security. It also taught me a real trade-off — turning PMF to
"required" is the right call against deauthentication attacks, but cheap IoT gear
often can't speak it, so on the IoT network I had to balance the ideal against what
the devices could actually support. Security settings that lock out the devices they
protect aren't security, they're an outage.

## Managing it as code

Here's the part I enjoyed most. Clicking through a dashboard is fine once; it's
miserable when you want to make the same change across devices, or script something
repeatable. UniFi's controller has an API underneath the web interface, and once I
understood how it authenticates — a session token plus a cross-site-request token the
interface uses on every call — I could drive it directly.

That turned network changes into something programmable. I could adjust an access
point's channel width, flip wireless settings, set a device's fixed address, or read
the live client list from a script instead of a mouse. It's the same shift that
infrastructure-as-code brings to servers: a change becomes something you can write
down, repeat, and reason about, rather than a sequence of clicks you hope you
remember next time. For someone heading toward identity and infrastructure work,
learning to treat the network as an API rather than a GUI was the most valuable part
of the whole build.

## Why UniFi was the right call

I could have built all of this with separate consumer boxes and a pile of
workarounds. Doing it on one managed platform meant I could think about the network
as a single designed system — zones, rules, radios, and automation — and explain every
decision in it. That coherence, more than any single feature, is why the whole thing
is something I can put my name to.

*Tech: UniFi (security gateway, PoE switch, Wi-Fi 6 access point), VLAN segmentation, zone-based firewalling, WPA3 / PMF, the controller API.*
