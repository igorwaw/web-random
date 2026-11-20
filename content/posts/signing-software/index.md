---
title: "Signing your software in 2026: GPG, SSH signatures and Sigstore"
date: 2026-11-19T00:00:00
draft: true
tags:
  - security
  - linux
---

In the [reproducible builds post](../reproducible-builds/) I covered how to check that a binary was built from the source it claims. That leaves the other half of the question: who released that source (and the binary) in the first place? That's what signatures are for. A valid signature proves that the file was signed by the holder of a particular private key and hasn't been modified since.

The cryptography is the easy part. The hard part is knowing which key is the right one - and keeping that key safe. The three common approaches deal with that in very different ways.

## GPG - the classic

OpenPGP (with GnuPG as the main implementation) has been the standard for signing open source releases since the 1990s. Distributions sign their package repositories with it, and many projects publish `.asc` or `.sig` files next to their tarballs.

```bash
$ gpg --armor --detach-sign release.tar.gz
$ gpg --verify release.tar.gz.asc release.tar.gz
gpg: Signature made Thu Nov 19 23:01:01 2026 CET
gpg:                using EDDSA key C7A1EDB199C81005A757B5211802A9877FD2E507
gpg: Good signature from "Example Signer <me@example.com>" [ultimate]
```

The technical part works fine. The trust part is where it falls apart.

PGP's answer to "is this the right key?" was the **web of trust**: people meet in person, check each other's IDs and sign each other's keys. If you trust Alice, and Alice signed Bob's key, you can trust Bob's key a bit. Key signing parties were a thing at conferences. In practice, almost nobody outside a small circle of developers ever took part, and the web of trust never scaled.

The keyserver network didn't age well either. In 2019 attackers flooded the keys of some well-known developers with tens of thousands of signatures, making them unusable for anyone who downloaded them - and the SKS keyservers, by design, couldn't delete anything. Today `keys.openpgp.org` works differently (verified email addresses, no third-party signatures), but the old network is gone.

So in practice, verifying a GPG signature means getting the key from the project's website, over HTTPS - which is only as trustworthy as the website. Still useful: if the download mirror is compromised but the website isn't, the signature catches it. And for distributions, the trust problem doesn't exist - the repository key comes with the OS installation.

Then there's the usability. GPG's command line and its concepts (subkeys, trust levels, expiry, revocation certificates) have a reputation for a reason. Signing release files is one of the easier uses, but it still takes a while to set up properly.

## SSH signatures - the key you already have

Since OpenSSH 8.0 (2019), `ssh-keygen` can sign arbitrary files. Most developers already have an SSH key, already protect it, and have already uploaded the public key to GitHub or GitLab.

```bash
$ ssh-keygen -Y sign -f ~/.ssh/id_ed25519 -n file release.tar.gz
Signing file release.tar.gz
Write signature to release.tar.gz.sig
```

Verification needs an `allowed_signers` file, mapping identities to public keys:

```bash
$ echo "me@example.com $(cat ~/.ssh/id_ed25519.pub)" > allowed_signers
$ ssh-keygen -Y verify -f allowed_signers -I me@example.com -n file -s release.tar.gz.sig < release.tar.gz
Good "file" signature for me@example.com with ED25519 key SHA256:XS7mTuo20H9BeiW1uNa7N5tqsuIG0ryxILXQ3NjBnL4
```

And after modifying a single byte of the file:

```
Signature verification failed: incorrect signature
Could not verify signature.
```

The `-n file` part is a **namespace**. A signature made for one purpose (e.g. `git`) won't verify for another (e.g. `file`), so a signature on a commit can't be reused to "sign" something else.

Git supports SSH signing too, since version 2.34:

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

GitHub and GitLab show such commits as "Verified" if the same key is uploaded to your account as a signing key. For me, that's the sweet spot for individual developers: no new tools, no new keys, nothing to learn. The trust question is the same as with GPG (where do you get the public key from?), but there's no pretence of a web of trust - you trust a key because it's listed on the person's GitHub profile or project website.

For release files, there are also **minisign** and OpenBSD's **signify** - small tools that do just this one thing, with tiny keys you can put in a README. OpenBSD has been signing its releases with signify since 2014.

## Sigstore - no keys at all

Sigstore, a Linux Foundation project launched in 2021, takes a radically different approach: **keyless signing**. Nobody manages a long-term private key.

1. You authenticate with an identity provider you already use (GitHub, Google, Microsoft) via OpenID Connect - or, in CI, the pipeline's own identity.
2. A certificate authority called **Fulcio** issues a certificate for a fresh, ephemeral key, valid for just ten minutes and bound to your identity (an email address, or a CI workflow).
3. You sign the artefact with that key, and the key is thrown away.
4. The signature and certificate are recorded in **Rekor**, a public, append-only transparency log - so the signing event can be verified later, long after the certificate expired.

The tool is `cosign`:

```bash
cosign sign-blob release.tar.gz --bundle release.tar.gz.sigstore.json
cosign verify-blob release.tar.gz --bundle release.tar.gz.sigstore.json \
    --certificate-identity me@example.com \
    --certificate-oidc-issuer https://github.com/login/oauth
```

Signing opens a browser window to log in. Verification checks the whole chain: the certificate was issued by Fulcio, for that identity, by that identity provider, and the signing was logged in Rekor.

What's verified changes too. Instead of "this was signed by key `C7A1EDB1...`" you check "this was signed by `me@example.com`", or even "this was built by the release workflow in the `example/project` repository on GitHub". That last part is where Sigstore really shines - it can tie an artefact to the exact CI pipeline that produced it, not just to a person.

And it's spreading fast. Kubernetes signs its releases with it. npm can publish packages with provenance, and PyPI added attestations - both built on Sigstore. GitHub's artifact attestations use it under the hood.

The downsides: you depend on Sigstore's infrastructure (or have to run your own), on your identity provider's security (a compromised GitHub account can sign as you), and everything you sign is public in the transparency log - by design, but worth knowing.

## Which one should you use?

| | GPG | SSH | Sigstore |
| --- | --- | --- | --- |
| Long-term key to protect | Yes | Yes (already have it) | No |
| How you trust the signer | Key fingerprint, web of trust in theory | Key fingerprint | Identity (email, CI workflow) |
| Extra tools | gpg | None | cosign |
| Works offline | Yes | Yes | No |
| Public record of signatures | No | No | Yes, by design |
| Best for | Distro repositories, legacy | Git commits, small projects | CI-built releases, containers |

My take: sign your commits with SSH, because it costs nothing. If you publish releases from CI, Sigstore is the modern answer - especially since npm and PyPI integrate it directly. GPG isn't going anywhere, but unless a distribution or ecosystem requires it, I wouldn't start a new project with it.

And remember that a signature only answers "who". Whether that person (or pipeline) should be trusted, and whether the code is any good, is still up to you.
