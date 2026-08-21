---
title: 'It''s always the same six OAuth2 mistakes'
description: 'Six OAuth2 misconfigurations turn up on nearly every review I do: loose redirect_uri, decorative PKCE, a lingering implicit flow, over-broad scopes, tokens in logs, unchecked state, and the RFC 8693 token exchange that comes after them.'
pubDate: 'Aug 23 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

The fastest finding I ever pulled out of an OAuth integration took under a minute. I copied the authorization request out of the address bar, pasted it into an editor, and changed one parameter. `redirect_uri` went from the app's real callback to a subdomain I controlled under the parent domain. The authorization server handed the victim's code straight to me. Nobody had exact-matched the callback, so any subdomain of the right domain passed, and I owned one.

That was one of six mistakes behind almost every OAuth finding I write up. None of it is the spec's fault. Implementers cut the same corners every time, and each one stays invisible until someone changes a parameter and watches. Here are the six, with what an attacker does to each and how you catch it.

## The redirect_uri that matches too much

The spec and the 2025 Security BCP (RFC 9700) want exact string matching on `redirect_uri`. Production hands you prefix matching, subdomain wildcards, or a valid registered host that also runs an open redirect. The attacker registers `https://app.corp.example.com.evil.example/`, or chains a `?next=` redirect on the real host, and the code rides out to them. Treat the registered value as the only legal one and attack it: change the host, append a path, add `@evil.example`, send a second `redirect_uri`, downgrade `https` to `http`. Anything that still receives the code is loose matching. Grep the export for `*` in callbacks too.

## PKCE that isn't there, or isn't enforced

Public clients (SPAs, mobile apps) can't keep a secret, so a stolen code is bearer-grade unless PKCE (RFC 7636) binds it to the client that started the flow. Two failures look identical from outside. First: no `code_challenge` in the authorize request. Second, the one people miss: the client sends `code_challenge`, but the token endpoint will still redeem a code with no matching `code_verifier`. That's decorative PKCE, and worse than none because it looks handled. Test enforcement, not presence. Capture a real code, POST it to the token endpoint with no verifier, and if tokens come back it's theatre. Reject `code_challenge_method=plain`; you want S256.

## The implicit flow that won't die

`response_type=token` returns an access token in the URL fragment. No code exchange, no client authentication, the token sitting in browser history and readable by any script on the page, and no refresh token, so apps fake it with hidden iframes. RFC 9700 and the OAuth 2.1 draft agree: stop using it, move to auth code plus PKCE. Grep for `response_type=token` and `id_token token`, then open the client config. In Entra ID it's the token checkboxes under Authentication; in Auth0, the Implicit grant toggle. It's almost always left over from a 2016 tutorial.

## Scopes nobody actually scoped

A third-party app asks for `Directory.ReadWrite.All` to read one group, or full Drive when `drive.file` would do. Users click Accept; the consent screen is background noise. Now a long-lived token with a wide blast radius sits in someone else's database. This is the machinery behind consent phishing (MITRE T1528): a malicious app that never asks for a password, only a grant. Audit consented apps in Entra Enterprise Applications and Google Workspace app access, and flag the broad ones. Then check the half nobody checks: does the resource server enforce scope per endpoint? Call a write route with a read-only token. If it works, the scopes were decoration too.

## Tokens where they get logged

A bearer token is unbound: whoever holds it, uses it. The moment a code or token lands in a query string, it's in your access and proxy logs, and the `Referer` header the browser sends to every third-party pixel on the page. RFC 6750 already says keep tokens out of the query. Grep the logs and the SIEM for `code=`, `access_token=`, and `id_token=` in URLs, and read the callback page's `Referrer-Policy`. One honest caveat: `Referrer-Policy` stops the browser leaking onward, but it can't unwrite a token already in your access log, logged the instant the request came in. Tokens belong in the `Authorization` header or a POST body. Nowhere else.

## No state, or state nobody validates

`state` binds the authorization request to the browser session that began it. Drop it and you get login CSRF and authorization-code injection: the attacker starts a flow, keeps their code, then gets the victim's browser to finish the callback with it, quietly linking the victim into an attacker-controlled account. Missing `state` is the rare case. The common one is a `state` that's dutifully sent and never checked on return. Tamper the returned value at the callback; if you're still logged in, it's ornamental. For OIDC, `nonce` does the same job for the id_token, so validate that too.

## What actually fixes it, and what it costs

The six are hygiene: exact-match redirects, enforced S256 PKCE, no implicit, least-privilege scopes checked server-side, tokens out of URLs, validated state. Adopt OAuth 2.1 defaults and most become the easy path instead of a fight.

What interviewers probe now is the layer past hygiene: token exchange, RFC 8693. Rather than pass one broad user token through every hop, each hop calls an STS with `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`, presents the incoming token as `subject_token`, and gets one scoped to a single `audience` with a reduced `scope`. That kills the confused-deputy problem, where a downstream service acts with the user's full authority only because it received the user's full token. Delegation turns explicit through `actor_token` and the `may_act` claim. Azure's on-behalf-of flow is a proprietary cousin of the same idea.

The cost is real. You need an STS that genuinely implements 8693, and plenty of IdPs still don't. You pay a round trip per hop. And you've built a token-minting endpoint that has to answer "who may act as whom," which done wrong trades six small holes for one large one. Token exchange contains blast radius. It doesn't excuse the other six. I still open every review with the authorize URL and a text editor.
