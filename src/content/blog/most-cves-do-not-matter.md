---
title: 'Most CVEs do not matter. I built a way to find the ones that do.'
description: 'There are roughly a quarter of a million published vulnerabilities. You cannot learn from a firehose. So I threw almost all of them away and kept only the ones attackers are actually using — then labelled what kind of bug each one is. The single biggest category was not what I expected, and it pointed straight at the work I want to do.'
pubDate: 'Sep 9 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

I wanted to get properly fluent in vulnerabilities — not the headline of the week, but the
underlying patterns, enough to look at a new one and place it instantly. My first instinct
was the naive one: get all of them and study all of them.

That does not work, for two reasons. There are around a quarter of a million published
vulnerabilities, which is not a syllabus, it is a landfill. And the overwhelming majority of
them will never be used against anyone. Studying the full set means spending almost all of
your attention on bugs that do not matter, in order to occasionally trip over one that does.

So I inverted it. Instead of learning every vulnerability, I decided to learn only the ones
that are provably being exploited in the real world, and to organise those by the *kind* of
mistake behind each one. That is a small, sharp, high-signal set, and it turned out to say
something useful about where to point my career.

## The honest starting point

The request I actually made, to myself, was closer to "feed me every CVE so I get good at
this." It is worth being honest that this is not how learning works, for a person or a
machine. You do not get good at vulnerabilities by having them read to you. You get good by
seeing enough real examples that the patterns become obvious, and then by being tested on
them until recall is automatic.

So what I built is not a magic training set. It is two mundane, effective things: a curated
reference I can pull from on demand, and a drill that quizzes me on it. The value is entirely
in what got left out.

## Keeping only the weaponised ones

The filter is a public catalogue of vulnerabilities that are *known to be exploited* — the
ones with confirmed real-world attacks behind them, maintained as a live list. It sits at
under two thousand entries, against that backdrop of a quarter of a million. Every single
one has an attacker story attached. That is the entire point. If a bug is on this list, some
real intrusion used it; if it is not, it is theory until proven otherwise.

That one decision cut the study material by more than ninety-nine percent and raised the
quality of every remaining item. It refreshes itself daily, so new confirmed-exploited
entries show up on their own, each tagged with a separate score that estimates how likely it
is to be exploited in the near future. Ranking by "already used" and "likely to be used
next" is a far better use of attention than reading vulnerabilities in the order they happen
to be numbered.

## The label that reframed everything

Then I sorted each one by the type of flaw it is — authentication bypass, remote code
execution, path traversal, deserialization, and about ten other families. This is where it
got interesting.

The largest single category was not remote code execution. Everyone assumes it would be; RCE
is the one with the scary reputation. The biggest category, by a clear margin, was
**authentication bypass** — flaws that let an attacker skip the "prove who you are" step
entirely.

I did not plan that result, which is why it landed. The direction I have been steering
toward — identity, access, the machinery that decides who is allowed to do what — is not a
niche corner of security. Measured by what attackers are actually breaking in the wild, it
is the single most common way in. Roughly a fifth of the whole exploited set was also linked
to ransomware, which is the same lesson wearing a costume: the way in is usually a login that
should have said no and did not.

## Making it real instead of a reading list

A reference you only read is a reference you forget. Two things keep this one alive.

The first is drilling. I can pull a handful of the exploited vulnerabilities at random and
force myself to name the class and the core trick before checking. When I want depth, I take
one family at a time and work through it in the shape that actually sticks: the technique,
the trace it leaves behind, and the thing that stops it. Attack, indicator, defence. If you
cannot state the defence, you have not finished learning the bug.

The second is that a subset of these are not just readable, they are *runnable*. Dozens of
the exploited vulnerabilities have a ready, self-contained vulnerable environment available.
The daily refresh flags which new entries fall into that runnable subset, so the pipeline
does not just tell me "this is being exploited," it tells me "and you can safely stand this
one up and watch it happen."

## The rule I will not break

Every lab I stand up is bound to loopback only, and the setup refuses to run at all if any
part of it would be reachable from the network. I proved that discipline the boring way on
the first one: I deployed a genuinely nasty, widely-exploited flaw, checked that it answered
only on the local machine and was actively refused from everywhere else, then tore it down.
That check caught me nearly misreading an unrelated service's port as my own exposure — a
reminder to verify against what is actually listening, not what you assume is listening.

A vulnerability lab that is itself reachable is not a lab, it is an incident. Fail-closed is
the only acceptable default when the whole point of the exercise is to run something
dangerous on purpose.

## What I took from it

Less is the feature. The instinct to consume everything is the enemy of getting good at
anything, and the highest-leverage move here was deciding what to ignore. Filter to what is
actually being used, rank by what is likely next, label by the underlying pattern, and drill
until recall is automatic.

And sometimes the data quietly confirms a bet you have already made. I did not set out to
prove that identity was where the action is. I set out to organise some vulnerabilities, and
the biggest pile turned out to be exactly the thing I had been walking toward anyway.
