---
title: 'SAML forges logins, OIDC leaks tokens, and "just use OIDC" is wrong'
description: 'The failures that actually bite are XML signature wrapping in SAML and token confusion in OIDC, and which protocol you run is dictated by the app you federate to, not by preference.'
pubDate: 'Aug 24 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

I once logged into a SAML application as a user who was never provisioned. No stolen password. I took a response the identity provider had already signed, pasted a second assertion into it, and let the two halves of the validator disagree about which assertion counted.

That gap is most of the SAML attack surface, and it has a name: XML Signature Wrapping.

## The signature is valid. It is guarding the wrong element.

SAML signs XML. The signature carries a Reference whose URI points at the ID of the assertion it covers. Validation resolves that ID, checks the digest, confirms the crypto. Then a separate piece of code, the application's SAML consumer, walks the document to read the NameID and attributes and decide who you are. Nothing forces those two code paths to land on the same element.

So you keep the signed assertion intact, move it into a wrapper the verifier still resolves by ID, and inject a forged assertion where the consumer happens to look, usually the first `<Assertion>` returned by `getElementsByTagName`. The signature check passes over the original. The identity read returns your forged NameID. Somorovsky and colleagues documented this in their 2012 USENIX paper "On Breaking SAML," where eleven of the fourteen frameworks they tested fell to it. Fourteen years on I still find it in home-grown validators, because the fix runs against instinct: the code that verifies the signature has to hand the consumer the exact element it verified, not the whole document.

## SAML also breaks when nobody is attacking it

The quieter SAML failure is time. Every assertion carries `Conditions` with `NotBefore` and `NotOnOrAfter`, and the SubjectConfirmationData carries its own `NotOnOrAfter` plus an `InResponseTo` that has to match the request the SP sent. That last field makes SAML stateful: the SP has to remember an outstanding request ID across a browser redirect.

Put the IdP behind a load balancer and let one node's clock drift four minutes. Half your logins fail with "assertion not yet valid," and which half depends on which node answered. Most SPs allow only a few minutes of skew, 180 seconds being a common default; past that the `NotBefore` sits in the future and the assertion is thrown out before anyone reads it. I watched a team chase this as an intermittent SSO outage for a day before someone thought to run `ntpq`. The protocol was correct. The clocks were not.

## OIDC does not save you. It moves the traps.

OIDC is cleaner to operate: JSON, no canonicalization, stateless validation against a JWKS endpoint. The traps just moved.

The oldest is the implicit flow. `response_type=token`, the access token delivered in the URL fragment, straight into browser history, Referer headers, and server logs, everywhere a URL quietly comes to rest, with no client authentication anywhere. RFC 9700, the OAuth Security Best Current Practice finalised in 2025, deprecates it outright. Use authorization code with PKCE and stop.

The subtle one is confusing the two tokens. The ID token's audience is your client; it exists to tell your app who just logged in. The access token's audience is an API; it is a bearer credential for calling something. Send the ID token to an API as a Bearer and you have handed an identity document to a service that asked for an access grant. Worse is the API that accepts it: any signed JWT, no `aud` check, no `iss` check. Chain that with `alg:none` or an RS256-to-HS256 key confusion and the signature stops meaning anything. And do not drop `nonce` and `state`. The `nonce` binds the ID token to one authentication and kills replay; `state` binds the callback to this browser and kills login CSRF. They read like boilerplate. They are the boilerplate that does the work.

## When I still reach for SAML in 2026

"Just use OIDC" is advice for a greenfield most people don't have. You rarely pick the protocol; the application you federate to picks it for you, and a large slice of enterprise SaaS plus nearly every older on-prem app speaks SAML 2.0 and nothing else. Research and education run on SAML federations: InCommon, eduGAIN, the UK Access Management Federation. None of them are migrating on your timeline. If the estate is ADFS-shaped, SAML is the native tongue.

OIDC wins where you genuinely have the choice: native mobile (authorization code and PKCE per RFC 8252, because SAML has no honest mobile story), single-page apps, service-to-service calls, anything API-first. Real estates are both, indefinitely. An IAM engineer who can operate only one of them is half-equipped.

## What actually fixes it, and what it costs

For SAML, never hand-roll assertion validation. The move that feels safe but doesn't work is validating the schema and confirming the signature verifies: a wrapped document is schema-valid and the original signature does verify, which is the whole trick. What works is a maintained library that verifies the signature over the exact element it returns to you, hardens the parser, and rejects any response carrying more than one assertion. Then run NTP like it matters, because it does. The cost is a hard dependency on that library's assumptions and upgrade cadence, plus validation logic you can't easily read at 2am when it breaks.

For OIDC, authorization code with PKCE, every time. Verify `iss`, `aud`, `exp`, and `nonce`, and check the signature against the right JWKS key with the algorithm pinned. Keep the two tokens in separate mental buckets and never let one do the other's job. The cost is extra round trips and the discipline to run five checks on a token that looks perfectly fine after the first one.

Neither protocol fails in the demo. Both fail in the estate. The work is knowing which failure you signed up for.
