---
title: 'Hybrid identity in a homelab: syncing on-prem AD to Okta and Entra'
description: 'IAM job ads ask for hybrid identity and directory sync experience. Instead of just reading about it, I built the real thing.'
pubDate: 'Aug 15 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

> Draft — this one is pure CV gold for an IAM role. It *is* the job.

IAM job ads ask for "hybrid identity / directory sync" experience. So I built the
real thing in a homelab instead of just reading about it.

## What I actually built

- On-prem **Active Directory** (domain HOMELAB) running on Proxmox
- **Entra Connect Sync** pushing users into Microsoft Entra ID with Password Hash Sync
- **Okta** AD import and reconcile, managed as code with Terraform (CI green)
- A joiner / mover / leaver lifecycle flow across both directories

## The bit I got stuck on

*(Tell it straight — this is where the value is.)*

- Security defaults re-enabling and breaking headless auth — what happened, how I caught it
- The design split: Terraform owns the groups, AD owns the membership — and why that boundary matters

## Why it matters for the role

Maps directly to the language on the job spec: directory synchronisation, SSO,
least privilege, identity lifecycle management.

*Tech: Active Directory, Microsoft Entra ID, Okta, Terraform, Proxmox.*
