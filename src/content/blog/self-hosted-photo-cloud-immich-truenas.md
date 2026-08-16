---
title: 'I replaced Google Photos with a NAS I control'
description: 'Family photos are the one thing you never want to lose and never want leaked. So I stopped renting cloud storage and built my own — TrueNAS, ZFS encryption, and Immich — then spent an evening debugging why my wife could not log in.'
pubDate: 'Jul 30 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

Every photo of my family lived in someone else's cloud. That's a strange thing
to be comfortable with: the pictures you'd run back into a burning house for,
sitting on a server you don't control, under a policy that can change any time.
So I built a replacement I own end to end.

## The foundation: TrueNAS on ZFS

I started with a small NAS box and, instead of the vendor's stock operating
system, wiped it and installed TrueNAS — an open, ZFS-based system. That choice
mattered for two reasons. First, the stock firmware phoned home and carried a
history of nasty security holes; getting rid of it removed that whole surface.
Second, ZFS gives me things a cloud subscription never will: checksummed data
that detects silent corruption, snapshots, and **native encryption on the
datasets that hold photos** — so even someone with physical access to the drive
gets ciphertext, not memories.

## The app: Immich

For the actual photo experience — the app on your phone that auto-backs-up
everything and gives you a searchable timeline — I run [Immich](https://immich.app),
an open-source project that's genuinely competitive with Google Photos. It does
machine-learning search and face grouping, all on my own hardware, nothing
leaving the house.

One deliberate detail I'm glad I got right: the photo library lives *inside* the
encrypted ZFS dataset, so the images inherit encryption automatically. (A trap
worth flagging: the app's database — filenames, locations, face data — sits
separately, and if you care about that metadata you have to make sure *it's*
encrypted too. Encrypting the photos but not the index that describes them is a
half-measure.)

## The evening it didn't work

Here's the part that actually taught me something. I'd set it all up, my phone
connected fine, and then my wife tried to log in — and the app simply refused to
accept the server address. Not a wrong password. It wouldn't even validate the
URL.

I spent a while poking at the app before stepping back and asking the right
question: *could her phone even reach the server at all?* It couldn't. Her phone
had joined a different Wi-Fi network in the house — one that sits outside my main
router entirely, with no route to the server. No setting in the app could have
fixed that. The packets had nowhere to go.

The fix that made it robust everywhere — home or out on mobile data — was to put
her phone on my mesh VPN, so the server is reachable over an encrypted path
regardless of which network she's on. That also quietly solved a second problem:
photo backup no longer stops the moment she leaves the house.

**The lesson I keep relearning:** when something "won't connect," stop debugging
the application and prove the network path first. One command that forces a
connection over each interface would have told me in ten seconds what the app's
error message obscured for far longer.

## Was it worth it?

Yes — but with eyes open. Self-hosting means *you* are the reliability team. RAID
is not a backup, encryption keys need somewhere safe to live, and a photo you
can't restore isn't backed up. I'll write those lessons up separately. But the
core trade is one I'd make again: the most irreplaceable data I own now lives on
hardware I control, encrypted, on my terms.

*Tech: TrueNAS SCALE, ZFS (native encryption, snapshots), Immich, a mesh VPN for remote access.*
