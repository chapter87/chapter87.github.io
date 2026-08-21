---
title: 'mTLS is easy to turn on and expensive to keep on'
description: 'The security win of mTLS hides an operational bill nobody budgets for: 2am cert-expiry outages, no inventory of what presents which cert, manual rotation that doesn''t scale, and revocation that quietly doesn''t work.'
pubDate: 'Aug 20 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

A batch job that had run clean for a year stopped at 2:14 on a Sunday morning. Not slow. Stopped. The log line was `x509: certificate has expired or is not yet valid`. No attacker, no bad deploy, no config change. A date had passed. Someone issued that client certificate with a twelve-month life, a year ago, and the calendar caught up with us. The actual fix took four minutes. Finding the right private key took two hours, because nobody had written down which of our forty-odd services presented that cert.

That is the shape of nearly every mTLS incident I have worked. The cryptography does exactly what it promises. The operations around it are what fall over.

## mTLS earns its place

I am not going to talk you out of it. Mutual TLS is a real security win. Both sides present a certificate, both sides verify, and the workload's identity becomes the cert itself: a URI in the subject alternative name, signed by a CA you chose to trust. That is a proper non-human identity (NHI). It beats a shared bearer token sitting in an environment variable, waiting to leak into a log aggregator. A stolen cert without its private key is useless. Man-in-the-middle gets hard.

The catch is that a certificate is the one NHI credential with a built-in death date. That death is a safety feature. It is also the bill nobody budgets for.

## Four bills come due

**Expiry.** Every cert has a `notAfter`. It will be reached. Each one is a scheduled outage you did not schedule.

**Inventory.** You cannot rotate what you cannot find. Most shops have no map from workload to cert to CA to expiry date. They build one live, at 2am, under a Sev1.

**Rotation by hand.** Generate a CSR, get it signed, distribute it, reload the service. Fine at five services. At five hundred it is a full-time job, and every service that needs a SIGHUP or a restart to pick up the new key turns rotation into a change window with a rollback plan.

**Trust that ossifies.** You stand up a root CA once, push the bundle to every trust store, and never touch it again. The root is valid for a decade. If that key is ever compromised, rotating it means reaching every workload you own, which is why nobody does.

## The part the guides skip

Revocation mostly does not work. CRLs grow stale and enormous. OCSP bolts a network round-trip onto every handshake, so teams soft-fail it or switch it off. In practice, a compromised workload certificate stays trusted until it expires on its own. If your certs live a year, your blast radius after a key leak is a year.

The mTLS security model quietly assumes a working revocation path, and most deployments do not run one. The honest version of "mTLS protects us" is "mTLS protects us, minus a revocation window we are choosing not to look at."

## Short lifetimes flip the whole problem

The fix is not a better calendar reminder. It is making certificates live hours instead of months, and issuing them by machine.

SPIFFE with SPIRE is the clearest example. A workload proves what it is through attestation. Node attestation binds it to a machine, using a Kubernetes projected token or an AWS instance identity document. Workload attestation binds it to a process, checking its Unix UID or its Kubernetes service account. SPIRE returns a short-lived SVID: an X.509 certificate carrying the SPIFFE ID in the URI SAN, default TTL one hour, rotated before it lapses.

Watch what that removes. Rotation is continuous, so there is no expiry event to sleep through. Revocation collapses into "stop renewing," and a leaked cert dies within the hour by itself. The registry of which workload may receive which identity is the inventory you never had. If SPIRE is too heavy, step-ca and cert-manager reach a similar place over ACME.

## What short-lived costs you

Name the trade honestly, or it will burn you.

You have swapped a slow, predictable failure for a fast, correlated one. If the identity plane goes down, nothing renews, and with one-hour TTLs your mesh starts failing inside the hour. The issuer is now tier-0 infrastructure, the same rank as DNS. Clock skew becomes a production concern: `notBefore` is unforgiving, and short windows leave little slack when NTP drifts. Attestation still has to bootstrap trust somewhere; SPIFFE's own authors named the book about it *Solving the Bottom Turtle*. And you will not fit the third-party appliance or the managed database into SPIFFE, so you keep a long tail of manual certs. That tail is exactly where you still get paged.

## What actually fixes it, and what it costs

Automation plus short lifetimes, run as real infrastructure rather than a spreadsheet of dates.

The cost is concrete. Run the issuer highly available and monitor it on issuance rate and TTL distribution, not just "is it up." Alert on any certificate whose remaining life is measured in months, because that is the one lying to you. For the long tail you cannot automate, put real expiry alerting on it, at 30 and 7 days, routed to a named human who owns that service. "Someone will notice" is the plan that produced the 2am page. Replace it with a system that renews on its own, and a short list of exceptions a person is paid to watch.
