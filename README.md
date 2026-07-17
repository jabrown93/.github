# jabrown93/.github

User-level defaults shared across all `jabrown93` repositories.

- **Renovate** — shared preset in [`renovate-config.json`](renovate-config.json),
  consumed via `"extends": ["github>jabrown93/.github//renovate-config.json"]`.
- **Reusable GitHub Actions workflows** — in [`.github/workflows/`](.github/workflows),
  consumed via `uses:` (see below).
- **Composite actions** — in [`.github/actions/`](.github/actions), consumed via
  `uses:` from a step (see below).

## Reusable workflows

Each workflow is `on: workflow_call`. The third-party actions inside are pinned
by commit SHA (with a `# vX.Y.Z` comment that Renovate maintains). Consumers
**also pin this repo by commit SHA** — some repos require digest-pinned `uses:`
refs — with a `# vX.Y.Z` comment so Renovate can bump them:

```yaml
uses: jabrown93/.github/.github/workflows/<name>.yml@<sha> # v1.1.0
```

Trigger events (`push`, `pull_request`, `schedule`, branch filters) and
`concurrency` stay in the **caller** — a reusable workflow cannot declare them.

### `node-build.yml` — lint + build a Node.js project

| input | default |
|---|---|
| `node-versions` | `'["20.x", "22.x", "24.x"]'` (JSON array) |
| `runs-on` | `ubuntu-latest` |

```yaml
name: Build and Lint
on: [push, pull_request]
jobs:
  build:
    uses: jabrown93/.github/.github/workflows/node-build.yml@<sha> # v1.1.0
```

### `codeql.yml` — CodeQL advanced analysis (single language)

| input | default |
|---|---|
| `language` | `javascript-typescript` |
| `build-mode` | `none` |
| `runs-on` | `ubuntu-latest` |

```yaml
name: CodeQL Advanced
on:
  push:
    branches: [main, beta]
  pull_request:
    branches: [main, beta]
  schedule:
    - cron: '25 22 * * 5'
jobs:
  analyze:
    uses: jabrown93/.github/.github/workflows/codeql.yml@<sha> # v1.1.0
    permissions:
      security-events: write
      packages: read
      actions: read
      contents: read
```

### `stale.yml` — close stale issues and PRs

Defaults match the prior per-repo stale config; every knob is overridable
(`days-before-stale`, `days-before-close`, the messages, the labels). The
schedule lives in the caller.

```yaml
name: Close stale issues and PRs
on:
  schedule:
    - cron: '30 1 * * *'
jobs:
  stale:
    uses: jabrown93/.github/.github/workflows/stale.yml@<sha> # v1.1.0
    permissions:
      actions: write
      issues: write
      pull-requests: write
```

### `npm-release.yml` — semantic-release + npm trusted publishing

| input | default |
|---|---|
| `node-version` | `'24'` |

Secrets `APP_ID` and `APP_PRIVATE_KEY` must be passed by the caller.

> npm trusted publishing matches on the **caller** repo + entry-point filename.
> Keep the caller workflow named as registered on npmjs.org (`release.yaml`),
> and verify a publish succeeds after converting.

```yaml
name: Release
on:
  push:
    branches: [main, beta]
concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false
jobs:
  release:
    uses: jabrown93/.github/.github/workflows/npm-release.yml@<sha> # v1.1.0
    permissions:
      contents: write
      issues: write
      pull-requests: write
      id-token: write
    secrets:
      APP_ID: ${{ secrets.APP_ID }}
      APP_PRIVATE_KEY: ${{ secrets.APP_PRIVATE_KEY }}
```

### `docker-release.yml` — semantic-release + Docker image publish

| input | default |
|---|---|
| `image` | *(required)* full image ref, e.g. `ghcr.io/jabrown93/crosswatch` |
| `dockerfile` | `./Dockerfile` |
| `context` | `.` |
| `platforms` | `linux/amd64,linux/arm64` |
| `build-args` | `""` (extra newline-separated args, appended after the automatic `APP_VERSION=v<version>`) |
| `extra-plugins` | `""` (extra semantic-release plugins beyond commit-analyzer/release-notes-generator/changelog/git/github) |

Secrets `APP_ID` and `APP_PRIVATE_KEY` must be passed by the caller. Two jobs
(`release` then `image`) so a build/push/sign failure never strands a
tagged-but-imageless release — re-run just the failed `image` job and it
reuses `release`'s outputs instead of re-evaluating semantic-release. The
image is checked out and built from the release tag itself, tagged `:latest`
+`:v<version>` on `main` or `:beta`+`:v<version>` on a prerelease branch, and
signed keylessly with cosign. See `jabrown93/AURA/.github/workflows/release.yml`
for a richer example (weekly dependency roll-up, rolling `:edge` tag, manual
re-publish) this was modeled on but deliberately left out of the shared
version — add those as caller-side extensions if a repo needs them.

```yaml
name: Release
on:
  push:
    branches: [main, beta]
concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false
jobs:
  release:
    uses: jabrown93/.github/.github/workflows/docker-release.yml@<sha> # v1.2.0
    permissions:
      contents: write
      packages: write
      id-token: write
    with:
      image: ghcr.io/jabrown93/<repo>
    secrets:
      APP_ID: ${{ secrets.APP_ID }}
      APP_PRIVATE_KEY: ${{ secrets.APP_PRIVATE_KEY }}
```

### `dt-sbom-upload.yml` — upload a CycloneDX SBOM to Dependency-Track

| input | default |
|---|---|
| `runs-on` | *(required)* in-cluster runner label, e.g. `arc-oss-homebridge-onkyo` |
| `artifact-name` | `sbom` (must hold `sbom.cdx.json`) |
| `project-name` | `github.com/<owner>/<repo>` from GitHub context |
| `project-version` | the caller's commit SHA |
| `is-latest` | `'true'` |

The privileged half of the Dependency-Track setup — pair it with the
[`generate-sbom`](#generate-sbom--cyclonedx-sbom-for-npm--maven--syft) composite
action, which builds the SBOM on a hosted runner and hands it over as an
artifact. Only this half touches the cluster: it runs on the caller repo's
REPO-scoped `arc-oss-<repo>` set, exchanges the run's OIDC token for the DT CI
key via OpenBao, then POSTs the SBOM.

Project identity comes from **trusted** GitHub context, never from anything the
generating job produced, so a compromised dependency can't inject values that get
executed on the cluster runner. Those contexts still resolve to the *caller's*
repo and SHA under a reusable workflow.

**Never call this from a `pull_request` event** — that would put fork-controlled
code in reach of the in-cluster runner. Use the advisory
producer/consumer split for PRs instead.

The caller repo must be in the `dt-sbom-upload` OpenBao role's
`bound_claims.repository` allowlist **and** have an `arc-oss-<repo>` runner set.
The OIDC `repository` claim is the caller's repo, so adopting this workflow needs
no role change. See `jabrown93/homelab` → `docs/dependency-track.md`.

```yaml
name: Dependency-Track SBOM
on:
  push:
    branches: [main]
  workflow_dispatch: {}
jobs:
  sbom:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@<sha> # v7.0.0
      - uses: jabrown93/.github/.github/actions/generate-sbom@<sha> # v1.4.0
        with:
          ecosystem: npm
  upload:
    needs: sbom
    uses: jabrown93/.github/.github/workflows/dt-sbom-upload.yml@<sha> # v1.4.0
    permissions:
      contents: read
      id-token: write
    with:
      runs-on: arc-oss-<repo>
```

## Composite actions

Pinned and consumed the same way as the workflows above, but referenced from a
**step** rather than a job:

```yaml
uses: jabrown93/.github/.github/actions/<name>@<sha> # v1.4.0
```

### `generate-sbom` — CycloneDX SBOM for npm / maven / syft

| input | default |
|---|---|
| `ecosystem` | *(required)* `npm`, `maven`, or `syft` (filesystem scan) |
| `sbom-path` | `sbom.cdx.json` |
| `artifact-name` | `sbom` |
| `node-version` | `'24'` (ecosystem `npm`) |
| `cyclonedx-npm-version` | `4.2.1` (ecosystem `npm`) |
| `java-version` | `'25'` (ecosystem `maven`) |
| `java-distribution` | `corretto` (ecosystem `maven`) |

Generates the SBOM and uploads it as an artifact. Used by **both** the
push-to-main `dt-sbom-upload.yml` caller and the advisory PR license check, which
need an identical SBOM and differ only in what they do with it.

**Always run this on a hosted runner.** `npm ci` and `mvn` execute dependency
lifecycle scripts and build plugins — that code is untrusted and must never touch
an in-cluster runner. The action does **not** check out the repo; the caller does,
so it controls `fetch-depth`.

Maven emits under `target/` per the pom's `cyclonedx-maven-plugin` `outputName`,
so maven callers pass `sbom-path: target/sbom.cdx.json` — the path is a property
of the caller's build config, not something the action can infer. The artifact
still arrives holding `sbom.cdx.json` at its **root** either way, because
`upload-artifact` flattens a single-file path to its basename. Consumers
therefore never parameterize the filename.

```yaml
      - uses: actions/checkout@<sha> # v7.0.0
      - uses: jabrown93/.github/.github/actions/generate-sbom@<sha> # v1.4.0
        with:
          ecosystem: maven
          sbom-path: target/sbom.cdx.json
```

## Versioning

Releases are tagged `vMAJOR.MINOR.PATCH` and cut **automatically** by
[`release.yml`](.github/workflows/release.yml) — semantic-release reads the
Conventional Commits merged to `main` and publishes the tag plus a GitHub Release
with notes. Nothing is versioned by hand. Consumers pin a SHA; Renovate proposes
the bump, and the tag is what makes the `# vX.Y.Z` comment next to that opaque
digest readable.

| commit on `main` | effect |
|---|---|
| `feat: …` | minor |
| `fix: …` / `perf: …` | patch |
| `feat!: …` or `BREAKING CHANGE:` footer | major |
| anything else (`ci:`, `docs:`, `refactor:`, `chore:`) | no release |

Breaking changes to a workflow's inputs/secrets are what bump the major — mark
them `!`.

**Dependency bumps.** The third-party action SHAs pinned inside the reusable
workflows are part of what consumers execute, so Renovate labels those bumps
`fix(deps)` (see the rule in [`renovate.json`](renovate.json)) and each one ships
as a patch. This repo's own CI-only tooling — the semantic-release
devDependencies in `package.json` and the weekly lock-file-maintenance commit —
stays `chore(deps)` and deliberately does **not** release: nothing a consumer runs
has changed.

`package.json` exists only to pin that tooling; there is no runtime JavaScript
here and nothing is published to npm.
