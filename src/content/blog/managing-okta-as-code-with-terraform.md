---
title: 'Managing Okta as code with Terraform — and deciding what NOT to automate'
description: 'I already had a working Okta tenant, built by hand. The interesting part of putting it under Terraform was not the code — it was drawing the line between what infrastructure-as-code should own and what it must never touch.'
pubDate: 'Aug 17 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

Most Terraform tutorials start from an empty account and build up. Real life
almost never looks like that. By the time someone decides to manage an identity
provider as code, the tenant already exists — built by hand, wired into a
directory, serving live single sign-on. My homelab was exactly that situation:
an Okta tenant, already syncing users from on-prem Active Directory, already
doing SAML SSO into an app. The job wasn't to *build* it in Terraform. It was to
**adopt** it without breaking anything.

## Adopt, don't recreate

The dangerous way to bring an existing tenant under Terraform is to write
resource blocks and `apply`, because Terraform's state starts empty — it thinks
nothing exists, so it tries to *create* things that are already there. On an
identity provider that's live, that's how you end up with duplicate groups or a
destroyed SSO app.

The safe way is declarative `import` blocks (Terraform 1.5+). You describe the
resources, point each `import` at the ID of the thing that already exists, and
your acceptance test before touching anything is a single line in the plan:

> **`0 to change, 0 to destroy`**

If the plan wants to change or destroy something, your code doesn't yet match
reality — fix the code, not the tenant. I tuned my group definitions until the
descriptions matched the live ones exactly, and only then did the plan come back
as "import these, add one new thing, change nothing, destroy nothing." That
applied cleanly, and a re-plan showed zero drift. That zero-drift re-plan is the
whole game: it proves the code is now a faithful mirror of the live system.

## The real design decision: who owns what?

Here's the part worth putting on a CV, because it's a judgement call, not a
syntax question. Once the same users and groups exist in Active Directory, in
the identity provider, *and* in Terraform, you have to answer one question or
you'll fight drift forever: **for any given thing, which system is the source of
truth?**

I split it in one sentence:

> **Active Directory decides *who is in* a group. Terraform decides *what a group can reach.***

So Terraform owns the group objects and the application entitlements — the
access grants. It deliberately does **not** own:

- **Group membership.** The directory is the source of truth for that, synced in
  by the agent. If Terraform tried to manage membership too, it would fight the
  directory import on every sync. Code must not argue with the directory.
- **The directory-mastered group copies.** Those are owned by the sync, not by me.
- **The live SSO application objects.** This one is a trap. Importing a working
  SAML app means reproducing every attribute statement and signing setting
  *exactly*, and any mismatch turns into a destructive plan against live SSO. So
  I reference those apps as read-only data sources instead of managing them.
  Terraform can *point at* the SSO app to grant a group access to it, without
  *owning* the fragile app itself.

That last decision — treating the live SSO app as a data source, not a resource
— is the difference between infrastructure-as-code that helps and
infrastructure-as-code that takes down your logins during a routine change.

## Why this is actually better than clicking

The payoff shows up the first time you grant access. Instead of clicking through
an admin console, a new entitlement is a one-line change:

```
saml_app_groups = ["G-Engineering"]        →  ["G-Engineering", "G-IT"]
```

`terraform plan` comes back as **`1 to add`**. That diff is reviewable in a pull
request. Someone can see, in version control, exactly who granted which group
access to which application and when — which is precisely the kind of audit
trail access reviews are supposed to produce, except here it's free, because
it's just git history. I proved the full lifecycle by creating a brand-new
"Contractors" group purely through Terraform, and confirmed afterwards via the
API that the existing users' logins were undisturbed — the change was purely
additive.

## Secrets discipline and a CI gate

Two things I was strict about, because an identity provider is exactly where you
don't want a leak:

- **No token ever lives in a `.tf` file.** The provider reads credentials from
  the environment, mapped in from a local secrets file that's outside the repo.
- **State and plan files are git-ignored.** Both `*.tfstate` and the saved
  `tfplan` contain secrets in cleartext — and the plan file is easy to forget; it
  nearly got swept into my first `git add`. Watch for it.

For CI, the pull-request gate runs with **zero credentials**: format check,
`init -backend=false`, and `validate`. It catches broken code without ever
needing access to the live tenant. A credentialed `plan` is a separate,
manually-triggered job — because while state is still local, a CI plan would
start from empty state and propose re-importing the world, which is noise, not a
gate. Knowing *why* not to run plan-on-every-PR yet is as much a part of the
design as the pipeline itself.

## What I took away

The Terraform itself was the easy 20%. The 80% that mattered was boundary-drawing:
deciding that the directory owns membership, code owns entitlements, and the
fragile live SSO apps get referenced but never owned. That's the same instinct
you need in a real identity team — automation is powerful right up until it
starts fighting the system of record, and knowing where to stop is the skill.
