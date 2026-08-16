---
title: 'How I reach every device I own without opening a single port'
description: 'Port forwarding is how home labs get breached. I connect my laptop, servers, NAS, and phone with a mesh VPN instead — every device reachable from anywhere, nothing exposed to the internet. Here is the setup and the one mistake that quietly undermines it.'
pubDate: 'Jul 21 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

The old way to reach your home server from outside was to forward a port on your
router — punch a hole in the firewall and hope. That hole is also how a huge number
of home labs and NAS boxes get compromised: exposed management interfaces, scanned
and brute-forced around the clock by the entire internet.

I don't forward any ports. Instead, everything I own lives on a **mesh VPN** — a
private, encrypted network overlaid on top of the real one, where each device gets
a stable private address and can reach the others directly, from anywhere, with
nothing visible to the public internet.

## What it connects

My laptop, a cloud VPS, the home server and NAS, and my phone are all members of
one private mesh. From a coffee shop I can SSH into my server as if I were on the
sofa. My phone backs up photos to the NAS at home *and* over mobile data, because
the NAS address it uses is the mesh address, which works everywhere. None of these
services listen on the public internet at all — the only way to reach them is to be
an authenticated member of my mesh.

This also composes beautifully. My VPS is reachable *only* over the mesh — its
provider firewall drops public SSH entirely, so the single path in is the encrypted
overlay. And when my ISP started interfering with DNS, I fixed it by routing DNS
through this same mesh, out of reach of the local network. One private fabric,
many problems solved.

## The mistake that quietly undoes it

Here's the part I want anyone copying this to internalise, because it's the trap I
walked into. A mesh VPN like this often ships in **"allow everything" mode by
default**: every device on the mesh can reach every other device on every port.
That feels convenient, and it silently throws away most of the security benefit.

My NAS sits behind a carefully built network firewall at home. But it's also on the
mesh — and because the mesh was in allow-all mode, *every* node on it, including my
internet-facing VPS, could reach the NAS's admin interface directly, **completely
bypassing that home firewall.** I'd built a strong wall and left a private door
around the side of it, held open by a default I hadn't changed.

The fix is access control lists: define, explicitly, that only my trusted admin
devices may reach the NAS, and only on the ports they need — and keep the
internet-facing VPS firmly out. The important, slightly scary detail: the moment
you define *any* rules, the mesh flips from default-allow to default-**deny**, so
you have to enumerate every legitimate path that already works or you'll lock
yourself out of your own network. That's not a reason to avoid it. It's the whole
point — a network that denies by default and permits on purpose.

## The takeaway

A mesh VPN gets you the *reachability* of port forwarding with none of the public
exposure, and that alone is a big upgrade. But "reachable only by my devices" is
only true if you turn off allow-all and write the rules. Convenience defaults are
where good architectures quietly spring leaks — on your firewall, and on the
private network you built to back it up.

*Tech: Tailscale (WireGuard-based mesh), ACLs, SSH, a cloud VPS, defense in depth.*
