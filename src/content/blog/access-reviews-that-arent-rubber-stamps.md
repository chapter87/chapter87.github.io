---
title: 'You can''t review your way out of it'
description: 'Quarterly access reviews rubber-stamp almost everything and remove almost nothing, because the real problem is upstream in provisioning, not in the review.'
pubDate: 'Aug 17 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

Every quarter the tool sends an email. The manager opens it, sees 240 lines of their team's access, and clicks Approve All. They e-sign. Ninety seconds, start to finish. The audit evidence lands, the campaign closes at 99 percent, the dashboard goes green. Nobody removed anything. Nobody could have, because nobody read it.

I have run these campaigns and watched them close above 99 percent approval. That number does not mean the estate is clean. It means the review did nothing.

## The reviewer cannot answer the question you asked

Put yourself in the manager's seat. The tool shows you an entitlement called `APP_FIN_GL_JE_POST`, assigned to someone on your team, and asks: keep or revoke? You have no idea what it grants. It looks like it touches the general ledger. Revoke it and you might break someone's Monday. Keep it and nothing happens to you. So you keep it. Everyone keeps it.

This is the core rot. We hand line managers entitlements named for an ERP's internal schema, with no plain-language description, no statement of what business action they permit, and no penalty for guessing safe. The reviewer is not lazy. They have been asked a question nobody gave them the means to answer, and the cheapest correct-looking answer is always approve.

## Bulk approve is the design, not the abuse

A manager with 30 reports and 8 entitlements each faces 240 decisions. The interface puts Approve All at the top. Revoking one line demands a typed justification. Doing nothing is a single click; doing something is friction times 240. Decision fatigue does the rest: even a diligent reviewer who reads the first screen is stamping blind by the third.

You cannot train your way out of this. Blaming reviewers for approve-all is like blaming drivers for speeding on a road engineered for 90. Fix the road.

## What a real review looks like

Three things separate a review that means something from theater.

**Risk-based scoping.** Stop certifying everything every quarter. Most entitlements are low-risk, and running them all on the same cycle just buries the dangerous ones in noise. Put privileged access, separation-of-duties violations, and toxic combinations (create a vendor, then approve a payment to it) on a tight cycle. Let the rest go annually, or only on change. A shorter review gets read.

**Show last-used data.** This is the single most useful change I know of. Next to each entitlement, show when it was last exercised. "Last used: 214 days ago" turns an impossible question into an obvious one. Entra ID Access Reviews can recommend removal for accounts that have not signed in for 30 days. AWS IAM Access Analyzer surfaces unused roles and permissions, and Access Advisor gives you service-last-accessed timestamps. If your governance tool can consume activity data, feed it. A reviewer who can see that nobody has touched an entitlement in seven months will actually revoke it.

**Micro-certifications on change.** The quarterly calendar is the wrong trigger; the mover event is the right one. When someone changes team, certify the delta that day, in front of the manager who asked for the move, while the context is fresh. Now you are asking about five entitlements they can reason about, not 240 they cannot. That catches privilege accumulation the moment it happens, instead of up to 90 days later.

## Kill the entitlements nobody can explain

Run this test on your own estate. Pick a sensitive entitlement and ask three questions: who owns it, what it grants, why this holder has it. If nobody can answer all three, that is your finding. Not a risk to accept. A thing to delete.

The worst offenders are ownerless. A sync account stood up for a project three years ago, still holding write, still nobody's job to question. It passes every certification forever, because the tool routes it to a manager who does not recognize it and waves it through with the rest. Certification will never catch this on its own. You have to go looking for unexplained access and delete it.

## What actually fixes it, and what it costs

Here is the part most write-ups skip. You cannot review your way out of a lifecycle problem.

Certification is a detective control. It finds access that should not exist. But if your joiner-mover-leaver process grants without expiry, if birthright roles over-provision on day one, if movers add entitlements and never shed the old ones, then the quarterly review is you paying, every 90 days, to detect a bug you have chosen not to fix at the source. The cleanup never ends because the tap is still running.

The upstream fix is the uncomfortable one. Provision with expiry, so access removes itself. Engineer roles so a day-one grant matches the job, not the job title. Make mover events remove as aggressively as they add. Do that, and certification shrinks to the high-risk slice that genuinely needs a human eye.

That last step is the one auditors fight, because "we review less" sounds like weaker control. It is stronger. A program that certifies less, backed by provisioning that corrects itself, catches more than a campaign that certifies everything and reads nothing. The ninety-second approve-all was never a control. It was the receipt for its absence.
