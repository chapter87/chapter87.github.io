---
title: 'The sync account owns your domain'
description: 'The directory-sync service account holds DCSync rights by design, yet shows adminCount 0 and joins no privileged group, so the group-membership audit that should catch it looks straight past it.'
pubDate: 'Aug 7 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

I keep finding the same account on internal assessments. The privileged-group audit is clean: Domain Admins has the four people it should, Enterprise Admins is empty, everyone sensitive sits in Protected Users. Then I pull the ACL on the domain head and there is a user object, call it the directory-sync account, that can replicate every secret in the domain. Its adminCount is 0. It belongs to no group that matters. On paper it is a service account for the cloud. In practice it can hand me the krbtgt hash.

The gap between what that account can do and what a membership audit shows is why it survives review after review.

## Why the sync account holds the keys

When you stand up on-prem-to-cloud identity sync with password hash sync turned on, the installer creates an AD service account (the classic name is MSOL_ followed by a hex string) and grants it two control-access rights on the domain naming context: Replicating Directory Changes (GUID 1131f6aa-9c07-11d1-f79f-00c04fc2dcd2) and Replicating Directory Changes All (1131f6ad-...). Together, those two rights are DCSync. They exist so the sync engine can read password hashes and push them to the tenant. This is not a misconfiguration. It is the feature working as designed.

The rights live as ACEs on the domain object itself. Not through Domain Admins. Not through any protected group. So the account carries no adminCount flag, AdminSDHolder never touches it, and every audit that enumerates group membership, which is most of them, returns nothing.

## Why gMSA doesn't save you

Newer and hardened deployments run the sync service under a group Managed Service Account. That feels safer: no password in a config file, a long machine-generated key it rotates itself every 30 days. It changes nothing about the exposure.

The gMSA's managed password is retrievable by any host listed in its msDS-GroupMSAMembership attribute, and the sync host is on that list because it has to be. Get SYSTEM on that host and you can pull the password blob and use the identity from anywhere. The connector account that actually authenticates into AD is worse. Its credential sits encrypted in the sync engine's database, DPAPI-protected under the service account, and a host admin can decrypt it to cleartext in seconds. gMSA solves password-in-config and rotation. It does nothing about a host compromise, and host compromise is the whole attack.

## The chain, end to end

Compromise the sync host, often the softest Tier 0 box in the building because nobody files it as Tier 0. Recover the connector credential or the gMSA password. Run DCSync, pull krbtgt plus every user hash, and forge a Golden Ticket for domain persistence that outlives a password reset.

Then it stops being an on-prem problem. The same host holds the credentials that talk to the tenant. If Seamless SSO is enabled, on-prem AD contains a computer account (AZUREADSSOACC$) whose Kerberos key, by default, never rotates. DCSync it once and you can forge tickets that impersonate any synced user to the cloud, no password required. Your Active Directory breach is now a tenant breach. One host did that.

## What the usual detection misses

The standard DCSync alert is event 4662 on a domain controller with the replication GUIDs in the Properties field. It catches a random workstation running DCSync just fine. The trouble is the sync account fires that exact event legitimately, all day. So teams whitelist it, and the whitelist becomes the blind spot: abuse now looks identical to business as usual.

Pin the source. Legitimate replication comes from DC machine accounts, and from the sync account only when it originates on the sync host. The alert that matters is those replication GUIDs invoked by any other principal, or the sync account replicating from anywhere but its own host. That second clause catches credential theft, and almost nobody writes it.

## What actually fixes it, and what it costs

Audit ACLs, not groups. Read the DACL on the domain naming context and list every principal holding GetChanges and GetChangesAll. Do the same for WriteDacl and WriteOwner on the domain and on the sync account itself, because anyone who can rewrite that ACL can grant themselves DCSync tomorrow. In BloodHound these are the DCSync, GetChangesAll and WriteDacl edges. Cost: it is continuous. ACLs drift, and a clean result expires the day someone delegates a permission.

Tier the host. The sync server is a domain controller wearing a middleware costume, so staff it like one: admin only from a privileged access workstation, no mail, no browsing, patched on DC cadence, no workstation-admin RDP landing on it. And no, you cannot just drop the account into Protected Users. That group blocks NTLM and caps ticket lifetime, and service accounts break under it. Cost: you lose the convenience of a utility box, and PAWs are not free.

Rotate the things that do not rotate themselves, chiefly the AZUREADSSOACC$ Kerberos key on a real schedule if you use Seamless SSO. Cost: manual, and easy to forget for years at a time.

The honest limit: you cannot strip the privilege without killing the feature. Password hash sync requires replication rights. Move to pass-through auth or federation and you have only relocated the crown jewel, to an agent that can be hooked or to a token-signing key and the Golden SAML problem. Every option keeps one high-value secret somewhere. You cannot delete it. You can know exactly where it lives, run its host like the Tier 0 box it is, and alert on the ACL path instead of the group list. The cost is admitting your identity bridge is Tier 0, and finally paying to run it that way.
