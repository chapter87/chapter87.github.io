---
title: 'The account whose owner left two years ago'
description: 'Machine identities outnumber staff eighty to one, almost nobody can list them, and the dangerous ones are the orphans no one will risk switching off.'
pubDate: 'Aug 10 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

Every environment I have audited has at least one: a service account named something like `svc-datasync`, created three or four years ago, holding rights it should never have been granted. The person who made it has left. There is no ticket, no owner in the description field, no expiry date. Its `pwdLastSet` is nine hundred-something days in the past. When I ask the obvious question, "what breaks if we disable it?", nobody in the room can answer. That uncertainty is why it is still switched on. It is load-bearing precisely because nobody understands it well enough to remove it.

This is the whole problem in one account. Now count how many you have.

## You are outnumbered, and you cannot see them

In a typical enterprise, machine identities outnumber human ones by roughly eighty to one. In cloud-heavy shops the ratio runs past five hundred to one: service accounts, API tokens, OAuth grants, workload identities, CI runners, agents. Against that, one commonly cited survey puts the share of organizations with a full inventory of their service accounts under six percent. I believe it. Ask a security team for a complete, current list of their non-human identities and you get a spreadsheet that was accurate the day it was made and wrong the day after.

Humans are governed. They get onboarded, reviewed, and offboarded, with HR as the system of record and a real leaving process. Service accounts have none of that. They are born in a hurry and they never die.

## Why they rot

The mechanics rarely vary. A deploy fails at 2am. Someone creates an account to get it working and grants it far more than it needs (Domain Admin, or a wildcard cloud policy) because scoping it properly costs time nobody has at 2am. `DONT_EXPIRE_PASSWORD` goes on so an expiry can never force a change and break the service. The secret goes into a config file, probably several. The change works, the incident closes, and nobody walks the permissions back afterwards. No quarterly access review ever looks at it, because it is not a person. No one rotates the password, because it lives in files whose full list was lost long ago.

Over-privileged, immortal, undocumented, unrotated. Now multiply by the ratios above.

## What does not work

You cannot fix this by disabling accounts that look idle. A batch job that runs once a quarter stays silent for 89 days and then authenticates. Kill it on a 60-day last-logon heuristic and you take down billing on day 90. Stale-looking and dead are not the same state, and the distance between them is where you cause the outage you were trying to prevent.

Rotation is not a checkbox either. Changing a classic service account password means a coordinated change across every host that stores the secret, and the reason it sprawls is that nobody has that list. That is why rotation stalls. It is not laziness; rotating blind is a bet on which service falls over.

gMSAs solve rotation properly. The KDC manages a 240-byte password in `msDS-ManagedPassword` and rolls it every 30 days, with nothing in a config file. But a gMSA only covers a service running under the Service Control Manager on a domain-joined Windows host. It does nothing for the Python script on a Linux box, the SaaS-to-SaaS OAuth grant, or the vendor integration authenticating with a JSON key. Vaulting carries the same blind spot: CyberArk or HashiCorp Vault will rotate what you onboard, and your problem is every account you never onboarded.

## Inventory from the logs, not the directory

The directory tells you what exists. It does not tell you what is alive, or who owns it. So I do not start there.

I start from authentication. On Windows that means Kerberos service-ticket requests (Event ID 4769) plus logons of type 5 (service) and type 4 (batch), pulled across a rolling 90-day window so the quarterly jobs surface. In AWS it is the IAM credential report: `access_key_1_last_used_date` read against `access_key_1_last_rotated` tells you which keys are live and which have not been touched since they were minted. Every authenticating identity carries a source. The account will not name its owner, but the host it logs in from usually can.

Then two diffs. Directory accounts that never authenticate in the window are decommission candidates. Identities that authenticate but sit nowhere in my inventory are shadow, and those are the interesting ones. I assign an owner at the moment of discovery, taken from whoever owns the source host, because the account's own description field is empty and always will be. While I am in there, I flag any account carrying an SPN with RC4 still enabled. Those are Kerberoastable, hashcat mode 13100, and an orphan with a weak password is a lateral-movement gift.

## What actually fixes it, and what it costs

The fix is not a product. It is making non-human identities behave like the human ones: an owner that is mandatory at creation and rejected when blank, an expiry that defaults to on so neglect deletes an account instead of preserving it forever, and a leaver process that reassigns machine identities when their creator walks out the door.

The cost is honesty and a little breakage. You will not learn what is load-bearing by reading documentation, because there isn't any. You learn it by turning things off in a controlled way and watching what screams — inside a change window, with a rollback ready, never on a Friday. Every orphan you retire is one fewer set of forgotten privileges for an attacker to inherit. The number only moves once you accept that the boring governance work, not the tooling, is the job.
