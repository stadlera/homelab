# ADR-0001 — Pin GitHub Actions to SHA digests

* Status: accepted
* Date: 2026-05-24
* Issue: #4

## Context and Problem Statement

GitHub Actions workflows reference external actions via mutable tags (e.g. `actions/checkout@v4`). A tag can be silently moved to point to different code, making the build supply chain vulnerable to tag-hijacking. A decision is needed on whether to pin to immutable SHA digests instead.

## Decision Drivers

* This repository serves as a DevOps portfolio reference — security posture should reflect best practices
* Renovate is being adopted (issue #4) and can automate digest bump PRs, removing the toil argument against pinning
* Blast radius of a compromised action running in CI is high (code checkout, secret access, push rights)

## Considered Options

* **Option A** — Pin all `uses:` references to full SHA digests, managed by Renovate
* **Option B** — Use mutable version tags (`@v4`), rely on upstream maintainer integrity
* **Option C** — Pin major-version tags only (`@v4`), a middle ground with no automation path

## Decision Outcome

Chosen option: **Option A**, because Renovate eliminates the maintenance cost and the security benefit is unambiguous. SHA digests are immutable; a compromised upstream tag cannot affect pinned workflows.

### Consequences

* Good, because CI supply chain is protected against tag-hijacking attacks
* Good, because Renovate digest bumps are automerged (zero toil once configured)
* Bad, because diffs show opaque SHA strings rather than human-readable version tags — reviewers should check the Renovate PR description for the resolved version name

### Confirmation

All `uses:` entries in `.github/workflows/` must reference a full 40-character SHA with the version tag as an inline comment, e.g.:

```yaml
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

Renovate enforces this via `pinDigests: true` in `renovate.json` and will open PRs to pin any unpinned action or bump stale digests.

## Pros and Cons of the Options

### Option A — SHA digest pinning

* Good, because builds are fully reproducible and tamper-evident
* Good, because Renovate automates all updates
* Bad, because SHA strings are opaque without the inline comment convention

### Option B — Mutable version tags

* Good, because diffs remain readable
* Bad, because a compromised upstream release can silently alter CI behaviour
* Bad, because there is no automated update path — tags drift unnoticed

### Option C — Major-version tags only

* Neutral, because it prevents accidental minor/patch breakage
* Bad, because major tags are still mutable and offer no tamper protection
* Bad, because there is still no automated update path for digest-level changes

## More Information

* [StepSecurity — Why you should pin GitHub Actions to a commit SHA](https://www.stepsecurity.io/blog/github-actions-security-best-practices)
* [GitHub docs — Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions#using-third-party-actions)
* Renovate `pinDigests` option: https://docs.renovatebot.com/configuration-options/#pindigests
