---
title: 'My SIEM was installed for two months and watching nothing'
description: 'I built a security operations centre for my home lab and discovered the SIEM I had already deployed was monitoring exactly zero machines. Here is what it takes to go from installed to actually detecting, and the two configuration traps that make a healthy-looking install useless.'
pubDate: 'Aug 18 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

My home lab has always leaned offensive. Wireless auditing, vulnerability scanning,
a phone running a full pentest distro, an agent that pulls fresh CVEs every morning.
What it did not have was anything watching the other direction. If someone had spent
a night brute-forcing SSH against my DNS box, I would have found out roughly never.

So I set out to build the defensive half: a small security operations centre for my
own network. What I found when I started was more embarrassing than I expected.

## The install that wasn't monitoring anything

I already had a SIEM. I'd deployed Wazuh into a container on one of my hypervisor
nodes a couple of months earlier, and by every surface-level measure it was fine.
Manager, indexer, dashboard and log shipper all running. Every port listening. The
web interface loaded and looked exactly like a working security platform.

Then I listed the enrolled agents:

```
ID: 000, Name: wazuh (server), IP: 127.0.0.1, Active/Local
```

That's it. That's the whole list. Agent 000 is the manager monitoring itself. Every
other machine on my network — the hypervisors, the domain controller, the identity
VM, the DNS box, my laptop — was invisible to it.

This is the failure mode nobody warns you about. A SIEM does not error out when it
has no data sources. It sits there looking healthy and green and tells you nothing,
because nothing is what it has been given. **A dashboard that loads is not a dashboard
that's working.**

If you have deployed a monitoring tool and never explicitly verified what it is
receiving, go and check right now. I'd put money on some of you finding the same
thing.

## Trap one: waking a machine on the wrong broadcast address

My lab nodes are normally powered off, so step one was waking them remotely with
Wake-on-LAN. I had a note from a previous session with the exact magic-packet
command, which had reportedly worked before. I ran it. Nothing happened.

The note sent the packet to a broadcast address ending `.255` on the same third
octet as the machines. That is correct for a /24 network — the default most people
assume. My LAN is a /22, four times bigger, and its actual broadcast address is
three octets higher.

Wake-on-LAN is fire-and-forget UDP. There is no acknowledgement, no error, no
feedback of any kind. Sending a magic packet to the wrong broadcast address looks
*identical* to sending a correct one. The packet went nowhere and the command
reported success.

Check your netmask rather than assuming:

```
$ ifconfig en0 | grep "inet "
inet 192.168.x.x  netmask 0xfffffc00  broadcast 192.168.x.255
```

`0xfffffc00` is /22. The interface will tell you its real broadcast address if you
ask it. Once I sent to the right one, both nodes were up in about ninety seconds.

The wider lesson: **a documented command that "worked before" is a hypothesis, not a
fact.** Especially for protocols with no return path.

## Getting the first agent on, and proving it

Rather than start with my most important machine, I picked a lab VM that nothing
depends on. If enrollment went sideways, nobody's evening would be interrupted.

Installing the agent is unremarkable — add the vendor repository, install the
package with the manager address and a name baked into the environment, enable the
service. Within a minute the manager showed the new agent transitioning through
`Never connected` to `Pending` to `Active`, and the agent had already run a CIS
benchmark scan against itself, taken a full software inventory, and completed a
rootkit check.

Here's the part I think matters most, and the part I see skipped constantly.

**I did not trust the "Active" status.** Active means a network connection exists. It
does not mean detection works. So I generated a real attack: five failed SSH logins
against the new agent using a username that doesn't exist.

```
Rule: 5710 (level 5) -> 'sshd: Attempt to login using a non-existent user'
Src IP: 192.168.x.x
Src Port: 51490
Invalid user baduser_soc_test from 192.168.x.x port 51490
```

All five landed, correctly attributed, with the right source address, port and
username — and automatically tagged against PCI-DSS, NIST 800-53, GDPR, HIPAA and
SOC 2 control references. That last part is free and genuinely useful if you are
studying information security management, because it maps raw technical events onto
the control frameworks the coursework actually cares about.

Now I know the pipeline works: endpoint, to agent, to manager, to rule engine, to
alert. Not because it looked right, but because I attacked it and watched the alert
arrive.

## Trap two: giving a container more RAM does nothing on its own

The container was running the full stack — manager, indexer and dashboard — in 4 GB.
The indexer is an OpenSearch fork, and OpenSearch alone typically wants 4 GB of heap.
The host had spare memory, so I raised the container to 8 GB.

On a container this applies live through cgroups, no restart needed. `free` inside
immediately reported 8 GB. Job done, apparently.

Except it wasn't. I checked what the indexer was actually using:

```
$ grep -E "^-Xm" /etc/wazuh-indexer/jvm.options
-Xms1024m
-Xmx1024m
```

The JVM heap was hard-coded to 1 GB, written at install time based on the memory the
container had *then*. The JVM does not care that the box got bigger. It reads its
options file and takes exactly what it's told. I could have doubled that container's
memory every week and the search engine would have carried on using one gigabyte
forever.

I raised the heap to 3 GB — roughly half the container's RAM, leaving the rest for
the manager, the dashboard and the filesystem cache the indexer relies on — and
restarted just that service. Cluster health came back green.

**Resizing a machine does not resize the software on it.** Anything with its own
memory configuration — a JVM, a database, a PHP pool — needs telling separately.

## Touching the box the whole house depends on

The most valuable sensor I own is the Raspberry Pi running DNS for the entire house.
Every device's every lookup passes through it, which makes it the single best place
to notice malware phoning home.

It is also the machine that, if I break it, takes the internet down for everyone
including people who did not consent to my hobby.

The agent is passive — it reads logs and watches files, it has no involvement in
resolving DNS — so the actual risk was low. But "low risk" is a prediction, and I
wanted evidence. So I baselined first:

```
google.com     142.251.x.x
bbc.co.uk      151.101.x.x
github.com     140.82.x.x
```

Installed the agent, then ran exactly the same lookups again. All resolving, DNS
filter and resolver both still up, load normal. Under a minute of work, and it turns
"it should be fine" into "it is fine".

If you are going to touch infrastructure other people depend on, capture the
before-state. It costs almost nothing and it is the difference between knowing you
didn't break something and merely hoping.

## What this is actually for

Four things, in order of how much I value them:

**Knowing when something bad happens.** Failed logins, new accounts appearing,
privilege escalation, services starting that shouldn't. The unglamorous core.

**File integrity monitoring.** The SIEM watches critical files and tells me the
moment one changes. This is how you catch a web shell dropped into a document root,
a backdoored SSH configuration, or ransomware beginning to work through a share. It
is the single most useful thing here that nothing else in my lab does.

**Configuration and vulnerability auditing.** Every agent scores its host against a
CIS hardening benchmark automatically and matches installed package versions against
CVE feeds. I get told what's misconfigured without asking.

**Investigation afterwards.** One searchable timeline across every monitored host,
instead of me SSH-ing into six machines grepping logs by hand at midnight.

There's also a reason this pairs well with the offensive side of the lab. Running
attacks and then watching them arrive as alerts is how detection engineering is
actually learned. My own tooling becomes the test traffic. Rule 5710 firing above
wasn't a demo — it was me attacking my own machine and confirming the defence saw it.

## Where it stands, honestly

Two endpoints monitored. A right-sized SIEM with a proven detection path. That is a
foundation, not a security operations centre, and I would rather say so than pretend
otherwise.

Still to do:

- **The DNS query log.** The Pi is monitored, but the agent is only shipping system
  logs. It is not yet reading the DNS filter's query log, which is the entire reason
  that machine is worth monitoring. Detecting data exfiltration over DNS needs those
  queries plus rules that understand entropy, subdomain length and volume.
- **A network sensor.** Host agents see hosts. They do not see traffic between
  devices that will never run an agent — the smart plugs, the cameras, the TV
  sticks. That needs traffic inspection fed from a mirrored switch port.
- **The Windows side.** A domain controller and a member server produce the
  authentication telemetry that matters most if you care about identity, which I do.
- **Triage.** This is the part I am most interested in. Alerts that sit in a
  dashboard nobody opens are worthless. The plan is to have the SIEM do what it is
  good at — deterministic, rule-based detection — and have a language model do
  triage on top: deduplicate, add context, judge severity, and push the handful that
  matter to my phone.

That division is deliberate. The rules decide what is an alert. The model only
explains it and decides whether it's worth waking me. I am not putting a system that
can hallucinate in charge of whether a security event is real.

## The one thing worth taking away

A security tool that is installed is not a security tool that is working. Mine had
been sitting there for two months, healthy and green and monitoring nothing at all,
and it would have gone on doing that indefinitely if I had not gone and looked.

Check what yours is actually receiving. Then attack it yourself and confirm it
noticed.
