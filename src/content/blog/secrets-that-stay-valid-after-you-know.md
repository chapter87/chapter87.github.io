---
title: 'You can''t un-leak a secret'
description: 'Scanners flag a leaked credential in seconds, yet most stay valid for days: deleting the commit fixes nothing, and rotating a live key is risky, unowned work nobody volunteers for.'
pubDate: 'Aug 9 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

The alert fired forty seconds after the push. A cloud access key, the `AKIA…` kind, committed to a public repo by an engineer who'd pasted it into a test script and forgotten the `.env` was tracked. Our scanner caught it. GitHub's push protection might have stopped it at the push too, if it were on and the string matched a pattern it knew. Everyone in the channel felt good for about a minute.

The key was still valid two weeks later.

That gap, between "we found it" and "it can't be used any more," is the whole problem, and it's the part nobody plans for. Detection is solved. gitleaks, trufflehog, GitHub secret scanning, the commercial engines: they work, they're fast, and they fire on the right things. GitGuardian counted nearly 29 million new hardcoded secrets on public GitHub in 2025, up 34% year over year, the biggest single-year jump they've recorded. We're very good at finding them. We're terrible at killing them. Their other number is the one that stings: more than 90% of leaked secrets are still valid five days after the author gets the alert.

## Deleting the commit does nothing

The first instinct is the wrong one. Someone runs `git filter-repo` or reaches for BFG, force-pushes a clean history, and closes the ticket. The secret is gone from the repo. The secret is not gone.

A public commit is scraped within seconds. Archive services, dataset builders, and attackers running the same scanners you run all pull public events off the firehose in near real time. By the time your alert fires, the string has been copied to machines you'll never see. It sits in forks, in clones, in a stranger's reflog, in a hundred caches. Rewriting history stops the next accidental re-leak. It does not un-share what was shared. The only thing that remediates a leaked credential is invalidating the credential. Rotate it or revoke it. There's no other move.

So the fix is always rotation. And rotation is where it all falls apart.

## Why nobody rotates the key

Take that `AKIA…` key. To rotate it safely I need three facts: who owns it, what uses it, and what breaks when it dies. Usually I have none of them.

The key was created two years ago by someone who has since left. It has no tags. CloudTrail shows it still making `s3:GetObject` calls a few hundred times an hour, so something depends on it, but the log doesn't say what. Disable it and I might take down a nightly export nobody remembers building. Leave it and I'm sitting on a live, public credential.

That's the real reason behind that number. It isn't laziness, it's incentives. The engineer who rotates a production credential at 2pm and causes an outage owns that outage. The engineer who lets a flagged key sit owns nothing; the risk is diffuse, deferred, somebody else's problem. Every rational actor leaves the ticket alone. Multiply that by one shared secret baked into twelve services, each needing its own coordinated deploy, and the credential outlives the incident.

## These are all non-human identities

Look at what actually leaks: cloud access keys, API tokens, OAuth client secrets, database connection strings, SSH keys, CI personal access tokens. None of them is a person. They're non-human identities, and in a normal cloud estate they outnumber human accounts by something like fifty to one.

Humans have a lifecycle. You're provisioned on your first day and deprovisioned on your last; there's a joiner-mover-leaver process, an owner, a periodic review. A service account has none of that. An engineer creates it under deadline, it never expires, it's never reviewed, and its creator leaves the company. The secret is the identity: possession equals access, no second factor behind it. You can't put MFA on a static string in a config file. This is the population of identities nobody counts, and it's exactly the population that ends up in a public commit.

## What actually fixes it, and what it costs

The durable fix is to stop having long-lived secrets to leak in the first place.

Short-lived beats vaulted. The strongest pattern is no stored key at all. OIDC federation lets a CI job assume a cloud role and get back an STS token that lives fifteen minutes. There's nothing persistent to commit, and a token that expired an hour ago is a non-event when it leaks. Where you can't federate, use a vault that issues dynamic secrets with a TTL and revokes them on its own.

Own it and rotate on a schedule, not on incident. Every secret carries an owner tag and a stated purpose, and rotation runs automatically inside the secrets manager whether or not anyone is watching. A rotation you have to remember to run is a rotation that won't happen.

Here's the honest bill. OIDC needs your CI, your cloud trust policies, and your app runtime to all support it; legacy apps that read a static key from an environment variable have to be rewritten. Plenty of SaaS vendors still issue one permanent API key with no expiry and no rotation endpoint, and you can't vault your way out of a vendor that only hands you a forever-key. Dynamic secrets break connection pools that grab a credential at startup and hold it for hours. Vault itself becomes critical infrastructure you operate and defend, a fresh single point of failure with its own unseal keys.

You'll never reach zero static secrets. There's always a bootstrap credential: the token that lets a workload authenticate to the vault in the first place. The win isn't elimination, it's collapsing thousands of scattered, immortal keys into one short-lived, closely watched root of trust. Fewer secrets, shorter lives, real owners. Detection was never the hard part.
