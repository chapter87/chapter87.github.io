---
title: 'You disabled the account. It kept working.'
description: 'Disabling the directory account removes the login you can see and leaves the ones you can''t: local app passwords, API tokens and access keys, service accounts, and the resources the leaver still owned.'
pubDate: 'Aug 18 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

The ticket was closed the day she left. I disabled her AD account myself and watched the SSO tile drop off her dashboard. Eight months later her name surfaced in the finance app's audit log: a successful login at 02:14, from an address that wasn't ours. The directory account was still disabled. It had never mattered. The finance app kept its own local password, set during onboarding "in case SSO was down," and nobody had ever told the app she was gone.

That is the leaver problem. Offboarding removes the one identity you can see and leaves the dozen you can't.

## Disabled is not gone

Flip an AD account to disabled and you generate Event ID 4725. It feels final. It isn't. A workstation that is already logged in holds a Kerberos TGT valid until it expires, ten hours by default under MaxTicketAge, and every service ticket it already minted keeps working until then. Disabling the account does not reach into that laptop and tear up the tickets.

The cloud version is worse. Disable a user in Entra and the access tokens already issued stay valid until they expire, roughly an hour by default and longer wherever someone extended the lifetime. Refresh tokens can ride for up to 90 days. Revoking sign-in sessions resets the clock, but unless Continuous Access Evaluation is on for the resource, Graph and Exchange honour that token until it dies of old age. "Disabled in the portal" and "cannot call the API" are minutes to hours apart.

## The long tail nobody provisioned

SSO is the front door. The problem is every app with a back door.

Most SaaS supports SSO and a local login at the same time. You federate the app, turn on SSO enforcement, and leave the pre-existing local accounts in place because migrating them is work nobody scheduled. Under SSO enforcement those local credentials often still authenticate. It is the break-glass path you built on purpose. Deactivating the person in the IdP does nothing to a password living in the app's own store.

This is where orphans breed. An orphan is an account with no matching active worker in the HRIS. They come from movers who switched teams and kept the old entitlements, from contractors who were never in the HRIS to begin with, from accounts somebody created by hand outside the IGA tool, so the tool does not know they exist to deprovision them. You will not find these by waiting for a termination webhook. You find them by reconciliation: pull every app's user list, left-join it against the active-worker roster, and treat every unmatched row as guilty until explained.

## A token doesn't have a person

Access tied to an interactive login dies with the login. Access tied to a secret does not.

A personal access token the leaver minted in the CI platform authenticates as the token, not as the human. An AWS IAM user's access key, the AKIA kind rather than the short-lived ASIA, keeps signing requests long after console SSO is gone, because the key was never part of the login flow. A deploy key she pasted onto a repo is not attached to her user at all; it sits on the repository forever, and removing the person changes nothing.

Run aws iam list-access-keys across your accounts after a departure and you find out how much you missed. Every one of these has to be enumerated and revoked per platform. There is no master switch.

## The things they owned

The last category is the one write-ups skip: ownership. People accumulate it. The sync account she set up "temporarily" and still knows the password to. The distribution list, the app registration, the Terraform state, the on-call schedule, the shared drive, the domain renewal. Disable the human and none of it disappears. It goes unowned instead, which is worse, because now a production dependency answers to nobody and rotates on no schedule.

Deprovisioning that ignores ownership does not reduce risk. It hides it.

## What actually fixes it, and what it costs

Stop trusting events. Termination webhooks handle the happy path and miss everything the IdP never knew about. The control that works is boring: periodic reconciliation against an authoritative worker list, with a named human who owns the exception queue. That costs three things people underestimate. One is API read access into all thirty-plus apps, and plenty of vendors gate their API behind the same top pricing tier as SSO. Call it the SSO tax. Another is a canonical identifier to match on, because email is not stable across systems. The last is someone's real hours, every week, working the mismatches by hand.

Then kill long-lived credentials as a class. No standing IAM users holding access keys; federate to short-lived STS instead. Cap token lifetimes so a forgotten PAT expires on its own. Turn on CAE so revocation lands in minutes, not hours.

And make ownership a required field, not a nicety. Every service account, group, and automation carries a primary and secondary owner, and the leaver checklist refuses to close until whatever she owned has been reassigned to someone still here. The cost there is honest toil: a catalogue somebody keeps current, forever.

I used to treat the disabled account as the finish line. It was the part that never mattered.
