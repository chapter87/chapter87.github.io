---
title: 'My ISP was quietly hijacking every DNS query in the house'
description: 'My home DNS just stopped resolving. Chasing it down led somewhere I did not expect: my internet provider was intercepting DNS traffic transparently, so even queries I aimed at a public resolver were being answered by them. Here is how I found it and routed around it.'
pubDate: 'Aug 10 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

I run my own DNS at home — an ad-blocking resolver on a Raspberry Pi that every
device points at. One day, it stopped resolving. Websites failed, apps hung, the
whole house was effectively offline despite the internet connection being fine.

The fix taught me something genuinely unsettling about how much a consumer ISP
can see and touch.

## The symptom

My resolver was configured to do full recursive resolution — walk the DNS tree
from the root servers down, itself, trusting no one. Suddenly every one of those
lookups came back **SERVFAIL**. Meanwhile the ad-blocker in front of it had a
fallback resolver configured, but it never kicked in, because SERVFAIL counts as
a "valid" answer, not a timeout. So every lookup in the house died in a way the
fallback was designed not to catch.

## The part I didn't expect

To isolate it, I tried querying a well-known public resolver directly —
essentially, "don't use my setup, ask a public DNS server straight." It answered.
Fine, so the public resolver works... except the answer wasn't coming *from* the
public resolver. **My ISP was transparently intercepting all outbound DNS traffic
on the standard port and answering it themselves**, no matter who I addressed the
query to.

That's the trick. A simple forwarder "works" — because the ISP happily answers it.
But genuine from-scratch recursion to the root servers doesn't fit that
interception, so it just fails. My resolver wasn't broken. It was being quietly
overruled by the network it sat on.

This is worth sitting with: on a default home connection, the list of every
domain every device looks up can be seen and shaped by the provider, even when
you've explicitly pointed your devices somewhere else.

## The fix: take DNS off the network they can touch

The way out was to stop sending DNS over the path the ISP intercepts. I already
run a small VPS and a mesh VPN connecting my home network to it. So I pointed my
home resolver's upstream at the VPS **over the encrypted tunnel**. DNS queries now
ride inside that tunnel to a server I control, resolve there, and come back — and
the ISP can't intercept what it can't read.

For resilience, if the tunnel ever goes down the query *times out* (a real
failure the fallback recognises), and the ad-blocker drops to an encrypted-DNS
resolver on a non-standard port that the ISP doesn't intercept. Two independent
private paths, no cleartext DNS on the local network at all.

## Takeaways

- **"I set a custom DNS server" is not the same as "my DNS is private."** On many
  home connections, port-53 traffic is transparently intercepted regardless of
  what you configured. Verify it; don't assume it.
- **Encrypted DNS (DoT/DoH) or tunnelling DNS over a VPN** is what actually moves
  those lookups out of the ISP's reach.
- When a fallback exists but never fires, check *what condition* triggers it. Mine
  was built for timeouts and the failure mode was SERVFAIL — so it sat there
  useless while everything broke.

*Tech: AdGuard Home, Unbound, DNS-over-TLS, a mesh VPN, a small VPS.*
