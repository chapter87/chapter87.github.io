---
title: 'You have more roles than people'
description: 'Why RBAC explodes into thousands of roles, where ABAC quietly stops being auditable, and the boundary between them that survives an access review.'
pubDate: 'Aug 16 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

Pull the role catalog from any identity system that's run five years. Count the rows, then count the employees. At one place I worked the first number was past 4,000 and the second was around 3,000. More roles than people. Nobody designed that. It happened to them.

The role names give it away. `finance-ap-approver-emea-contractor-readonly`. `sales-crm-edit-northeast-manager-temp`. Each one is a real access decision someone made under deadline and froze into a role, because a role was the only tool the system offered. Nobody deletes them. Nobody can prove they're unused, and half are named after a team that reorganised two years ago.

## RBAC is clean, and that's why it breaks

Role-based access control earns its reputation. People go in roles, roles carry permissions, and the audit question is trivial: who is in this role? A recertification campaign becomes a list of members and a manager clicking approve. SOX access reviews, ISO sign-offs, the whole compliance machine assumes that shape. The NIST RBAC model, standardised as INCITS 359, is enumerable by design. You can always answer "who can access what" by reading the assignments.

Then reality shows up, and reality is contextual. RBAC has one lever for context: mint another role. Consider an approver who should sign invoices only under a threshold, only for their own cost centre, only from a managed laptop. RBAC cannot express a word of that. So you split the role. Region splits it again. Seniority, environment, and employment type each multiply the count. That is role explosion, and it is not a discipline problem. The model is doing the only thing it can.

## ABAC moves the context into policy

Attribute-based access control (NIST SP 800-162) decides at request time from attributes: the subject's department and device posture, the resource's classification and owner, the action, the environment. Instead of a role per combination you write one policy. "Approvers may sign invoices under £10,000 for their own cost centre from a managed device." The combinations collapse into conditions.

When access depends on runtime state, RBAC has nowhere to put it. Ownership relationships, break-glass windows, step-up on a risky action, classification rules that span every app. None of that pre-computes into roles without the explosion you were escaping. I have watched a 300-role tangle shrink to a dozen roles and a handful of policies. It is a real win.

## Where the audit falls apart

Here is the part the pitch skips. RBAC answers "what can Alice reach?" by reading a list. ABAC answers it by evaluating every policy against every resource she might touch, using whatever attribute values are true at that moment. The forward question, "can Alice do this right now," is easy. The reverse question, "enumerate everyone who can reach the finance app for the auditor," has no cheap answer. You are asking the policy engine to run backwards.

The failure is not that ABAC is insecure. It is that ABAC is opaque. Policies interact. In XACML you choose a combining algorithm, deny-overrides or permit-overrides, and a rule you forgot shadows one you wrote last week. Attribute data becomes load-bearing. If the HR feed still lists someone in the cost centre they left, they keep access, and nothing in the policy looks wrong. "Nobody can tell who can access what" is the usual complaint, and it is accurate. I have sat in a review where the honest answer to the control owner was "I'll have to run a job to find out." That answer fails the control.

## The line that holds

Do not pick a side. Use RBAC for the coarse grant and policy for the fine condition. A role, ideally a birthright derived from HR attributes, gets you into the finance app at all. That layer stays small, stable, and countable, so your access reviews still function. Everything conditional drops into versioned policy: the threshold, the cost centre, the device check, the delegation window. That is what PBAC means when it's not just ABAC with a new sticker. The role becomes one attribute the policy reads.

Tooling finally supports it. Open Policy Agent with Rego, Cedar behind Amazon Verified Permissions, OpenFGA for relationship-heavy models: each keeps policy as code you can review, test, and roll back. The discipline is a boundary. Coarse and few above the line, conditional and versioned below it. Push a cost-centre check up into a role name, or an app entitlement down into a swarm of policies, and you rebuild the mess in the other layer.

## What it actually costs

The fix is real, and it is not free. Three bills come due.

You now own an attribute pipeline. Policy is only as correct as the attributes under it, so someone has to keep the HR feed, the device signal, and the cost-centre map fresh and owned. Stale attributes are the new orphaned roles, and they hide better.

There is a policy lifecycle to run. Policy as code means tests, review, and a merge gate, or you've just moved unaccountable changes out of a role catalog and into a Git repo nobody guards.

And the reverse query is yours to build. ABAC won't tell you "who can access what" for free, so you stand up effective-access reporting and turn on decision logs, or your next audit is the meeting where you promise to run a job.

RBAC is countable and rigid. ABAC is flexible and opaque. The middle keeps the countability where auditors look and spends the flexibility where access is contextual, and the price of the middle is the tooling that makes a policy as reviewable as a role list used to be. Anyone who tells you ABAC is simply cleaner than RBAC has never run a recertification against a policy engine.
