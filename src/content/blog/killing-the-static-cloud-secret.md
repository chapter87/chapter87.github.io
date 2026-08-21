---
title: 'No secret to steal is only half the fix'
description: 'A long-lived cloud key in a CI config is the liability worth killing first, but OIDC federation and SPIFFE trade it for trust-policy, blast-radius, and availability problems most write-ups skip.'
pubDate: 'Aug 8 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

## The key that never expired

A secret scanner flagged a config file in a repo nobody had touched in two years. Inside was an AWS access key ID: `AKIA` and sixteen more characters, committed in 2023. I pasted it into `aws sts get-caller-identity`. It answered. The key was live, tied to an IAM user whose policy could list the production data bucket, and it had been cloned onto every laptop and CI cache that ever pulled the repo.

That is NHI7, long-lived secrets, the seventh item on the OWASP Non-Human Identity Top 10 and the one I find most often. Not because teams are careless. A static key is the path of least resistance: paste it into the CI settings once and the pipeline works forever. Forever is the problem.

## What a standing key actually costs

A long-lived credential has no natural end. It outlives the employee who made it, the project that needed it, and the laptop it leaked onto. The same string works from your build runner and from an attacker's VPS just the same; the credential has no audience. Rotation breaks things, so nobody rotates. It sits valid for years, and your only detection is hoping a scanner finds the copy first.

You cannot secure a string designed to be copied and never change. You have to stop issuing it.

## Federation: proving who you are, not what you hold

OIDC federation mints a token on demand instead of storing a key. GitHub Actions is the clearest case. Grant the workflow `id-token: write` and the runner can request a signed JWT from `token.actions.githubusercontent.com`. That JWT describes the run: a `sub` like `repo:corp-org/payments:ref:refs/heads/main`, an `aud`, the repository, the workflow ref.

On AWS you register that issuer as an IAM OIDC provider and write a role trust policy that assumes the role only when the claims match. The `configure-aws-credentials` action calls `AssumeRoleWithWebIdentity` with the JWT and gets back temporary STS credentials: `ASIA` prefix, good for an hour, then gone. Nothing is stored. No `AKIA` in the settings, nothing in git, nothing on a laptop. The scanner has nothing to find because there is no secret.

That is the real win. It is not the whole story.

## Where federation quietly fails

The stored secret becomes a trust policy, and a trust policy is just as misconfigurable. Two failures I see constantly.

The wildcard `sub`. Write `StringLike` with `repo:corp-org/payments:*` and you no longer trust "main on this repo". You trust every branch, every pull request, every environment. Anyone who can push a branch or land a workflow assumes the role. Pin the subject to a specific ref or environment, and require `aud` equal to `sts.amazonaws.com`. Omit the `aud` condition and it is worse.

The over-privileged role. Federation says nothing about what the role can do. Short-lived credentials to an admin role are admin for the hour you hold them. "No secret" is not "least privilege"; that is NHI5, a separate fight.

And the token is a bearer token while it lives. A poisoned dependency or a compromised action can read it from runner memory and use it inside the window. The `tj-actions/changed-files` compromise in March 2025 did exactly that, dumping runner secrets into build logs. An hour is plenty for an automated attacker. Short-lived shrinks the window; it does not close it.

## SPIFFE: make the workload prove what it is

For service-to-service auth inside your own infrastructure, SPIFFE and its implementation SPIRE go further. Each workload gets a SPIFFE ID, `spiffe://corp.example.com/ns/payments/sa/api`, delivered as an SVID: an X.509 certificate with that ID in the URI SAN, or a short-lived JWT. SPIRE's default X.509 TTL is one hour, and it rotates at half-life on its own.

The Workload API is a Unix domain socket the workload connects to holding no credential at all. The agent inspects the calling process (UID, Kubernetes service account, container image) while node attestation proves the machine it runs on, via an AWS instance identity document or a projected Kubernetes token. Identity comes from what the workload verifiably is, not from a string it carries.

## Why SPIFFE is not a free lunch

Attestation still has to root in something. On a cloud instance the identity document is free; on bare metal with no TPM you fall back to a join token, which is a secret. The SPIFFE community wrote a book about this, *Solving the Bottom Turtle*: you only move the first turtle somewhere defensible, you never delete it.

Then there is the operational bill. A production SPIRE deployment is a highly available server with a real datastore, an agent on every node, an upstream CA, and a controller managing registration entries. Identity becomes a tier-0 dependency: when SPIRE is down, workloads cannot rotate SVIDs and fail closed. You have swapped a leaked-secret risk for an availability risk. It also covers less than you would like: a SaaS vendor that only accepts a static API key cannot be federated into, and most databases do not speak OIDC.

## What actually fixes it — and what it costs

Kill the `AKIA` key this quarter. OIDC federation for CI-to-cloud is a day of work: register the provider, write one trust policy per environment with a pinned `sub` and an explicit `aud`, attach a least-privilege role, add a session policy to cap it. That single change removes the largest, dumbest liability you own, and it is cheap.

SPIFFE is the right long-term shape for east-west service identity, provided you budget it as the platform it is: HA server, entry automation, break-glass, on-call. If you cannot staff that, Vault dynamic secrets on short leases are an honest bridge. No permanent key, far less to run.

The end state is not zero secrets. Nobody reaches zero. It is no long-lived secrets: every credential that remains is short-lived, audience-bound, attested, and least-privileged. The standing key in the config is the one you can actually kill. Start there.
