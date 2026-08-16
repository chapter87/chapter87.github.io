---
title: 'Breaking into a deliberately-vulnerable Android app, exercise by exercise'
description: 'I set up a real mobile pentest lab on my own phone against AndroGoat, an intentionally-insecure training app, and worked through the OWASP mobile classics: intercepting HTTPS, dumping secrets from exported components, and — the big one — running Frida on a phone I could not root.'
pubDate: 'Jul 16 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

Reading about mobile app vulnerabilities is one thing. Watching a bank-style login
screen hand you its stored password over a USB cable is another. To actually learn
Android security, I built a proper lab on my own phone and went after
**AndroGoat** — an OWASP training app that is deliberately full of the exact flaws
real apps ship with. Here's the tour.

## Setting up interception on a phone I don't fully control

The foundation of app testing is seeing the traffic. I ran an intercepting proxy on
my laptop and tunnelled the phone's traffic to it over USB — no need to trust the
Wi-Fi — then installed the proxy's certificate so I could read HTTPS.

Plain HTTP and HTTPS decrypted immediately. **Certificate-pinned** connections did
not — and that's correct behaviour, an app pinning its server's certificate is doing
security *right*. Getting past that became the headline challenge below.

## The OWASP classics, each one landed

- **Secrets sitting in the app.** Decompiling the app surfaced hardcoded promo codes
  and planted fake cloud keys — the everyday mistake of shipping credentials inside
  the package where anyone can read them.
- **Exported components leaking data.** This was the eye-opener. Several of the app's
  internal components were exported with no permission guard, meaning *any other app
  on the phone* — not just me with a cable — could invoke them. One content provider
  handed over a table of usernames and passwords on request. A broadcast receiver
  leaked credentials in a notification. An activity let me jump straight past the
  login screen to the logged-in view. No exploit chain, just asking politely.
- **Insecure storage.** Using the app's own debug access, I read its private files:
  passwords sitting in a local database and in preferences as **plain text**. On a
  lost or backed-up phone, that's game over.
- **Insecure logging.** The app logged usernames and passwords to the system log on
  every login — readable by other privileged apps and by anything that scoops up
  crash reports.
- **QR-code XSS.** A screen rendered scanned QR text straight into a web view with
  scripting enabled and no encoding, so a crafted QR code executed JavaScript inside
  the app. I extended it to the real-world version — "quishing" — where a QR encodes
  a *link* that opens the phone's browser on an attacker page. That's the one that
  works with any normal camera, which is why those "scan to pay" codes deserve
  suspicion.

## The big win: Frida on a phone that can't be rooted

Here's the result I'm proudest of. My phone is a modern Samsung with a locked
bootloader and Knox — genuinely *cannot* be rooted without tripping a hardware fuse
and wiping the device, which I wasn't about to do. Most Frida tutorials assume root.
So how do you get a dynamic-instrumentation hook into an app on a phone you can't
root?

The answer is to **repackage the app itself** with a Frida "gadget" injected into
it, then install your modified copy. The instrumentation rides *inside the app*, so
it needs no system-level root at all. With that in place I hooked the app's
certificate-pinning check and switched it off at runtime — and the pinned connection
that had defeated me at the start decrypted cleanly in my proxy. I used the same
technique to force the app's "is this device rooted?" check to lie, and to sail
straight through a biometric prompt with no fingerprint at all, by making the
success callback fire directly.

That's the lesson worth carrying: **client-side security checks run on hardware the
attacker controls.** Pinning, root detection, biometric gates — all of them can be
subverted by someone instrumenting the app, even without rooting the phone. They
raise the bar; they are not a wall. Real security has to live on the server, because
the client can always be turned against itself.

## Why this mattered to me

This is the closest thing to the actual job of a mobile security tester, done safely
on a training target on my own device. Every finding maps to the OWASP Mobile
standard, and more importantly, each one rewired how I think: never trust the
client, never store secrets on the device, never export a component without asking
who else can reach it.

*Tech: Android, an intercepting proxy over USB, jadx/apktool (static analysis), Frida + objection (gadget injection, no root), OWASP MASVS.*
