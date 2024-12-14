Title: The state of post-quantum cryptography at the end of 2024
Date: 2024-12-14 21:00
Status: published
Category: random
Tags: security
Slug: post-quantum-2024

![Cloudflare Post-Quantum connection check]({static}/images/pqc-cloudflare.png)

The recent hype about Google Willow inspired me to recheck what's happening in the world of post-quantum cryptography.

You probably already heard the alarming news - all of our encryption is becoming useless. The reality is not as tragic. What is true is that most __asymmetric__ algorithms (those with public and private key) are very weak against quantum computers. They are widely used for two purposes:

- Most encrypted protocols (that utilize a much faster symmetric encryption for the vast majority of the traffic) randomly generate a key at the beginning of the session and use asymmetric encryption to send this key to the other party. This is called KEX (Key EXchange) or KEM (Key Encapsulation Mechanism), depending on the exact way.
- Digital signatures, for example those that confirm who's the owner of a TLS certificate or a publisher of a software package.

But the alarming articles routinely fail to mention the good news:

- We don't have a quantum computer that can break a modern encryption (although it doesn't exists, it already has a name: __CRQC__, Cryptographically-Relevant Quantum Computer), current ones are either too small or too unstable. We are getting closer, experts predict CRQC will be available in the next few years.
- Symmetric algorithms, which do almost 100% of the work, are safe. There are some quantum-based attacks that slightly reduce the effectiveness of the cyphers, but simply increasing the key length (which we had to do anyway due to the widespread use of GPU computing for cracking encryption) mitigates the risk.
- Hash algorithms (used, among other things, for storing encrypted passwords) are also safe. Again, the real danger came from GPUs, but this has already been taken care of.
- Current software can be easily modifed for post-quantum secrecy. It's modular, algorithms are changed every few years anyway, due to increase in CPU/GPU power and sometimes when the weaknesses are found.
- We already have some quantum-resistant asymmetric algorithms in use and adoption is rising very fast - according to Cloudflare, from 2% in January to 13%  in December 2024 in the traffic they see.

## Official standards

 In August 2024 NIST, (National Institute of Standards and Technology, a US government agency) published the first 3 official standards and more are coming:

- FIPS203 for key exchange.
- FIPS204 and FIPS205 for digital signatures.

Of those, at least FIPS203 was already (relatively) widely used, although not always called that. NIST doesn't develop new algortihms, it chooses from the submitted proposals. The one that became FIPS203 is mostly known as Kyber or ML-KEM.  Adversaries can (and do) harvest encrypted data now to decrypt it in future (just as you can easily crack 1990s encryption with a PC of today), therefore it's a good idea to move to post-quantum standards as soon as possible.

Securing digital signatures against the quantum computing is not as urgent. Once CRQC are available, they will be a huge threat. Quick refresher: signatures can be checked with a public key, but to make them you need a private key, which should be well guarded. Obtaining the private key from the public key is, for all practical purposes, impossible with a classical computer (as in: even if you use all the world computing power to break just one key, you'd still need more time than the life expectancy of the Sun). But it's fast with a quantum computer. However, it's not much use retroactively. You could use a CRQC to decrypt TLS traffic harvested now, that's a real threat for some use cases. But forging 2024 certificate in 2034 will be useless. We just need to switch to quantum-safe algorithms before CRQC.

## Client support - mostly done

Chrome has implemented Kyber/FIPS203 since version 115 released in August 2023. That also includes all derivative browsers such as Edge, Opera and Vivaldi. Firefox added support in January 2024. The only major browser that doesn't support it is Safari.

Messaging apps such as Signal, Whatsapp and iMessage also support at least one form of post-quantum cryptography.

## Server support - not as good

I found it pecular that server-side lags behind client-side in security standards. But Google decided to introduce Kyber early, other browsers followed and since they aggresively auto-update (for good reasons), pretty much all browsers support a quantum-resistant key encapsulation, and most client-side software uses browser components to do the connection. But, until servers get the support, new protocols won't see much use. And servers tend to run older software, also for good reasons: new algorithms mean new security risks. It would be ironic to increase security to the future theoretical threats only to decrease to the current threats - and yet it happened. Few years ago SIDH was a top contender for a quantum-resistant KEM, but it was broken in 2022. And not by a quantum computer, not even by a powerful GPU, a puny single-core CPU was enough - yikes.

So where do we stand now, in December 2024?

- OpenSSL, the library responsible for something like 50% of the TLS support [citation needed], has a policy of only supporting official standards. They are now working on adding FIPS203, but current versions are missing it. However, OpenSSL can be extended with "providers" - external libriaries that add additional crypto algorithms.
- Go language, probably the second most common technology in the server world, has it's own crypto libraries and you guessed it: official versions don't support Kyber.
- OpenSSH supports FIPS203, but only in the latest version 9.9, released in September 2024. It is, of course, too new to be included in mainstream Linux distros (Kali is a notable exception, though I'm stretching the definition of mainstream).

## Cloudflare to the rescue

Cloudflare is one of the early adopters. The company provides many services, including TLS termination. Incidentally, I use it for my photo gallery. Nice to know that my completely insignificant website that only contains 100% public information is well protected against state-level adversaries of the future! The also publish [stats and articles about the topic](https://blog.cloudflare.com/tag/post-quantum/).

## Open Quantum Safe

Open Quantum Safe or OQS is a research project of the Linux Foundation. One of their works is liboqs, a C library providing a large number of quantum-resistant algorithms, plus integrations into current software, such as OpenSSL or OpenSSH. If you want to get started with it, the fastest way is compiling a provider for OpenSSL. It is available on [the GitHub repo](https://github.com/open-quantum-safe/oqs-provider).

I checked a few recent Linux distro and none had a liboqs package. Why, if it's free, opensource, and comes from a reputable source? One of the reasons might be, ironically, security. The goal of the project is to help with research and software testing, it's not intended for production use. The library contains numerous algorithms, not only the official ones or top contenders, but also those that aren't well tested, or even those that were rejected. Which means a large attack surface. In fact, there are known security vulnerabilities in liboqs. If you want to help with testing, then go ahead. If you want to have a full-featured client, be careful. But if you want to protect your server - don't do it, you will be introducing more vulnerabilities than you're fixing.

## How to check common software for post-quantum support

- Browser: go to [Cloudflare Research Post-Quantum](https://pq.cloudflareresearch.com/)
- SSH client: type `ssh -Q kex` and look for a line starting with mlkem
- SSH server: connect with `-vv` option and check "peer server KEXINIT proposal" (as a bonus "local client KEXINIT proposal" will show what your client supports). Alternatively, `nmap --script ssh2-enum-algos -sV -p 22 your.ssh.host`. In any case, look for mlkem.
- OpenSSL: `openssl list -kem-algorithms` and look for mlkem or kyber.

![SSH client supporting FIPS203 (mlkem)]({static}/images/pqc-ssh.png)
