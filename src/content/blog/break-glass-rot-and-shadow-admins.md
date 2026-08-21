---
title: 'PAM protects the admins you know about'
description: 'Two privileged-access failures never appear on any admin roster: the emergency break-glass account nobody has tested, and the shadow admins who hold Domain Admin power through an ACL without ever joining the group.'
pubDate: 'Aug 14 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

The domain had twelve Domain Admins. I know because I read the group membership myself. Then I pointed BloodHound at the same domain and asked a different question: not who is a Domain Admin, but who can become one. The answer came back in the low hundreds. A help desk team of about thirty could reset the password on a service account. That service account could write to a group. The group was nested, two hops down, inside Domain Admins. Not one of those thirty people was an admin on paper. Every one of them was an admin in practice.

That gap is where two of the quietest failures in privileged access management live. Neither shows up when you count admins the usual way, because the usual way counts group membership, and privilege does not come only from membership.

## The account nobody can use

Break-glass first, because it's the control people are proud of. You create an emergency admin (RID 500 on-prem, or a dedicated cloud-only account in Entra ID) and exclude it from Conditional Access and MFA on purpose, so a broken federation or a bad policy push can't lock every human out of the tenant at once. Sensible. Microsoft tells you to keep two.

Then you seal the password in an envelope and forget it exists.

Two years later, three things may be true and you don't know which. The password might have aged out, if the account follows the domain policy and nobody set `DONT_EXPIRE_PASSWORD` (userAccountControl bit 0x10000). Someone might have "cleaned up stale accounts" and disabled it. Or it works fine and you still can't tell, because you excluded it from the exact policies that generate your sign-in telemetry. You built an account to bypass your controls, then stopped watching it. `pwdLastSet` reads two years old and no one has logged in with it since.

## The admins who were never in the group

Shadow admins are the other half. These are accounts with admin-equivalent power granted directly in the directory, through an ACL, never through a privileged group.

The rights that do it are specific. `ForceChangePassword` (extended right `00299570-246d-11d0-a768-00aa006e0529`) lets you reset a Tier 0 account's password. `GenericAll` and `GenericWrite` let you rewrite its attributes: add an SPN and Kerberoast it, or point `scriptPath` at your payload. `WriteDacl` lets you grant yourself anything else you're missing. The cleanest one is the DCSync pair: `DS-Replication-Get-Changes` and `-Get-Changes-All` (`1131f6aa-…` and `1131f6ad-…`). Hold both and you can replicate every password hash in the domain using your own credentials. No admin logon, no MFA anywhere in that transaction, because you never authenticate as an admin. You ask a Domain Controller for the hashes and it hands them over.

Built-in groups hide the same power in plain sight. Account Operators can edit most non-protected users. Backup Operators hold `SeBackupPrivilege`, which reads `NTDS.dit` straight off the DC. `adminCount=1` is a decent lead for finding protected objects, but treat it as a lead, not an inventory. It goes stale, and orphaned values linger long after an account stops being privileged.

## Why your PAM vault misses both

A privileged access vault rotates and records the accounts someone onboarded into it. That's the whole model, and it's why both failures slip past.

Break-glass is deliberately not onboarded: if the vault is the thing that's down, you still need a way in, so access can't depend on the vault. Shadow admins were never recognized as privileged, so nobody onboarded them either. The vault's coverage is a list of accounts a human decided to call privileged. Both of these are off that list.

"But all our admins have MFA" doesn't save you. The break-glass account is excluded by design. The shadow admin authenticates as an ordinary user, then exercises a directory right. MFA on the target account never fires, because nobody ever logs in as the target.

## Finding them for real

For break-glass, stop asking "does the password exist" and start asking "did a human authenticate with it this quarter, and did the alert fire." Check `pwdLastSet` and the `DONT_EXPIRE_PASSWORD` flag. Then actually log in, and record that you did. Alert on any use: in Entra, a Sentinel rule filtered to the account's object ID; on-prem, Event ID 4624 and 4672 for that SID, plus 4768 for the TGT request. The point of the test is the alert, not the login.

For shadow admins, run BloodHound as a standing control. Ingest weekly and query the shortest path to Tier 0. ACLight2 and PingCastle cover ground BloodHound doesn't. Turn on the object-access auditing that produces Event 5136 when a DACL changes, and 4662 with those replication GUIDs when someone attempts DCSync. ACLs drift every day the help desk delegates something, so a scan from last quarter is already wrong.

## What actually fixes it, and what it costs

Both fixes are the same unglamorous shape: someone owns the list and keeps proving it's still right.

Break-glass: at least two accounts, a long random password, `DONT_EXPIRE_PASSWORD` accepted as a deliberate risk, a quarterly login test that gets logged, and an alert on any sign-in that you have watched fire with your own eyes. A silent alert is worse than none, because it tells you you're covered when you aren't. Cost: discipline and a pager.

Shadow admins: the count of principals with a path to Tier 0 becomes a number on a dashboard that has to trend down. You adopt a tiered admin model and delete the convenience delegations the service desk keeps adding back. Cost: this is a program, not a scan, and most of the spend is arguing with people whose jobs you're making slower.

The tool finds them once. Only a person keeps them found.
