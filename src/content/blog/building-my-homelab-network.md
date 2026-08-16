---
title: 'Building my home lab network from scratch'
description: 'How I designed and wired a segmented home network with UniFi gear, a Raspberry Pi service node, Cat8 cabling, VLANs, and a real security posture — and the two problems that nearly broke it.'
pubDate: 'Jun 18 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

I wanted a home network I could defend, not just one that gave me Wi-Fi. So I tore out the ISP router's flat setup and built something with structure: a proper gateway, a managed switch, a dedicated access point, a Raspberry Pi doing real work, and separate VLANs so a compromised smart doorbell can't see my laptop. This is the story of designing and wiring that from nothing.

## The gear and the plan

The backbone is a UniFi stack. The gateway does routing, firewalling, and intrusion prevention. A PoE switch feeds everything downstream and powers the access point over the same cable that carries its data, which means one run instead of two. The access point handles Wi-Fi on both bands. Sitting on the switch is a Raspberry Pi 4 that I turned into the network's service node.

Before I bought a thing, I sketched the segmentation. The core idea is that not every device deserves to talk to every other device. My computers, my phones, my IoT junk, and a lab zone for security testing each got their own VLAN, with firewall rules between them that deny by default. The IoT VLAN can reach the internet and nothing on my trusted network. The lab VLAN is walled off hardest of all, because the whole point of a lab is to run things I don't trust.

## Running the cable

I ran Cat8 for the fixed backbone. Honestly, Cat8 is overkill for a home internet line that tops out well under a gigabit, and I knew that going in. I did it anyway because the cable is the one thing you don't want to redo — pulling it through walls and trunking is the painful part, so I future-proofed the physical layer even though today's traffic will never touch its ceiling. The terminations taught me patience. Cat8's tighter tolerances are less forgiving than the Cat5e I'd crimped before, and my first two ends failed a continuity test because I'd let the pairs untwist too far back in the connector. Retwisting and reseating fixed it. Lesson learned: on higher-category cable, the last centimetre matters more than the whole run.

## The Pi earns its keep

The Raspberry Pi became my DNS and ad-blocking node, running AdGuard Home in front of a local recursive resolver. Every device on the network points its DNS at the Pi, which means ads and tracker domains get blocked for the whole house at the network level, before an app ever loads them, and my resolver does the lookups itself instead of handing my browsing history to a third party. I hardened the Pi properly: host firewall on, SSH locked to key-only with passwords disabled, and Fail2ban jailing any address that fails to log in. For access when I'm away, the Pi joins a mesh VPN so I never expose SSH to the open internet.

That mesh VPN caused my second real problem. My first remote-access attempt used a plain VPN tunnel with a port forwarded on the gateway, and it kept dropping. The cause turned out to be upstream filtering I didn't control, silently killing the traffic. Rather than fight it, I switched to a mesh VPN that relays over standard encrypted web traffic, which sails through the same filtering that had been blocking me. The defensive takeaway stuck with me: an inbound port forward is an attack surface you have to babysit, and a mesh VPN that only makes outbound connections is both more reliable and smaller as a target.

## Security posture

On the gateway I turned on intrusion prevention in blocking mode, not just detection, so it actually drops malicious traffic instead of politely logging it. DNS leaves the network encrypted. I also stood up a honeypot on an unused address: nothing legitimate should ever connect to it, so any device that does is either misconfigured or hostile, and that's a signal worth an alert. It's cheap tripwire security, and tripwires are how you catch the quiet stuff.

What I actually built here is the thing employers ask about in IAM and security interviews: I can point to a network I designed, segmented, wired, and hardened myself, and explain every trade-off I made. The overkill cable, the VPN I abandoned, the deny-by-default rules — each one is a decision I can defend. That's worth more than any certificate line.
