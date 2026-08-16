---
title: 'Post-quantum SSH in my home lab'
description: 'Why I turned on a hybrid post-quantum key exchange for SSH between my lab machines, what "harvest now, decrypt later" actually means, and the one config file that did it.'
pubDate: 'Jul 13 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

There is a quiet attack that does not need a quantum computer today. It only needs one to exist eventually. It is called "harvest now, decrypt later," and it is the reason I spent an evening making the SSH between my lab machines resistant to a machine that has not been built yet.

## The threat: patient attackers and recorded traffic

When you open an SSH session, the two ends first agree on a shared secret using a key exchange. Almost every connection you have ever made used an elliptic-curve exchange like X25519. It is fast, it is trusted, and against today's computers it is effectively unbreakable.

The problem is not today. A large enough quantum computer running Shor's algorithm would tear through that elliptic-curve math and recover the shared secret from a recorded session. That computer does not exist yet in any practical form. But an adversary does not have to wait for it to start collecting. They can capture your encrypted traffic now, sit on it for years, and decrypt it the moment the hardware catches up.

So the honest question is not "can someone break this today?" It is "will anything I send today still matter in ten or fifteen years?" For a lot of traffic the answer is no. For credentials, keys, personal data, and anything you would not want read aloud a decade from now, the answer is yes. Encryption is supposed to protect the future, not just the present, and a recording made today is a decryption problem that stays open until you close it.

The catch is that you cannot fix a recorded session after the fact. If it was captured under a classical key exchange, it stays vulnerable forever. The only defence is to change the key exchange before the recording is made. That is why post-quantum key exchange matters right now, even though the threat is years out.

## The fix: a hybrid key exchange

I switched my SSH server to prefer a hybrid post-quantum key exchange called `sntrup761x25519-sha512`. The word that matters is hybrid. It runs two exchanges at once and combines them: the classical X25519 curve, plus Streamlined NTRU Prime, a lattice-based algorithm designed to resist quantum attacks.

Combining them is the safe move. If the post-quantum piece turns out to have a flaw, the classical X25519 half still protects you, exactly as it does today. If a quantum computer eventually breaks X25519, the NTRU Prime half holds the line. You only lose if both fail, which is a far better position than betting everything on one. This algorithm has shipped in OpenSSH for years and is enabled by default in recent versions, so it is a mature choice rather than an experiment.

## What I actually changed

The change itself is small and reversible, which is how a lab change should be. I dropped a single config file into the SSH server's configuration directory that sets the key-exchange preference order. The hybrid post-quantum exchange goes first, followed by the classical curve algorithms as fallback. Keeping the classical options in the list matters: it means a client that does not understand the new exchange can still connect, so there is no risk of locking myself out of my own box.

Before reloading anything, I validated the config with the built-in syntax check so a typo could not take the service down. Then I reloaded, opened a fresh session, and confirmed from the connection details that it had genuinely negotiated the post-quantum exchange rather than silently falling back. That last step is the one people skip. Setting a preference is not the same as using it, and the only way to know is to look at what an actual connection negotiated. Undoing it is just as simple: delete the file and reload.

## Where this goes next

This is one link in the chain, not the whole chain. The obvious next step is moving to an ML-KEM based exchange once all my machines run an OpenSSH new enough to speak it, since ML-KEM is the algorithm NIST has actually standardised. Beyond SSH, the VPN mesh tying my lab together still rides on a classical curve, so a post-quantum sidecar for that is on the list too.

None of this is exotic. It is a few lines of config and a habit of thinking about what your traffic is worth years from now, not just today. That habit, more than any single algorithm, is the point.
