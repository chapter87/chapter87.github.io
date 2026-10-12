---
title: 'I tried to make a laptop untraceable. Here is what actually worked.'
description: 'I set up a new laptop from a blank disk and tried to make it untraceable. The honest answer is that "untraceable" is not a setting — it is a question about who you are hiding from. A build log, a self-pentest, and the parts that turned out to be theatre.'
pubDate: 'Oct 12 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

I had a laptop to set up from nothing — a Dell mobile workstation, blank disk, no operating system. The job in my head was simple and slightly romantic: build it so it leaves no trail. Make it untraceable. Start from the firmware, do it properly, don't cut corners.

I did build it properly. But the first useful thing I learned is that the goal I started with was the wrong goal, and most of the effort people pour into "untraceable" is spent defending against the wrong adversary. So this is both a build log and a correction to my own starting assumption.

## "Untraceable" is not a switch

Before any configuration, there is one question that decides everything, and almost nobody asks it first: **who are you hiding from, and for how long?** The right answer is wildly different depending on the honest answer to that.

- A **website or ad network** sees your address on the internet and a browser fingerprint. Those are beaten by blending into a crowd — not logging in, not being unique.
- Your **internet provider** sees every site you visit. That is beaten by routing through Tor or a trustworthy tunnel, and encrypting your DNS.
- A **person on the same café or hotel network** sees your laptop's hardware network address and the name it announces. That is beaten by randomising those.
- A **thief or a border officer holding the physical machine** sees everything unless the disk is encrypted. That is beaten by full-disk encryption and a firmware password.
- A **nation-state tapping the internet backbone at both ends of your connection** can correlate your traffic by timing alone. That is beaten by *nothing you can configure on a laptop.* It is a mathematical ceiling, and pretending otherwise is how people get a false sense of safety.

Once I wrote that list out, the romance drained away in a good way. My realistic, achievable goal became: **unlinkable to websites, invisible to my provider and the local network, and amnesic enough that nothing persists.** That is attainable. "Invisible to everyone, including state-level traffic analysis, from a laptop I use every day" is not. Decide your tier honestly and stop paying a tax for a threat you don't actually face.

One principle sits underneath all of it: **blend in, don't stand out.** Anonymity comes from looking identical to thousands of other people, not from being clever. Every "privacy tweak" that makes you *different* — a rare browser, an exotic font set, a custom screen size — makes you *more* identifiable, not less. I kept tripping over that paradox, so it's worth saying twice.

## The boring part that everything rests on

I started at the disk, not the desktop. A secure machine built on an install image you didn't verify is a secure machine built on trust you never checked.

So the install media was verified by cryptographic hash before it went anywhere near the laptop, and — the step almost everyone skips — verified *again by reading it back off the stick* after writing. That second check earned its keep immediately: one USB stick in my drawer was quietly dying, and it returned a different hash on every read, including the hash of nothing at all when it briefly dropped off the bus. If I had trusted the first read, I'd have installed from corrupt media and spent a week blaming the wrong thing. A stick that fails a readback is not a stick you reformat and reuse; it's a stick you throw away.

From there: a clean install with **full-disk encryption** so the powered-off machine is a brick to anyone holding it, and **Secure Boot left on** so nothing unsigned can slip underneath the operating system and undo every layer above it. The encryption header — the small block of metadata without which the disk is unrecoverable even *with* the passphrase — got backed up off the machine, because an encrypted disk you can't open is just an expensive way to lose your data.

## Starting from the firmware

This is the "from setup mode" part I was most excited about and where I learned the most about what is real versus what is theatre.

The wins were the mundane ones. Set a **firmware administrator password**, so nobody can change the boot order, disable Secure Boot, or boot off their own USB stick without it — that is what pairs with disk encryption to defeat the person holding the machine. **Disable network boot** so the laptop can't be coaxed into booting off the local network. Turn off the radios and interfaces you don't use.

Two things genuinely surprised me. The first: modern business laptops ship a firmware-level anti-theft and tracking module, and on current hardware its default has quietly moved to *enabled*. It only fully activates under specific conditions that don't apply to a Linux machine, but leaving a firmware-resident tracking and persistence module switched on by default is not something I wanted to discover later. I turned it off.

The second was a lesson in knowing when to stop. I wanted to go further — replace the proprietary firmware entirely, neutralise the hidden management engine that lives below the operating system on Intel platforms. On this generation of hardware, that road is closed. The manufacturer fuses a key into the silicon so the processor refuses to run unsigned boot firmware, which makes the open-firmware route effectively impossible without the vendor's signing key. The tools for stripping the management engine are only proven on older chips and will likely brick this one. The realistic win — confirming that the out-of-band management interface was never provisioned or exposed in the first place — I already had. Chasing the rest would have been high-risk effort against a threat I don't face. I wrote "don't bother" next to it and meant it.

## What still leaked after all that

Here is the humbling part. With the disk encrypted, the firmware locked down, the firewall dropping everything, and the radios off, I ran a traceability audit against the finished machine. It still leaked, in ways that had nothing to do with any of the work above.

The wired network port was announcing its **real, factory-burned hardware address** to every network I plugged into — the wireless side randomised its address, but the cable side didn't, so a single café switch would see the true one. My **browser fingerprint was unique** — the combination of graphics card string, screen size, fonts, and processor count was enough to pick me out of a crowd on its own. The browser was handing my public internet address to any page's JavaScript through a video-call feature most people have never heard of and never use. And my address on the internet **geolocated to street level** — close enough to matter.

None of those are fixed in the firmware or the disk. They live one layer up, and they are where tracing actually happens.

The network address was the quick one: randomise the hardware address on *every* interface, not just the wireless one — with a catch I nearly missed. If you randomise the hardware address but leave the network-lease identifier derived from it, the network re-links you anyway, and you've achieved nothing. You have to close both, or neither counts.

The fingerprint was the one that changed how I think. The instinct is to harden your everyday browser — pile on settings and extensions until it's a fortress. That makes it *more* unique, because every change is a new distinguishing mark. The thing that actually works is the opposite: use a browser engineered so every one of its users looks identical — the same graphics string, the same screen size, the same everything. A stock, untouched privacy browser beats a lovingly hand-tuned one for anonymity, every time, precisely because you disappear into its crowd instead of standing out from it.

## I attacked my own machine

Hardening you haven't tested is a hope, not a result. So once it was built I treated it as a target — scanned it from another machine on the network and from a server out on the internet, then actively tried to get in.

From the internet, it was gratifyingly dead: **zero open ports, nothing forwarded, no way in.** From the local network, the same. The only thing listening anywhere was key-only remote access, and a bogus login attempt got the door shut in its face.

The real finding wasn't remote at all. It was local, and it was mine. A process already running as my own user could become **root — full administrative control — without a password**, by abusing my membership in the virtualisation group, which could reach a system service that does its work as root. I proved it both directions, read a file only root can read and wrote a file only root can write, then closed it the same night: tightened the service's permissions and removed myself from the group. The lesson generalised past this one bug. I run a lot of semi-trusted code as my own user — automation, security tooling, browser agents — and the password prompt on administrative commands is the boundary that protects me from *code running as me that I didn't authorise*. Anything that quietly removes that prompt is worth more attention than a dozen firewall rules.

The second lesson I'll state as a principle rather than a confession, because it's the one that catches everyone: the thing that turns one compromised machine into a compromised *fleet* is reusable credentials sitting in plaintext — an access key with no passphrase is a skeleton key for everything it can reach. Moving those to hardware-backed tokens, where the secret never exists as a file at all, is now top of my list. A machine is only as contained as its weakest key.

## The limits, stated plainly

So I don't over-trust my own setup:

- **Traffic correlation beats Tor.** An adversary who can watch both ends of your connection can line up the timing and unmask you, and nothing on a laptop changes that. It's inherent to anything fast enough to browse with.
- **Your own mistakes beat all of it.** One login to a real account, one reused username, one photo with the location still baked in, one distinctive turn of phrase — any of those undoes every layer below it. People are caught by behaviour, not by broken cryptography.
- **A compromised machine sees everything before it's encrypted.** Which is the real argument for amnesic systems and disposable virtual machines: they shrink the window.
- **Encryption protects a powered-off machine, not a running one.** The keys are in memory while it's on. When it matters, shut it down — don't just close the lid.

## Where I landed

The honest conclusion is that trying to turn one everyday laptop into an anonymity machine is the trap. A machine you use for your actual life accumulates logs, histories, and habits that identify you no matter how it's configured. So I split the job. The daily machine is hardened for **privacy** — less tracked, encrypted, quiet on the network, nothing phoning home — and I accept that it is *me*. For anything that genuinely needs to be **anonymous**, that belongs on an amnesic system that forgets everything at shutdown, or an isolated environment that can't leak my real address even if something inside it is compromised. Different jobs, different tools, never crossed.

"Untraceable" turned out to be the wrong word for what I actually wanted, which was *unlinkable, invisible to the middle of the network, and amnesic when it counts.* That I can build. And the most valuable hours weren't the clever ones — they were the boring verification, the honest threat model, and the afternoon I spent attacking my own work and finding the hole that none of the firmware wizardry would ever have closed.
