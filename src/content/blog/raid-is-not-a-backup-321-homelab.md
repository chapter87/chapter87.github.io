---
title: 'RAID is not a backup: the day I found my domain controller had none'
description: 'I thought my homelab was backed up. Then I actually looked. The backup job was pointing at machines that no longer existed, the schedule never fired because the lab is usually powered off, and the one "second copy" was the same disk wearing a disguise. Here is how I rebuilt it properly.'
pubDate: 'Jul 27 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

"It's backed up" is one of the most dangerous sentences in tech, because it's
almost always said without checking. I said it about my homelab. Then I sat down
to actually verify it, and found the whole safety net had quietly rotted.

## What inspection turned up (not what I assumed)

My virtualisation host had a scheduled backup job, so I felt fine. Reading it
properly, three separate things were wrong:

1. **The job listed the wrong machines.** It named virtual machine IDs from an
   older build of the lab. Since then I'd rebuilt the important boxes — including
   the **domain controller that runs my entire identity lab** — under new IDs. The
   job was faithfully backing up VMs that no longer existed and skipping the ones
   that mattered. My DC had *zero* coverage.

2. **The schedule had effectively never run.** It was set for a fixed weekly time
   — but the lab is powered off by default. A time-based job on a machine that's
   asleep at that time simply never fires, and there's no catch-up. The newest
   backup on disk was weeks old.

3. **The "second copy" wasn't one.** What looked like network-attached backup
   storage turned out to be a re-export of the host's *own local disk*. One drive
   failure would have taken the "original" and the "backup" together.

Any single one of these would have been enough to lose the lab. All three at once
is the classic way real backups die: not with a bang, but with a config that
drifted out from under an assumption nobody re-checked.

## Rebuilding it around 3-2-1

The principle is old and correct: **3** copies of your data, on **2** different
kinds of media, with **1** copy off-site. I rebuilt toward that:

- **A real second copy** on a separate NAS, over a locked-down network path. One
  detail worth knowing: I forced the modern version of the file protocol, because
  the firewall only allowed the single port it needs — the older version quietly
  needs extra ports and would have failed in confusing ways.
- **A catch-up mechanism instead of a naive schedule.** Rather than "back up every
  Sunday at 2am," a small job runs on boot and daily, reads the *current* list of
  machines, and backs up only the ones whose newest copy is missing or older than
  a week. It's self-maintaining — add a VM and it's covered, no edit required — and
  it copes with a lab that's usually off.
- **Verification, not faith.** Every backup is integrity-checked and a checksum
  manifest is written next to it. A backup you haven't tested is a rumour.

## The bug that made the whole thing pointless: silence

Here's the one that stuck with me. A while later, a reboot left the backup drive
un-mounted, so the nightly copy silently failed for days. I only found it by
looking. **The backup system had no way to tell me it was failing** — its
alerting was never configured, so every failure happened in perfect silence.

That's the real lesson underneath all of this. Backups don't fail loudly. They
fail quietly, months before you need them, and the only thing that saves you is a
system that *shouts when it breaks*. So the last thing I built wasn't more backup
— it was monitoring: a daily health report that tells me pools are healthy and
the last copy succeeded, and, crucially, **goes silent if the whole thing is
down**. No message in the morning is itself the alarm.

## If you take one thing

Go check your backups right now — not that the job *exists*, but that it ran, that
it covers what you think it covers, that the copy is somewhere a single failure
can't reach, and that *something will tell you* the day it stops. RAID protects
you from a dead disk. It does nothing for a bad config, a fat-fingered delete, or
ransomware. RAID is not a backup.

*Tech: Proxmox VE, vzdump, TrueNAS, NFSv4, systemd timers, ZFS snapshots + replication.*
