---
title: 'I counted my machine identities. There were forty. I have one.'
description: 'Every person has a single human identity and a swarm of non-human ones — SSH keys, API tokens, service accounts — that nobody ever counts. So I counted mine. The number was forty to one, every one of them long-lived, and one of them had already reached somewhere it should not have. This is what an honest audit of your own machine identities looks like.'
pubDate: 'Aug 22 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

The identity you think about is the human one. Your login, your password, your face at the
laptop. It is exactly one identity, and the whole security industry is built around
protecting it.

The identities you never think about are the other kind. The SSH key that lets your laptop
into a server. The API token in a config file. The service account a container runs as. The
credential a scheduled job uses at three in the morning while you sleep. These are
*non-human identities*, and in a typical cloud environment they outnumber the human ones by
around forty-five to one.

Nobody counts them. That is the whole problem. You cannot protect what you have never
enumerated, and the breaches of the last two years — the big, named ones — were mostly not
stolen passwords. They were stolen tokens, leaked keys, and forgotten service accounts.

So I decided to count my own. Not read a report about someone else's — mine, on the actual
machines I use every day.

## The count

I wrote a small script that walks the usual hiding places — SSH keys, cloud credential
stores, container registry logins, `.env` files, the tokens that CLI tools quietly cache —
and reports what it finds. One rule, enforced in the code: it never reads, prints, or stores
a secret *value*. It only records structural facts. Is this key protected with a passphrase
or not? How old is it? Who can read the file?

On one laptop it found eighteen non-human identities. On one server, fifteen more. Add the
handful of remote accounts I already knew about, and the working total was around forty.

Forty machine identities. One human me.

But the ratio was not even the alarming part. Two other numbers were.

**Every single one was long-lived.** Not one short-lived, auto-expiring, or federated
credential in the entire estate. Forty standing keys to forty doors, none of which rotate on
their own. That is the exact risk category behind most token-theft breaches: a credential
that was valid two years ago is still valid today.

**Three of my SSH keys had no passphrase at all.** Two of them were root access to servers.
If someone copied those files, they would need nothing else — no password prompt, no second
factor. Just the file.

And ten `.env` files, several of them readable by any account on the machine, some over a
year old.

## The one that had already escaped

Here is the finding that turned this from an exercise into a lesson.

While testing some security tooling, I ran a program with no arguments and no credentials —
expecting it to fail. Instead, it silently picked up a cloud credential that was cached on
my laptop, authenticated on its own, and started reading data from an organisation I had not
told it to touch.

It stopped there, it was read-only, and no harm was done. But I had not *asked* it to do
that. A long-lived credential, sitting ambient on disk, was grabbed automatically by a tool
that never prompted me. That is non-human identity risk in one sentence: the credential's
reach was larger than I realised, and I found out by accident.

The audit I had just run had, in fact, already flagged that exact credential. I just had not
connected the two until it happened.

## Fixing it — and being honest about what I did not fix

The tempting move is to write the triumphant version where I rotate everything and end on a
clean scoreboard. That is not what happened, and the gap between the easy fixes and the hard
one is the actual point.

**The easy win, done immediately:** every `.env` file that any account on the machine could
read is now locked to owner-only. Seven files that were world-readable are now not. That is
pure improvement with zero risk — nothing legitimately needed the loose permissions, so
tightening them broke nothing.

**The hard one, deliberately left for later:** the three passphrase-less SSH keys. It would
have taken me thirty seconds to slap passphrases on them. I did not, on purpose. Those keys
are used by automation — scheduled jobs and an agent that logs into servers unattended. Add
a passphrase the naive way and every one of those breaks silently at the worst possible
moment. Doing it *properly* means introducing a key agent so the automation still works
while the keys on disk stay protected — and setting a passphrase I actually record safely,
not one I invent in a hurry.

So the honest scoreboard is: secret-file sprawl, closed. Long-lived unprotected keys, half a
plan and a calendar entry. That second line is not a failure to report — it is the difference
between security theatre and security. The dangerous version of this work is the one where
you "harden" something into a lockout and find out during an outage.

## What I would tell anyone

Run the count on yourself. You will be surprised, and the surprise is the value — you cannot
reason about a risk you have never measured. Then look at three things, in this order:

1. **Lifetime.** A long-lived credential is a standing liability. How many of yours expire on
   their own? If the answer is "none," that is your headline, not the count.
2. **Blast radius.** For each credential, what can it actually reach — and is that more than
   you assumed? The ones that surprise you are the ones that matter.
3. **Reach you did not grant.** The credential a tool picks up *without being asked* is the
   most dangerous kind, because its scope is invisible until something exercises it.

The human identity gets all the attention. The forty others are the ones that get you.
