---
title: "Software supply chain attacks beyond xz"
date: 2026-11-19T00:00:00
draft: true
tags:
  - security
---

In the [reproducible builds post](../reproducible-builds/) I wrote about the xz backdoor - years of patient social engineering to get one malicious build script into one library. It was impressive, but also unusual. Most supply chain attacks are much simpler, much faster, and target the place where modern software is weakest: dependencies. A typical JavaScript or Python project pulls in hundreds of packages, written by people you've never heard of, and updates them automatically. Every one of them is a way in.

Here's a short tour of the main techniques, with real examples, and what actually helps.

## Typosquatting

The simplest one: publish a malicious package with a name similar to a popular one, and wait for someone to make a typo in `pip install` or `npm install`. `reqeusts` instead of `requests`, `colourama` instead of `colorama` (a nice trap for British English speakers). The package usually works - it often just wraps the real one - but also runs a payload on installation, stealing credentials, SSH keys, cryptocurrency wallets or browser data.

Both PyPI and npm remove thousands of such packages every year, and security companies publish reports about new waves almost every week. It's cheap and automated, so attackers can afford a very low success rate.

## Dependency confusion

In 2021 Alex Birsan published research that made many companies nervous. Large organisations often use internal packages, with names like `company-auth-utils`, hosted on a private registry. Birsan noticed that these names leak - in public JavaScript files, error messages, GitHub repositories. He then published packages with the same names on the public npm, PyPI and RubyGems registries, with a higher version number.

Many build systems were configured to check both the private and public registry, and pick the highest version. The public one won. His (harmless) packages ran inside Apple, Microsoft, PayPal, Tesla, Shopify and dozens of other companies - earning him over 130,000 dollars in bug bounties.

It wasn't just a demonstration: in late 2022, PyTorch's nightly builds pulled a malicious `torchtriton` package from PyPI instead of PyTorch's own index, exactly this way.

## Hijacked and handed-over maintainers

Why create a new package when you can take over an existing one?

- **event-stream (2018)** - a popular npm package whose maintainer had lost interest. A volunteer offered to take over, was given publishing rights, and added a dependency containing code targeting one specific Bitcoin wallet application. No account was hacked - the access was given away voluntarily, which is what makes it similar to xz, on a smaller scale.
- **ua-parser-js (2021)** - a package with millions of weekly downloads. The maintainer's npm account was compromised, and malicious versions installing a cryptominer and a password stealer were published. They were live for a few hours.
- **The npm phishing wave (2025)** - a maintainer of `chalk`, `debug` and other basic packages fell for a convincing phishing email from a fake npm support domain. Malicious versions, intercepting cryptocurrency transactions in browsers, were published for packages with billions of weekly downloads combined. Shortly after, the **Shai-Hulud** worm went a step further: it stole npm tokens from developers' machines and used them to publish infected versions of their other packages automatically.

## Build pipelines and CI

The build system is part of the supply chain too, and a very attractive part - it has access to all the secrets.

- **Codecov (2021)** - attackers modified Codecov's Bash uploader script, which thousands of CI pipelines downloaded and ran with `curl | bash`. For two months, it sent the environment variables of every pipeline - full of credentials - to the attackers.
- **tj-actions/changed-files (2025)** - a popular GitHub Action used by tens of thousands of repositories. Attackers compromised it and **moved the existing version tags** to point to a malicious commit, which dumped the CI secrets into the build logs. Anyone using `uses: tj-actions/changed-files@v45` got the malicious version, without changing anything. Many of those build logs were public.
- **3CX (2023)** - a desktop phone app signed and shipped to hundreds of thousands of customers with a backdoor. The attackers got into 3CX through another supply chain attack: an employee had installed a trojanised version of a different company's trading software. A supply chain attack inside a supply chain attack.

## What actually helps

There's no single fix, but there are many cheap measures that would have stopped most of the attacks above.

**Lock files with hashes.** `package-lock.json`, `poetry.lock`, `uv.lock`, `Cargo.lock` record not just versions but checksums of each package. Install with `npm ci`, `pip install --require-hashes` or equivalent, so a build can't silently pick up a new version or a modified file.

**Don't update instantly.** Most malicious versions are found and removed within hours or days. Several package managers can now refuse versions newer than a given age (e.g. pnpm's `minimumReleaseAge`, uv's `exclude-newer`). A few days of delay would have avoided most of the npm incidents above. Dependabot and Renovate can also be configured to wait.

**Pin GitHub Actions by commit hash**, not by tag:

```yaml
# Tags can be moved:
- uses: actions/checkout@v4
# Commit hashes can't:
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

(Copy the real hash from the action's release page - the one above is an example of the format.) And give workflows the minimum `permissions:` they need, so a compromised step can do less damage.

**Configure private registries properly.** Use scoped packages (`@company/auth-utils`) on npm, and a single index that proxies the public one, instead of checking several and picking the highest version. That kills dependency confusion.

**Trusted publishing.** PyPI and npm can accept new releases only from a specific CI workflow, authenticated with OpenID Connect, with no long-lived tokens that can be stolen. Combined with provenance attestations (see my [post on signing software](../signing-software/)), you can check that a package was built from a specific repository, by a specific workflow.

**Mandatory 2FA** for maintainers, preferably phishing-resistant (security keys or passkeys). PyPI requires 2FA for all publishers, npm for maintainers of high-impact packages. It wouldn't have stopped event-stream or xz, but it would have stopped most account takeovers.

**SBOMs** (Software Bill of Materials, in SPDX or CycloneDX format) - a list of everything that went into your build. It doesn't prevent anything, but when the next incident is announced, you can answer "are we affected?" in minutes instead of days.

**Fewer dependencies.** The most boring advice, and the most effective. Every dependency is code you run, written by someone you trust implicitly. Before adding a package for something you could write in 20 lines, it's worth asking whether it's worth it.

## SLSA

If you want a framework for all this, **SLSA** (Supply-chain Levels for Software Artifacts, pronounced "salsa") is an OpenSSF specification describing increasing levels of build integrity: from "the build process is documented and produces provenance" up to "builds run on a hardened, isolated platform, and the provenance can't be forged by the project's own maintainers". Reproducible builds fit nicely on top of that: SLSA tells you how something was built, reproducible builds let you check it.

## Should you care?

As a developer: yes, more than about almost any other security topic. Your own code is probably a small fraction of what you ship. Lock files, a short update delay and pinned CI actions take an afternoon to set up, and protect you from the majority of the attacks described here. The xz-style long game is much harder to defend against - but it's also much rarer.
