---
title: 'The password reset that fixed nothing'
description: 'An OAuth consent grant hands an attacker a refresh token that survives the password reset, the MFA re-enrollment, and most of your incident response.'
pubDate: 'Aug 5 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

Someone in finance clicks a link. The page looks like a document-signing login, and they authenticate for real against your tenant. Real MFA prompt, real tap on the phone, real approval. Then a Microsoft consent screen asks to read their mail and "maintain access to data you have given it access to." They click Accept. That second permission is `offline_access`, and the click just handed an attacker a refresh token.

You find out four days later, when the mailbox starts forwarding supplier invoices to an address nobody recognizes. You do the correct things. Reset the password. Revoke and re-enroll MFA. Sign out every session. The forwarding continues. Nothing you did touched the problem, because the problem was never the password.

## The checkbox is the whole attack

This is the illicit consent grant. An attacker registers a multi-tenant app, or compromises a legitimate one, then sends a normal-looking consent URL pointed at your own authorization endpoint, `login.microsoftonline.com/common/oauth2/v2.0/authorize`, carrying a scope string like `offline_access Mail.Read Mail.Send Files.ReadWrite.All`. The victim authenticates against the real Microsoft, so everything on screen is genuine. On Accept, the tenant records an `oAuth2PermissionGrant` and the app trades the authorization code for an access token good for about an hour, plus a refresh token.

That refresh token is the prize. As long as it keeps getting used, it rolls forward on a 90-day inactivity window, minting fresh access tokens on demand. No password enters the loop after that first consent. The "integration" the user approved is a persistence mechanism with a redraw button.

## Why the controls you trust miss it

Password reset is the reflex, and here it does nothing. Microsoft's own token-revocation table draws a line between public clients (a phone, a desktop app) and confidential clients (an app registration holding a client secret). A password change or admin reset revokes refresh tokens for public clients. For confidential clients it does not. A registered OAuth app with a secret is a confidential client, so its refresh token survives the reset by documented design.

MFA fares no better. The victim already cleared MFA to reach the consent screen, and refreshing an access token is a non-interactive flow that never re-prompts. The attacker rides the MFA the user already passed.

Disabling the account helps more, because a refresh eventually fails against a disabled user. An *application* permission is worse. Take Exchange's `full_access_as_app`: no user sits in the loop at all. The app authenticates as itself with its own credential, so you can disable every human in the tenant and it keeps running.

## The named version

In January 2024, Microsoft disclosed its own version of this. APT29 (Midnight Blizzard) password-sprayed a dormant test tenant account that had no MFA, found a legacy OAuth app already holding elevated access to the corporate environment, then stood up more malicious apps and granted them the `full_access_as_app` role over Exchange Online. They read senior leadership mailboxes for weeks. The initial foothold was a weak password. Everything after it was OAuth. Resetting that first account's password would have shut the door they walked in through and left every door they built wide open.

This pattern carries its own MITRE technique: T1528, steal application access token, feeding T1550.001, use it. Watch for the credential-addition variant too, where an attacker adds their own client secret to an existing over-privileged app. It needs no phishing and surfaces only as `Add service principal credentials` in the audit log.

## Finding the grants you already have

In Entra, pull every grant through Graph: `oauth2PermissionGrants` for delegated consent, `appRoleAssignedTo` for application permissions, `servicePrincipals` for the apps themselves. ROADrecon, AzureHound, or the older `Get-AzureADPSPermissions.ps1` will dump the lot. Rank by scope. Treat `offline_access` beside any `Mail.*`, `Files.ReadWrite.All`, `Directory.Read.All`, `full_access_as_app`, or a bare `.default` as suspect. Cross-check `passwordCredentials` and `keyCredentials` for secrets added recently, especially by anyone other than the app owner. Pull sign-in activity for apps that consented once and then went dormant, or suddenly got busy. Four audit operations carry the story: `Consent to application`, `Add delegated permission grant`, `Add app role assignment to service principal`, and `Add service principal credentials`.

Okta has the same shape. `GET /api/v1/users/{id}/grants` lists a user's OAuth grants, the System Log records consent as an `app.oauth2.as.consent.grant` event, and a `DELETE` on the grant revokes it. Same lesson underneath: the grant and its refresh token outlive the password.

## What actually fixes it, and what it costs

Revoke properly. `Revoke-MgUserSignInSession` kills a user's refresh tokens, but app-only access means you also delete the service principal, or at minimum the attacker's added credential and the permission grant. Miss the service principal and the app keeps authenticating as itself.

Shrink the blast window. Continuous Access Evaluation turns "wait up to an hour for the token to expire" into near-real-time revocation, though only for CAE-aware clients, and not everything honors it.

Stop the next one. Set user consent to off, or to verified publishers with low-impact permissions only, and stand up the admin consent workflow so integrations get reviewed before they exist. Restrict who can register applications while you are in there.

The cost is real, and it is operational, not technical. Turn user consent off and every SaaS trial becomes a ticket, so someone has to own that queue with a turnaround people will actually wait for, or shadow IT routes around you. Cleaning old grants means deleting things nobody documented and occasionally breaking a live integration to learn what it was for. There is no free version of this. The expensive version is still cheaper than a refresh token quietly reading your CFO's mail for ninety days.
