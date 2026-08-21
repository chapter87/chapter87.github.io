---
title: 'Every exclusion is a door'
description: 'A working Entra Conditional Access baseline, and the honest reason it fails both ways: the exclusions you cut so real people can log in are the exact holes attackers walk through.'
pubDate: 'Aug 13 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

Six months after you switch on the MFA policy, you're reading a finance user's sign-in log: a successful auth, no prompt. The client app reads "IMAP4." The Conditional Access tab says the require-MFA policy did not apply. The source IP is a hosting provider in a country where you employ nobody. The password was correct, and on paper that account was protected.

That's Conditional Access in one screenshot. The policies work. They only apply where you scoped them, and you scoped out more than you remember.

## A baseline that holds

Here is what I deploy on every Entra tenant, in report-only mode first.

- Block legacy authentication for all users and all apps. It's the highest-value policy you'll set. Microsoft's own figures: over 99% of password-spray and 97% of credential-stuffing attacks ride legacy protocols, because POP, IMAP, SMTP AUTH and old MAPI can't render an MFA challenge. They return a yes/no on the password and nothing else. A require-MFA policy won't save you; the protocol never reaches the grant, so you have to block it by name.
- Require MFA for all users, all cloud apps.
- Require a compliant or hybrid-joined device for anything sensitive.
- Require MFA to register or join a device.
- Block unsupported device platforms, and drop persistent browser sessions on unmanaged ones.

Report-only is not a nicety. The first time I moved "require compliant device" straight to enforced, the logs showed it would have blackholed the field sales team on their personal iPads before lunch. Report-only writes the verdict, "would have been blocked," to the log without blocking anyone. You read a week of it, then you enforce.

## Failure one: so strict it locks people out

Then reality pushes back. An exec is abroad and the push never arrives. A service account running a nightly export can't do interactive auth. So you cut an exclusion. Each is reasonable on its own, and each is a hole.

Take the break-glass account. Microsoft tells you to create two cloud-only emergency accounts and exclude them from every CA policy, so a bad policy can never lock you out. That advice is correct. But an account excluded from all Conditional Access is exactly what an attacker hunts for: a valid credential with no MFA, no device check, no location check. If its password sits in a vault half the company can open, or it's named breakglass@ and hasn't rotated since the tenant was built, you did not build a safety net. You built a front door and labelled it.

You don't fix this by deleting the exclusion. You make the account expensive to use: a FIDO2 key instead of a password bypass, a split-knowledge secret no single person holds, and a Sentinel alert that fires on any sign-in to it at all. A break-glass login you did not perform is a phone-call-at-2am event, not a log line you find next quarter.

## Failure two: the gaps nobody notices

The second failure mode is quiet. Nothing breaks, nobody complains, and the hole stays open for months.

Legacy auth is the first gap, the IMAP success from the top. Skip the explicit block and it stays open however strict your MFA policy looks.

Trusted locations are the second. Someone adds the corporate VPN egress IP as a named "trusted" location and ticks "skip MFA from trusted locations." MFA is now waived for everyone on that VPN: the attacker who phished one VPN credential, and every contractor on a split tunnel you forgot about. A trusted location should lower a risk score. It should never waive a control.

Unmanaged devices are the third. "Require compliant device" only bites where it's assigned. Miss one app, one guest path, one temporary carve-out for the mobile fleet that outlived its ticket, and there's a way in from hardware you've never seen.

## The attack that survives MFA done right

Here's the part most baselines skip. With MFA enforced and legacy auth blocked, adversary-in-the-middle phishing (Evilginx, EvilProxy) still works. The kit proxies the real login page, the user passes MFA, and it lifts the session token Entra hands back. Replay that token and the sign-in log reads "MFA requirement satisfied by claim in the token." A Conditional Access block throws AADSTS53003; the replayed token just succeeds from the attacker's machine.

More factors don't help: the user completes every one for real, through the proxy. What breaks the attack is binding the credential or the token to hardware the attacker doesn't have. Phishing-resistant methods (FIDO2, certificate-based auth) won't authenticate to a proxied origin, so the login never completes. Token protection binds the token to the device, so a lifted token won't replay from a stranger's laptop. A compliant-device requirement raises the same wall: the proxy can't present the device certificate Entra demands. Stacking a fourth factor onto a phishable method changes none of this.

## What actually fixes it, and what it costs

Block legacy auth tenant-wide. Remove every "skip MFA from trusted location." Make device compliance, not the one-time code, the real gate to sensitive apps. Roll phishing-resistant sign-in to privileged users first. Move break-glass onto FIDO2 and alert on its every use. Re-read your exclusion list monthly and treat each entry as an open port, because that's what it is.

The cost is real, and you should say it out loud, not discover it mid-rollout. Phishing-resistant MFA means buying and shipping hardware keys. Device compliance means every device in Intune, which reopens the BYOD fight and loads the help desk. Killing the trusted-IP skip means the exec who used to walk straight in now gets prompted, and emails you about it by lunchtime. Shorter sign-in frequency means more reauthentication for everyone.

None of that is optional if the baseline is meant to hold. Conditional Access is not a wall you build once. It's the running list of every exception you've granted, and the only real question is whether you can still name all of them.
