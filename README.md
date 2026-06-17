# jabrown93/.github

User-level defaults shared across all `jabrown93` repositories.

- **Renovate** — shared preset in [`renovate-config.json`](renovate-config.json),
  consumed via `"extends": ["github>jabrown93/.github//renovate-config.json"]`.
- **Reusable GitHub Actions workflows** — in [`.github/workflows/`](.github/workflows),
  consumed via `uses:` (see below).

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

## Versioning

Releases are tagged `vMAJOR.MINOR.PATCH`. Breaking changes to a workflow's
inputs/secrets bump the major. Consumers pin a SHA; Renovate proposes the bump.
