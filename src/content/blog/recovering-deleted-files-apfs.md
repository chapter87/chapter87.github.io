---
title: 'Can you still carve deleted files off a Mac? I built the tool and tested it'
description: 'File carving is the classic deleted-file recovery trick. On a modern Mac with FileVault and TRIM it mostly fails, and the reasons map cleanly onto how APFS stores data. So I wrote a small carver, worked image-first, and found the real boundary.'
pubDate: 'Oct 01 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

"Recover deleted files" is one of those phrases that sounds like a single thing and isn't. The tutorials all reach for the same tool, file carving, and the same promise: the data is still on the disk, you just have to go scrape it back. I wanted a straight answer to whether that's true on the machine I actually use, a modern Mac. So I built a small carver, tested it on my own kit, and the short version is: on the Mac's internal disk, carving is almost the wrong tool, and *why* is the interesting part.

## What carving actually is

When you delete a file, the filesystem usually removes the pointer to it, not the bytes. The directory entry goes, the space is marked free, and the content sits there until something writes over it. Carving ignores the filesystem entirely and reads the raw device looking for the shape of a file: a known magic number at the start and a footer at the end. A JPEG begins `FF D8 FF` and ends `FF D9`. A PNG starts with `89 50 4E 47 0D 0A 1A 0A` and ends with its `IEND` chunk. Find a start, find the next end, lift everything in between. No filesystem required, which is the whole appeal.

That model has one assumption baked into it, and we'll come back to it, because it's where my first version broke.

## Why it mostly fails on an APFS Mac

Three things on a current Mac each independently work against you.

FileVault is on by default, so the data at rest is ciphertext. Even if the deleted bytes are physically still there, carving them gets you encrypted noise unless the volume is unlocked and you're reading through the filesystem, at which point you're not really carving free space any more.

APFS is copy-on-write. Updates don't overwrite in place, they write elsewhere and move a pointer, which is great for integrity and snapshots and terrible for the carver's assumption that a deleted file sits in one contiguous run.

And the SSD issues TRIM. When a file is freed, the filesystem tells the drive those blocks are no longer in use, and the controller is free to discard them. After TRIM runs, the old contents can be genuinely, physically gone, returned as zeroes on the next read. This is the one people forget. On a spinning disk the data lingered; on a TRIM'd SSD it often doesn't.

Put together: on the internal disk the bytes are encrypted, probably fragmented, and quite possibly already erased by the drive. Carving is built for none of that.

## Where it does work, and the one rule

Carving still earns its keep, just not where the tutorials point it. It works on removable FAT and exFAT media, on unencrypted volumes, and on a raw image of any of those, because there the freed bytes really do persist in place until something overwrites them. A camera SD card, a USB stick, a disk image: those are the honest targets.

So I built the tool to work **image-first**, which is also the first rule of any recovery or forensics work: never write to the source. You take a read-only image of the media, and everything you do happens against the copy. Touch the original as little as possible, ideally not at all. My lab never operates on a real disk; it spins up a small sparse disk image, does its damage there, and carves that.

## Building it, and the bug that taught me something

The carver itself is short. Scan the bytes, for each supported type find every magic, find the matching footer after it, lift the span, hash it. I wrote the tests first, which is the only reason I caught the following.

I set up the end-to-end test properly: create an exFAT image, write a known JPEG and a known PNG with recorded SHA-256 hashes, delete both, detach the image, then carve the raw image and check the recovered bytes hash-match the originals. The PNG came back byte-for-byte perfect. The JPEG came back as 18,833 bytes when the original was 4,108, and the hash was wrong.

The cause is that assumption I flagged earlier, plus a sharp edge. `FF D8 FF` is only three bytes, and three bytes turn up by chance all over a filesystem's metadata. My carver found one of those random `FF D8 FF` sequences *before* the real photo, then ran forward to the first `FF D9` it could find, which happened to be the end of the genuine JPEG. It stitched a false start onto a real end and handed me a corrupt blob that was technically a valid-looking span and completely wrong.

The fix is what real carvers like PhotoRec already do: don't trust the bare magic, validate the byte after it. A genuine JPEG follows `FF D8 FF` with a real marker, an APPn segment (`E0` through `EF`), a quantisation table (`DB`), a frame header (`C0`/`C2`), and so on. A random `FF D8 FF` in the middle of a FAT table is followed by something that isn't a marker, so you skip it. I added a test for exactly that false-positive case, tightened the signature, and the end-to-end run went green: both files recovered, byte-exact, hashes matching.

That's the small lesson worth keeping. A signature that's too short doesn't fail loudly, it fails quietly, by giving you a file that looks recovered and isn't. Carving without validation is how you end up confidently handing someone a corrupted image.

## So what actually recovers a deleted file on a Mac?

Not carving the internal disk. The vectors that genuinely work on APFS are the ones built into the filesystem and the OS, in roughly this order:

**Snapshots.** APFS snapshots, including the local ones Time Machine takes, are a frozen read-only view of the volume at a point in time. If a snapshot predates your deletion, the file is right there inside it. Mount it read-only and copy the file back out. This is the single most reliable undelete on a Mac, and most people don't know they have them.

**The Trash.** Obvious, but it's genuinely the first place to look, and the per-volume `.Trashes` directories catch things the Finder Trash doesn't.

**Time Machine.** If the file existed at a backup, it's on the backup drive. Boring, reliable, the reason you run backups.

Carving sits at the bottom of that list on a Mac, and only comes into play for removable media or an image you've already taken.

## The takeaway

Two things, depending on which side of it you're on. If you're trying to *recover*, reach for a snapshot or a backup first and treat carving as a last resort for removable media, not a party trick for your SSD. And if you're trying to make sure something is *gone*: on a FAT or exFAT stick, deleting is not erasing, the bytes are still carveable until overwritten, so wipe the free space or the whole device. On an APFS Mac with FileVault and TRIM, deletion gets you most of the way there for free, which is a nice thing to be able to say and a slightly unnerving one.

I kept the carver small and image-first on purpose. The goal was never a product, it was to find the boundary between what the tutorials promise and what the hardware actually allows, and the boundary turned out to be exactly where the filesystem's design put it.
