---
title: 'My television was calling home. I have the DNS logs.'
description: 'A story went around claiming hundreds of millions of smart TVs might be quietly listening. Instead of arguing about the headline, I checked my own network. My TV really had been sending data to its maker''s advertising and content-recognition servers — I can show you the exact records — and then I made it stop. Here is the evidence, and the line between what it proves and what it does not.'
pubDate: 'Sep 9 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

A story did the rounds recently claiming that up to a couple of hundred million smart TVs
could be secretly listening to the rooms they sit in. Headlines like that are built to be
argued about, and the argument is usually the wrong activity. The manufacturer denies the
scariest version, the researchers stand by a narrower one, and everyone shouts past each
other about a claim nobody in the conversation can actually check.

I can check one thing precisely: my own house. Every device on my network has to ask a
question before it can talk to anything on the internet — it has to look up the address of
the server it wants to reach — and I log every one of those questions. So rather than have an
opinion about two hundred million televisions, I went and read what mine had actually been
doing.

## What the story is really three stories

The "listening TV" panic bundles three separate things that deserve to be pulled apart,
because they are not equally true.

The first is data collection. Modern TVs from this maker scan the local network and note the
other devices nearby, run automatic content recognition to identify what is on screen, and
send that telemetry to the manufacturer's advertising arm. This is real and largely
documented; what is contested is the "always listening to audio" framing on top of it. It
only happens if the TV's own smart platform is online and the content-recognition and voice
features are switched on.

The second is a set of genuine security flaws in the TV's software that could let an attacker
already on your network do nasty things, including recording audio with the screen off. Some
of these are freshly disclosed; an older batch was already patched.

The third is the number itself. The headline figure is the total count of these TVs ever
sold, not a count of confirmed victims. It is a marketing statistic wearing a security
costume.

Separating those three is most of the work. Only the first one is something I can look for in
my own logs.

## What my logs actually showed

My network keeps a record of every address lookup every device makes, so I pulled the history
and searched for anything belonging to the TV's manufacturer.

It was there. Back in the spring, over a couple of days, my television had repeatedly looked
up the manufacturer's advertising and content-recognition servers by name — the ad-telemetry
host, the "service delivery platform" that drives on-screen recommendations and ads, and a
couple of the maker's regional back-ends, all resolving to the region I actually live in.
Every one of those lookups came from the TV and nothing else. This is precisely the
advertising and content-tracking behaviour the article describes, captured in my own house,
with timestamps.

Here is the part where I have to be disciplined, because this is exactly where these stories
go wrong. A record that my TV *looked up the address of* an ad server proves it was reaching
out to that ad server. It does **not** prove the TV was recording audio and shipping it off.
The lookups are hard evidence of ad and content-recognition telemetry. They are not evidence
of eavesdropping, and I am not going to inflate one into the other. What I can stand behind is
the narrow, provable claim: the television was talking to its manufacturer's advertising
infrastructure, and I have the records.

## Why it went quiet on its own

The other thing the logs showed is that all of this stopped months ago. Across the entire
recent window, there is not a single lookup to any of the manufacturer's servers, from the TV
or from anything else. The TV has made no address lookups at all lately, and it does not even
answer on the network right now.

The reason is almost funny. I put a separate streaming stick on that TV over the summer, and
from that moment the television became a dumb screen — the stick does all the streaming, and
the TV's own smart platform simply stopped connecting to the internet. The surveillance did
not stop because of a principled stand. It stopped because I accidentally cut off its network
access by changing how I watch things.

## The architecture problem hiding underneath

There is a more uncomfortable finding than the telemetry, and it is about where the TV sat
rather than what it did.

When it was online and busy phoning home, the television was on the same flat home network as
the phones, the laptops, and my main computer. Nothing separating them. That means the TV's
habit of scanning for and cataloguing nearby devices had a clear view of everything personal
on the network, and any exploitable flaw in its software would have had that same view to
work with. The convenient streaming stick, by contrast, lives on an isolated network segment
reserved for untrusted gadgets. The TV — the older, chattier, harder-to-patch device — was
the one with full run of the house.

That is backwards, and it is the actual lesson. The tracking is annoying; the placement is
the risk. A device you do not fully trust and cannot easily update belongs walled off from
the things that matter, not sitting next to them.

## Making it stop, without breaking it

Fixing this took two moves and a little care.

At the network level, I blocked the manufacturer's advertising and content-recognition
domains at the point where every device asks for addresses, so any attempt to reach them
resolves to nothing. Most of them were already being blocked by lists I run; the one that
mattered was the content-recognition platform, which was still resolving to live servers
until I added it explicitly. The care part is what I deliberately *left working*: the domain
the TV uses to fetch firmware updates, and its basic online check. Blocking an ad server is
good hygiene. Blocking a device's ability to receive security patches is how you trade a
privacy annoyance for a bigger security problem later.

On the TV itself, the settings that matter are the automatic content recognition feature
(often dressed up under a friendly name), the voice and far-field microphone options, and ad
personalisation. Turned off, with the smart platform kept off the network anyway because the
streaming stick does the real work, the television goes back to being a screen.

## What I took from it

Do not argue with the headline. Check your own house. A network that logs what every device
asks for turns a viral panic into a yes-or-no question you can answer with evidence, and the
answer for me was a qualified yes: real ad and content-recognition telemetry, no proof of the
scarier claim, and it had already gone dormant for an unglamorous reason.

The fix worth keeping is not the domain block, satisfying as it was. It is the realisation
that the least trustworthy thing in the room had been given the run of the network, and that
segmentation — keeping the devices you cannot fully trust away from the ones you rely on —
would have contained the whole problem before it started.
