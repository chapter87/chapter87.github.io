---
title: 'The servers running in my house, and why I killed SMS 2FA'
description: 'A tour of the services I self-host on one NAS — photos, passwords, media, and backups — how they stay reachable without being exposed to the internet, and why I ripped out text-message two-factor auth in favour of authenticator codes.'
pubDate: 'Jul 21 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

Over the last few months a single box in my house quietly replaced a stack of cloud
subscriptions. It's a NAS running an open, ZFS-based operating system, and on top of
it sits a small fleet of services that between them do what I used to rent from four
different companies. This is what runs on it, how it stays secure, and one hardening
decision I'd urge on anyone: getting rid of SMS two-factor authentication.

## What's actually running

- **Photos** — a self-hosted photo app that gives my family the auto-backup timeline
  and face search of the big cloud services, with the library sitting in an encrypted
  dataset on my own drive. ([The full story is here.](/blog/self-hosted-photo-cloud-immich-truenas/))
- **Passwords** — a Bitwarden-compatible vault server, so my password manager's data
  lives at home while I keep the polished official apps. ([Write-up here.](/blog/self-hosted-password-manager-vaultwarden/))
- **Media** — a Jellyfin server for my own media library, streamed to the TVs in the
  house and to my phone when I'm out.
- **Backups** — a backup server that pulls my virtualisation lab's machines in
  automatically, so the whole home lab has real, versioned restore points.

One box, four services, no monthly bills, and every byte of it under my control.

## The architecture rule: reachable, not exposed

The thing that makes self-hosting a security *upgrade* rather than a downgrade is a
single principle: **none of these services listen on the public internet.** There is
no port forwarded on my router, no admin page a stranger can find by scanning.

Instead, everything is reachable over a private mesh VPN. My phone can back up
photos and open my vault whether it's at home or on mobile data, because it reaches
the NAS over that encrypted overlay — but to the rest of the internet, the server
simply isn't there. That distinction, *reachable by my devices versus exposed to
everyone*, is the whole game. On top of that the NAS lives on its own locked-down
network segment, so even inside the house it only accepts connections from the
specific machines that are supposed to manage it.

## Why I ripped out SMS two-factor auth

Now the part I most want people to copy. For years the "extra security" everyone was
told to turn on was a code texted to your phone. It is far better than nothing — but
it is the *weakest* form of two-factor auth, and it's the one I've now removed
wherever I can.

The problem is that a text message isn't really tied to you; it's tied to your phone
*number*, and a phone number is disturbingly easy to steal. In a **SIM-swap** attack,
someone convinces (or bribes) a mobile carrier to move your number to their SIM.
The moment they do, every one of those "secure" login codes gets delivered to the
attacker, and your text-protected accounts fall like dominoes. It's not theoretical —
it's a well-worn playbook, and it's why serious targets get emptied despite "having
2FA on."

So on my servers I switched to **authenticator-app codes** (TOTP): a six-digit code
generated on a device I physically hold, from a shared secret that never travels over
the network and can't be redirected by anyone talking to my phone company. No SMS
path exists to attack. Where an account offers it, a hardware security key is better
still.

The one discipline this demands: **set up your escape hatches first.** App-based 2FA
means if you lose the authenticator, you're locked out — so before enforcing it I
made sure I had recovery codes stored safely and a separate key-based admin path, so
a lost phone is an inconvenience, not a catastrophe. Turning on strong auth without a
recovery plan is how people lock themselves out of their own house.

## The takeaway

Self-hosting isn't about saving a few pounds a month, though it does. It's about
moving the trust boundary to your own front door and then defending it properly:
nothing exposed to the internet, everything behind a private network, and the
strongest practical authentication on every door — which in 2026 means an
authenticator app or a hardware key, and *not* a text message.

*Tech: TrueNAS, Jellyfin, self-hosted photos + passwords, a backup server, a mesh VPN, TOTP 2FA.*
