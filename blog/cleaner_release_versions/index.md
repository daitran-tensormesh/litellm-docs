---
slug: cleaner-release-versions
title: "Cleaner LiteLLM release versions: standard PEP 440 + SemVer 2.0"
date: 2026-04-28
authors:
  - litellm
description: "Pre-releases follow PEP 440. Docker tags follow SemVer 2.0. New :latest, :rc, :dev floating tags. New `litellm-dev` package family for testing branches before merge."
tags: [release, packaging, docker, helm]
hide_table_of_contents: false
---

We've heard a few recurring pain points from users trying to install or pin LiteLLM in different environments:

- **"`pip install --pre litellm` doesn't pick up anything new"** — pre-releases existed but didn't follow PEP 440 conventions, so pip's resolver couldn't reliably surface them.
- **"Helm chart pull fails with `invalid semantic version`"** — chart versions had a `v` prefix and a `-stable` suffix, which strict SemVer 2.0 parsers (Helm v3 included) reject.
- **"`docker pull ghcr.io/berriai/litellm:latest` doesn't work"** — `:latest` wasn't being maintained; you had to know the specific tag for whatever release you wanted.
- **"How do I install a build from a feature branch to test it before merge?"** — there wasn't a clean answer; you had to clone and build locally.
- **"Image scanners and SemVer-aware tooling choke on `v1.83.0-stable`"** — version strings with a `v` prefix and free-form suffix don't classify as valid versions in most ecosystems.

Starting with `1.84.0`, LiteLLM standardizes on **PEP 440** (PyPI) and strict **SemVer 2.0** (Docker + Helm). Old tags stay where they are; new releases use the new naming.

<!-- truncate -->

## What changes

### PyPI: PEP 440 versions

| Channel | Example | How to install |
|---|---|---|
| Stable | `1.84.0` | `pip install litellm` |
| Release candidate | `1.84.0rc1` | `pip install --pre 'litellm>=1.84.0rc1,<1.85'` |
| Dev / nightly | `1.84.0.dev47` | `pip install --pre litellm` |

`pip install litellm` (no `--pre`) always resolves to the latest stable. `--pre` opens up RCs and dev builds. PEP 440 ordering puts `.devN < rcN < final`, so dependency resolution behaves correctly across the channels:

```bash
# Stable only
pip install litellm

# Latest pre-release (will pick the highest of rc / dev / stable available)
pip install --pre litellm

# Pin to a specific RC for testing the next release
pip install --pre 'litellm>=1.84.0rc1,<1.85.0'
```

The lockstepped `litellm-proxy-extras` and `litellm-enterprise` packages move with each `litellm` stable release, with their own version namespaces (e.g. `litellm-proxy-extras==0.4.x`).

### Docker: SemVer 2.0 tags (no `v` prefix)

| Channel | Content tag (immutable) | Floating tag |
|---|---|---|
| Stable | `1.84.0` | `:latest` |
| Release candidate | `1.84.0-rc.1` | `:rc` |
| Dev / nightly | `1.84.0-dev.47` | `:dev` |

Content tags are **immutable** — `1.84.0` always means that exact image. Floating tags advance with each new release in their channel:

```bash
# Always the latest stable
docker pull ghcr.io/berriai/litellm:latest

# Always the most recent release candidate
docker pull ghcr.io/berriai/litellm:rc

# Always the most recent dev / nightly build
docker pull ghcr.io/berriai/litellm:dev

# Or pin to an exact version (recommended for production)
docker pull ghcr.io/berriai/litellm:1.84.0
```

Same shape for the database and non-root variants:

```bash
docker pull ghcr.io/berriai/litellm-database:1.84.0
docker pull ghcr.io/berriai/litellm-non_root:1.84.0
```

And on Docker Hub:

```bash
docker pull docker.io/litellm/litellm:1.84.0
docker pull docker.io/litellm/litellm-database:1.84.0
docker pull docker.io/litellm/litellm-non_root:1.84.0
```

### Helm: same shape as Docker

Chart versions use the same SemVer 2.0 shape as Docker tags. Strict SemVer parsers — including Helm v3 — accept these without workarounds:

```bash
helm pull oci://ghcr.io/berriai/litellm-helm --version 1.84.0
helm pull oci://ghcr.io/berriai/litellm-helm --version 1.84.0-rc.1
```

The `appVersion` in `Chart.yaml` matches the Docker image tag a deploy will pull, so what you `helm pull` and what gets deployed are aligned.

## New: ad-hoc / branch builds (`litellm-dev`)

We've added a separate **`litellm-dev`** PyPI package and **`*-dev`** Docker image family for testing pre-merge code. When a fix or new feature on a branch needs validation before it's merged into the main release line, the build can be published to:

```bash
# PyPI (testing only, not for production)
pip install --pre litellm-dev

# Docker
docker pull ghcr.io/berriai/litellm-dev:1.84.0-dev.3
docker pull ghcr.io/berriai/litellm-database-dev:1.84.0-dev.3
docker pull ghcr.io/berriai/litellm-non_root-dev:1.84.0-dev.3
```

These are intentionally separate from the production `litellm` package and standard Docker repos. Anything you have pinned to `litellm` or `ghcr.io/berriai/litellm` will never silently pick up an in-flight branch build. **`litellm-dev` is for testing only.**

## Backwards compatibility

The old `v1.83.x-stable` tags and Helm `v1.83.x-stable` charts stay on their registries and PyPI **forever**. PyPI and OCI registry tag immutability mean you can keep pinning to anything you currently use.

If a maintenance patch lands on a pre-cutover release line (e.g. a fix on `1.83.x`), that patch may continue to use the old naming convention for consistency within the line — release notes will call out the format used.

## Pinning recommendations

| Use case | Recommendation |
|---|---|
| Production, deterministic deploys | Pin to an exact stable version: `litellm==1.84.0` / `:1.84.0` |
| Track the latest stable | `pip install litellm` / `docker pull ghcr.io/berriai/litellm:latest` |
| Test the next release | `pip install --pre 'litellm>=1.85.0rc1'` / `docker pull ghcr.io/berriai/litellm:rc` |
| Run nightlies | `pip install --pre litellm` / `docker pull ghcr.io/berriai/litellm:dev` |
| Test a branch / pre-merge change | `pip install --pre litellm-dev` (we'll point you at the version when you need it) |

`:latest` is useful for getting started but **not recommended for production** — pin to a content tag so your deployments are reproducible.

## What this means for verification

[Image signing (cosign) and SBOM attestation](https://docs.litellm.ai/blog/ci-cd-v2-improvements#verify-docker-image-signatures) continue as before. Verify with the published cosign public key:

```bash
cosign verify --key cosign.pub ghcr.io/berriai/litellm:1.84.0
cosign verify --key cosign.pub ghcr.io/berriai/litellm:1.84.0-rc.1
```

The new tag shape is friendly to the rest of the ecosystem — image scanners (Trivy, Grype), Helm clients, version-comparison libraries, and dependency UIs all classify these as valid SemVer / PEP 440 versions and order them correctly.

## tl;dr

- **Stable**: `pip install litellm` and `docker pull ghcr.io/berriai/litellm:latest` now work the way you'd expect.
- **Pre-releases**: opt in with `--pre` for PyPI or `:rc` / `:dev` floating tags on Docker.
- **Branches**: install ad-hoc builds from `litellm-dev` — never accidentally pulled by production deploys.
- **Existing pins**: nothing breaks. Old tags stay forever.
