---
title: 'Your MFA worked. They got in anyway.'
description: 'Real-time phishing proxies relay SMS, TOTP, and push approvals straight through, so the second factor stops nothing, and fixing it with origin-bound authenticators just makes account recovery the new weakest link.'
pubDate: 'Aug 12 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

A user on our finance team got a mail about a shared document. The link looked right. The login page looked right, because it was the real login page, proxied through a server the attacker controlled. She typed her password. Her authenticator app buzzed. She read off the six digits and typed them in. The document opened. By the time she closed the tab, someone else held her session cookie and was reading her mail.

Her MFA worked exactly as designed. That is the problem.

We spent a decade telling people a second factor stops account takeover. Against a specific, growing class of attack it does not, and the reason is baked into how the factor works.

## The factors that relay

SMS codes, TOTP (the six digits from an app), and push approvals share one property: what proves you are you can be carried across a wire by someone else. A code is a number. A push approval is a yes. Neither one knows which site it is being used to log into.

Real-time phishing turns that into a clean bypass. Tools like Evilginx2 and EvilProxy do not clone a page and hope. They run as a reverse proxy in front of the genuine service. You send your password, they relay it upstream. The real site issues its OTP challenge, they relay it down to you. You enter the code, they relay it back. The site is satisfied, mints a session cookie, and the proxy keeps it. Every factor was presented correctly. The attacker broke none of them. He sat in the middle and passed notes.

SMS carries a second flaw before phishing enters at all. SIM swap: talk a carrier rep into moving the number to a new SIM, and every code and every "text us to reset" now lands on the attacker's handset. NIST marked SMS as restricted in SP 800-63B years ago. It is still everywhere.

## Push fatigue is a design flaw, not a user mistake

Push-approve was meant to kill the typing. It opened its own hole. An attacker who already has your password, from a breach dump or an earlier phish, can fire login attempts all day, and each one buzzes your phone. Approve one, by reflex or just to stop the noise, and he is in. This is MFA fatigue, and it is how Uber landed on the front page in 2022.

Number matching helps. Microsoft made it the Authenticator default in 2023: instead of a bare Approve button, you type a number shown on the login screen. The door still does not close, because a competent social engineer will phone you, say he is from IT, and read you the number to type. The factor is still trusting a human under pressure to make the right call.

## Why a passkey does not relay

When you register a FIDO2 credential (a passkey, a hardware security key, Windows Hello for Business) the authenticator generates a key pair bound to the site's origin, its RP ID. The private key stays in hardware: a TPM, a secure enclave, the chip inside the key. At login, the browser hands the authenticator the origin it is talking to. The authenticator signs the server's challenge, and the origin travels inside the signed clientDataJSON.

Now run the Evilginx attack again. The victim is on corp-login.example, not corp.example.com, and the RP ID will not match: the browser never even offers the corp.example.com passkey to the phishing page. There is nothing to sign, no code to relay, no shared secret crossing the user; nothing the proxy can forward means anything off its own origin. The attack does not get harder. It loses the thing it was stealing.

Certificate-based auth and mTLS give you the same non-relayable property: the private key lives in hardware and proves possession without ever passing through the login form. CISA's phishing-resistant MFA guidance ranks these at the top.

## The part the vendor slides skip

You cannot move everyone to passkeys on a Friday. This is where I have watched rollouts stall.

Some of it is plumbing. Shared workstations and call-center floors where a credential bound to one person's device does not fit. Contractors on laptops you do not manage. The procurement app that still only speaks TOTP. You front what you can with the IdP and conditional access, and you write down what you cannot.

The real trap is account recovery. The day your login factor becomes unphishable, the attacker stops attacking your login and starts on the "lost my key" flow. If recovery falls back to an SMS code, a security question, and a help-desk call, you armor-plated the front door and left the back one open. Not theory: Scattered Spider's whole method against large enterprises in 2023 was phoning the help desk and talking them into resetting MFA. Every phishable fallback you keep "just in case" is a downgrade path. Your MFA is only as strong as the weakest factor a user can enroll or recover with, never the strongest one they own.

## What actually fixes it, and what it costs

I tell teams to enroll two phishing-resistant authenticators per person, so losing one does not dump the user into a weak recovery path. Hardware keys run about 25 to 50 dollars each, times two, times headcount, plus shipping and enrollment labor. Then strip the phishable fallbacks out of the auth policy. In Entra you hold that line with an authentication-strength Conditional Access policy that accepts only FIDO2, Windows Hello, or certificate auth. The sign-in logs, in the authentication-method detail, show who is still getting in on a weaker factor, so you can chase the tail. This step generates help-desk tickets and irritated mail. It is also the one that matters.

Then rebuild recovery to the standard of login: re-enrollment from an already-trusted device, in-person or manager-attested for edge cases, video-and-ID for remote staff, and a help desk that will not reset a factor over the phone without hard identity proofing. That is slower for users and costs more to staff. It is also exactly where the capable attacker is now standing. Spend there, or the passkeys were theater.
