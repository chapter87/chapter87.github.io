---
title: 'The password reset that never checked who I was'
description: 'A critical Keycloak flaw let an anonymous request finish another user''s password reset — no email link, no proof of identity. I reproduced it end to end in a sealed lab, patched it, and then found the more interesting problem: the whole attack left almost no trace in the logs. This is what I learned building the detection.'
pubDate: 'Sep 9 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

Keycloak is the open-source identity server a lot of organisations quietly run in the
middle of everything — it holds the logins, issues the tokens, and decides who you are
before any other system trusts you. So a critical, unauthenticated flaw in its
password-reset flow is about as bad as identity bugs get. An anonymous stranger, no
account, no credentials, ending up in control of someone else's login.

I reproduced it in a sealed lab against my own deliberately vulnerable copy, then patched
it and watched the patch hold. The reproduction was the easy half. The half that taught me
something was discovering that the entire attack ran without leaving a usable trace.

## Two harmless bugs that only bite together

The advisory points at one flaw. Reading the fix, it is really two, and neither is
dangerous on its own.

The first lives in the "Try another way" button — the one you click when the normal login
does not suit you and you want to pick a different method. When you press it, the server
scribbles a little note to itself: *show the method-picker screen next.* The bug is that
the note was not tied to the specific step you were on. It was a sticky note left on a
shared desk, and a later, unrelated step would read it and act on it.

The second is in the step that actually sets a new password after a reset. That step is
supposed to confirm one thing before it does anything: that the person finishing the reset
is the person the reset was for. It did not. It just said *success* and moved on.

Separately, these are nothing. Together, they let an unauthenticated request steer the
login flow into the "set a new password" step for an account that is not theirs, and then
finish it.

## The order is the entire trick

I stood up a vulnerable Keycloak in an isolated lab with a throwaway realm and a victim
account, confirmed a stranger could not touch that account normally, and started probing.

My first few attempts failed, and the failures were the instructive part. You cannot jump
straight to the password step — the server rejects it. You have to walk the flow in a very
specific order, and critically, you have to poison it *before you have told the server who
you are.* Requesting the reset for the victim forks the session behind the scenes, which
quietly invalidates the tokens from the fork and sends you back to a page that looks like a
dead end. It is not a dead end. Returning to the earlier, saved page re-triggers the sticky
note, and from there the flow walks itself into setting a new password for the victim.

I will not publish the step-by-step chain. The fix has shipped, and the point of doing this
was to understand the mechanism, not to hand anyone a tool. What matters is the shape of it:
the server confused *what step am I on* with *who is this*, and those are two questions an
identity system must never blur.

It worked. The victim's old password stopped working, my chosen password started working,
the account's credential timestamp was rewritten — and no reset email was ever sent or
consumed. There was no emailed link to click, because the flaw skipped the part of the
process the link is meant to gate.

## Version-vulnerable is not the same as exploitable

Before the reproduction, I had assumed any server on the affected version was exploitable.
That was wrong, and the distinction is worth keeping.

The built-in administrative realm on the very same server was never exploitable, because it
had password resets switched off entirely — and the reset flow is the only door this bug
opens. Same vulnerable code, zero exposure. If I had scanned for the version and reported
every instance as "critical, exploitable," I would have been wrong about a good number of
them. The realm's configuration decided the risk, not the version string. Checking the
actual setting before calling something exposed is the difference between a finding and a
false alarm.

## The bug I did not expect: the logs were empty

I re-broke the server on purpose to build a detection for it, fired the real attack, and
went to read the alerts. There were none. Not a rule bug — the server had produced no log
lines at all for the whole attack chain.

This turned out to be the most useful thing in the exercise. Keycloak was configured to
record security events, which I had assumed was enough. It was not. Its event logger writes
*failures* at a visible level and *successes* at a level below the default threshold. So a
failed login shows up, but "a reset email was requested" and "a password was changed" — the
exact events that describe this attack — were being generated and silently discarded before
they ever reached a log file.

Think about what that means. The single most sensitive sequence in an identity system,
somebody's password being changed, was invisible by default. An attacker could have walked
this exact path in production and left a clean record behind them. The fix is a one-line
change to the logging configuration, and I would now treat it as mandatory on any Keycloak
worth monitoring. But I only found it because I insisted on seeing the attack in the logs
before trusting my detection, instead of assuming the events were there.

Once the events actually flowed, the detection itself was simple: a reset email requested
for a user, followed within a couple of minutes by that user's password changing from the
same source. That correlation is the attack's signature. It is an *investigate this*
signal, not proof — a genuinely fast, legitimate reset can look identical, and whether the
emailed link was ever really clicked is simply not something the server records. Saying so
plainly is part of the job. A detection that pretends to be certain is worse than one that
is honest about what it can and cannot see.

## The patch, and the trap inside it

Patching was meant to be the boring part. It was not. The new version, unpacked as an
administrator, could not start — it needed to rewrite one of its own internal files on first
boot and did not have permission to, so it crash-looped and threw several minutes of
confusing, unrelated-looking errors downstream before I traced it back to a file-ownership
mistake I had made during the upgrade. Fixing the ownership and preserving the existing data
before swapping versions got it up in under a minute. With the patch in place, the attack
now dies at exactly the step where the fix adds the missing identity check, and the victim's
password survives.

## What I took from it

Three things stuck.

The dangerous bugs are often two boring bugs standing next to each other. Neither the sticky
note nor the missing check was scary alone; the combination was critical. Reviewing
components in isolation would have missed it.

An identity system has to keep *what are you doing* and *who are you* strictly apart, and
this flaw was the whole cost of letting them touch.

And a control you have not watched fire is not a control yet. "Events are enabled" was true
and useless. The reset flow was being defended by logs that were quietly throwing away the
one event that mattered, and the only way to know was to attack the thing and go looking for
the evidence.
