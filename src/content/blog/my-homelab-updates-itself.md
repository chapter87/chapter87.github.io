---
title: 'My homelab updates itself now, through a pull request'
description: 'I replaced the way I update my home server. Instead of logging in and changing things by hand, a bot proposes each update as a pull request, I approve it, and the cluster reconciles itself to match. Getting the loop working end to end took an afternoon. Getting it to stop lying to me took the rest of the day.'
pubDate: 'Sep 9 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

Updating a server by hand is a bad habit that feels like control. You log in, you change a
version number, you restart something, and if it works you move on without writing down what
you did. Multiply that by a few services and a few months and you have a machine whose exact
state nobody can describe, including you.

I wanted the opposite: a setup where the written-down description *is* the server, where
every change is a reviewed proposal, and where updates arrive as suggestions rather than as
me poking at production at eleven at night. That pattern has a name — GitOps — and I built a
small version of it at home. The loop works. The interesting part, as usual, was everything
that quietly did not.

## What the loop actually is

Three pieces. A git repository holds the desired state of everything running — which apps,
which versions, which settings. A component inside the cluster continuously compares reality
to that repository and drags reality back into line whenever they drift; nobody applies
changes by hand, they edit the repo and the cluster catches up. And a bot watches the outside
world for new versions of the things I run, and when it finds one, it opens a pull request
that bumps the version.

So the daily rhythm becomes: a machine notices an update, proposes it, I read the diff and
approve, and the cluster updates itself to match. I never touch the server directly. If I
want to know what is running, I read the repository. If I want to roll back, I revert a
commit. The state is text, reviewed and versioned like any other code.

I set it up on a single small node at home and gave it a real first job rather than a toy —
a game server, something with actual users and a real reason to stay up. A real workload
surfaces real problems in a way a hello-world container never will.

## The updates were silently going nowhere

The first trap cost me the most time and produced no error at all, which is the worst kind.

The cluster came up healthy. Every individual piece reported fine. And every workload I
deployed just... never became reachable. No crash, no failure, no log line pointing at the
cause — traffic between the pieces simply vanished.

The culprit was the host firewall. It had the main network interface parked in its most
restrictive zone, which is a perfectly good default for a machine facing the world. But a
cluster runs its own private internal network for pods to talk to each other, and that
internal traffic was being dropped on the floor with the same silent prejudice as an
unsolicited packet from the internet. Nothing was broken. The firewall was doing exactly its
job, to the wrong traffic. Once I explicitly trusted the cluster's internal ranges — and,
separately, opened the one real port the game needed for players — everything that had been
mysteriously dead came alive at once.

The lesson I keep relearning: "no error" is not "no problem." A dropped packet and an absent
service look identical from above.

## The bot that updated the same thing twice

The update bot has a sensible default: it already understands the format my apps are declared
in, and pulls the version out of each one on its own. Trying to be thorough, I *also* told it
to look for versions in a second way, and added a helper on top of that.

The result was that it now found the same version in three places and treated them as three
separate things to update. The first update would apply and rewrite the file; the second and
third would then fail, because the text they were looking for had already been changed by the
first. Cryptic errors, a red pipeline, and a genuinely confusing hunt, all caused by me
declaring the same dependency more than once out of a misplaced sense of diligence.

The fix was to delete my cleverness and let the bot do the one thing it already did well.
More configuration was the problem, not the solution.

## The hosted service that did nothing

There is a convenient hosted version of the update bot — install an app, grant it access,
and it is supposed to start opening pull requests for you. I installed it and waited. Half an
hour later, nothing. No pull request, no error, no sign it had ever looked at the repository.

I stopped waiting and ran the bot myself instead, directly, and it produced the first update
proposal within minutes. The only visible difference is cosmetic: the pull request is
authored by me rather than by the service's robot account. I still do not know why the hosted
path stayed silent, and I have stopped caring — the self-run version is more transparent
anyway, because I can watch exactly what it does.

## The test suite that refused to lie

Underneath all of this I wrote a test suite, split deliberately into two halves: fast checks
that need nothing but the files themselves, and slower checks that require a live cluster. The
one design decision I am most pleased with is that the suite refuses to print a success banner
while the live half is skipped. You cannot get a green "all good" by quietly running only the
cheap tests. If it did not verify the real thing, it will not claim the real thing works.

That sounds obvious and it is the opposite of how most quick scripts behave — most of them
happily report success for the parts they bothered to run. I would rather a test suite tell
me "I only checked half of this" than let me walk away believing something I never confirmed.

## What I took from it

The loop delivered on its promise. I updated a running service by rolling a version back,
watched the bot propose putting it forward again, approved the pull request, and saw the
cluster reconcile itself with no hands on the server. The server's entire state now lives in
a repository I can read, review, and revert.

But the honest headline is the debugging, not the demo. A silent firewall that dropped
internal traffic, an update bot tripping over dependencies I had declared three times, a
hosted service that did nothing, and a test suite I specifically built to stop me fooling
myself. The automation is the easy part to show off. Making it trustworthy is the actual
work, and most of that work is teaching your tools to fail loudly instead of failing quietly.
