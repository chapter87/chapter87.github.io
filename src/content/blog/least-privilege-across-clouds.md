---
title: 'The policy is not the permission'
description: 'An IAM policy tells you what was written, not what a principal can actually reach, and across three clouds nobody has computed the real number.'
pubDate: 'Aug 11 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

Ask a simple question during an access review: what can this one engineer actually do in production? Not what their job needs. What the credentials permit, right now, if they decided to use them. In a single tidy AWS account you might answer it. Across three clouds, with federation and a few years of accreted grants, nobody in the room knows. I've watched a team of good engineers stare at that question for twenty minutes and produce three different wrong answers.

The reason is the same everywhere. A policy is a document. A permission is a fact about the running system. They are not the same thing, and the distance between them is where least privilege quietly dies.

## The numbers make hand-authoring a fiction

AWS has more than 400 services and well past ten thousand distinct actions. GCP exposes several thousand granular permissions. Azure ships hundreds of built-in roles on top of its own action list. Nobody holds that in their head, so when an engineer needs "enough to deploy," they don't hand-write a tight policy. They attach `PowerUserAccess` and move on.

PowerUser feels safe because it is not `AdministratorAccess`. Read the actual policy: it is `Allow` on everything with a `NotAction` carve-out for `iam:*`, `organizations:*`, and `account:*`. You cannot create users. You can do everything else, including launching compute and calling `sts:AssumeRole`. PowerUser is admin with two extra steps.

## The gap is the whole game

Reading one policy tells you almost nothing, because effective access is a computation over a chain.

On AWS, effective permission is a computation across five inputs: the identity policy, any resource policy, any SCP, the permission boundary, and the session policy. Some grant, some only filter, and an explicit deny anywhere beats them all. A wildcard `Allow` means nothing if an SCP denies it. A narrow `Allow` can be widened by a bucket policy you never opened. Context counts: an action permitted only when `aws:MultiFactorAuthPresent` is true is denied to a session without MFA.

That is the static picture. The dynamic one is worse, and it is what policy-reading always misses: transitive access. `iam:PassRole` on `*` reads like nothing and is total ownership, because you can hand an admin role to an instance you control. Rhino Security Labs documented 21 AWS privilege-escalation methods, most starting from one permission that reads as harmless: `CreatePolicyVersion`, `AttachUserPolicy`, `PassRole` paired with `RunInstances`. GCP has its own set: `iam.serviceAccounts.actAs`, `getAccessToken`, or `setIamPolicy` on a resource lets a limited principal become whatever it can impersonate. Azure's version is a directory role. `Application Administrator` can add a credential to an existing service principal, and if that principal holds the Graph permission `RoleManagement.ReadWrite.Directory`, you have just made yourself Global Administrator.

None of that appears in the principal's own policy. You see it only if you build the graph.

## Three clouds, three mental models, one person

Now multiply by three. The same human, usually a group from Okta or Entra ID, lands in AWS as an SSO permission set scoped to one account, in Azure as Contributor on a subscription, in GCP as Editor at the organization node. Different teams set those up at different times, each with its own idea of reasonable. The blast radius for one identity runs from one account to an entire organisation, and nobody planned that spread.

You cannot diff Contributor against an AWS role, because the models share no primitives. Azure grants are additive and inherit down the management-group tree, so a forgotten Owner high in the tree makes every careful restriction beneath it moot. GCP is additive too, with deny policies bolted on late. AWS is deny-by-default, SCPs and boundaries subtracting from a starting point of nothing. There is no shared vocabulary in which "these three are equivalent" is a sentence you can even write. The unified picture does not exist by default, and the CIEM dashboards that sell you one are approximating it.

## What the analysis actually shows

Effective-permissions analysis is the opposite of reading policies. You take a principal and compute every action and resource it can reach, resolving the full chain and walking every escalation edge: assume-role, impersonate, add-credential, set-policy. This is BloodHound's insight pointed at cloud. PMapper builds it for AWS as a literal graph of who can become whom. AzureHound does it for Entra. Google's own recommender nibbles at the usage side.

Run it once and the vanity metric falls over. Microsoft's Permissions Management reports that identities use around 1% of the permissions they hold. True, and a trap. Drive that Permissions Creep Index to zero by stripping standing permissions and you still leave every `PassRole` and `actAs` edge intact. The score looks clean. The blast radius has not moved. A creep number without the transitive graph is measuring the wrong thing.

## What actually fixes it, and what it costs

Three things, in order.

Compute effective access across all three clouds, on a schedule, with the escalation graph included. Not policy text, not a single-cloud score. This is the only view that answers the review question honestly. It costs real engineering, because normalising three incompatible models is lossy, and the lossy parts are the escalation edges. So you keep the raw graph per cloud and stop pretending one number summarises it.

Right-size from observed usage: Access Analyzer generating policy from CloudTrail, Entra Permissions Management, Google's recommender. The honest limit is that a 90-day window cannot tell "unused because unneeded" from "unused because the quarterly close has not run yet." Strip on usage alone and you find out which it was during the DR test. Keep a human in that loop.

Make the standing grant small and the escalation path expire: permission sets with short sessions, Azure PIM eligible instead of active, short-lived service-account tokens. That breaks anything built around standing access, which is most of your automation, so it lands last and slowly.

None of this is a product you buy and switch on. The fix is a habit. Stop reading policies and start computing what they actually grant.
