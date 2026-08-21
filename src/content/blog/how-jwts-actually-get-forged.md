---
title: 'Most forged JWTs pass signature verification'
description: 'The JWT attacks that actually work satisfy the signature check instead of breaking it, which is why algorithm, audience and issuer validation matters more than the crypto.'
pubDate: 'Aug 21 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

The first time I forged an admin token in a review, I cracked nothing. I copied the RSA public key from the app's JWKS endpoint, the key it publishes on purpose, flipped the token header's `alg` from `RS256` to `HS256`, and signed my own payload with that public key as an HMAC secret. The middleware called `jwt.verify(token, key)`. The signature matched. I was `role: admin`. No secret guessed, no algorithm broken. It verified because I had handed it a signature it could verify.

The attacks worth worrying about don't defeat the signature check. They satisfy it. `alg:none` is the one everyone names and the one that matters least, because maintained libraries reject it outright. The forgeries that still land produce tokens whose signatures verify cleanly, and the only thing between the attacker and a live session is claim validation you wrote once and never went back to.

So here are the five I test for, and the exact mistake that opens each one.

## alg:none, the museum piece

The header says `{"alg":"none"}`, the signature segment is empty, and a naive verifier calls the token authentic. The enabling mistake is decoding without an algorithm allowlist, so the library honours whatever `alg` the token carries. Blocklists just invite bypasses: lowercase-and-compare against `"none"` and I send you `"nOnE"`. It's close to extinct in current libraries, but hand-rolled verifiers and ancient pinned versions still fall to it. If your defence is one string comparison, it isn't a defence.

## RS256 to HS256 key confusion

RS256 signs with a private key and verifies with the public one. HS256 uses a single shared secret for both. Change the header from `RS256` to `HS256`, sign with the server's public key as the HMAC secret, and a verifier that trusts the algorithm in the token will run HMAC with that public key. The key is public, so you already hold the secret. The bug is one line: `verify(token, key)` where `key` is the RSA public key and the algorithm comes from the header instead of being pinned. Tim McLean wrote this up in 2015 and it still ships. One honest catch: you need the exact public-key bytes the server uses, PEM encoding and trailing newline and all. Reconstruct the PEM wrong, the HMAC differs, nothing verifies. When the app exposes a JWKS with `n` and `e`, that reconstruction is a scripted step rather than a guess.

## A weak HMAC secret is one wordlist away

HS256 is only as strong as its secret, and defaults and placeholders like `your-256-bit-secret` ship constantly. Capture one token and the crack is offline and silent: hashcat mode 16500 chews a JWT against rockyou in seconds. RFC 7518 section 3.2 says the HMAC key must be at least as large as the hash output, 256 bits for HS256, and a dictionary word is nowhere near 256 bits of entropy. Rotating one weak secret to another weak secret fixes nothing. The fix is 32 random bytes from a real CSPRNG, nothing you can type from memory.

## kid, jku, x5u: when the header picks the key

`kid` tells the verifier which key to use. The failure is letting it reach the key instead of just naming it. `key = readFileSync(header.kid)` with `kid` set to `../../../../dev/null` hands you an empty key, and I sign with an empty HMAC secret. `SELECT key FROM keys WHERE kid = '<kid>'` string-interpolated turns into `' UNION SELECT 'attacker-secret`. Worse are `jku` and `x5u`, which point at a JWKS or certificate URL: honour them and I host my own keys and give you the link. Any header field that names or fetches key material is attacker input.

## The claims nobody checks: exp, aud, iss

This is where real systems bleed. A token with no `exp` never expires, and plenty of libraries only check expiry when the field is present, so a stolen token is permanent. Skip `aud` and a token minted for the throwaway analytics service, signed by the shared issuer, replays against the admin API. Skip `iss` and a token from another tenant, or a sibling issuer sharing key material, walks straight in. Every one of these signatures is genuine. `jwt.verify(token, key, {algorithms:['RS256']})` with no `audience` and no `issuer` option is the modal production bug, and it looks completely fine in review.

## What actually fixes it, and what it costs

Pin an algorithm allowlist to exactly what you issue, and never read `alg` from the token. Treat `kid` as an opaque lookup into a JWKS you pulled from a hardcoded issuer URL, never as a path or a query fragment, and ignore `jku` and `x5u` outright. Require `exp` to be present. Check `exp` and `nbf` with about 60 seconds of leeway for clock skew, and assert `aud` is this service and `iss` is your issuer. For OAuth access tokens, RFC 9068 wants a `typ` of `at+jwt` plus a real `sub` and `client_id`.

None of this is free. JWKS rotation means caching keys with a TTL and tolerating two valid `kid`s during rollover, and any fetch-on-unknown-kid path is an SSRF and DoS vector unless the URL is pinned and the fetch rate-limited. Bigger problem: a JWT can't be revoked mid-life. Stateless auth buys you no server session and charges you short lifetimes, five to fifteen minutes, plus refresh tokens, or a `jti` denylist in Redis that puts back the state you were trying to avoid. And that WAF rule blocking `alg:none`? Theatre. It stops the one dead attack and none of the live ones, because a key-confusion or missing-`aud` forgery is a byte-for-byte valid token.
