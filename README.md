# jabrown93/.github

User-level defaults shared across all `jabrown93` repositories: issue/PR
templates, community health files ([`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md),
[`CONTRIBUTING.md`](CONTRIBUTING.md), [`SECURITY.md`](SECURITY.md)), and the
shared Renovate preset.

- **Renovate** — shared preset in [`renovate-config.json`](renovate-config.json),
  consumed via `"extends": ["github>jabrown93/.github//renovate-config.json"]`.

## CI library

The reusable GitHub Actions workflows and composite actions that used to live
here (`node-build`, `codeql`, `stale`, `npm-release`, `docker-release`,
`dt-sbom-upload`, `generate-sbom`) have moved to
[`jabrown93/ci`](https://github.com/jabrown93/ci). That repo versions each
component independently (release-please, `<component>-vX.Y.Z` tags) instead of
one shared stream — see its README for usage and pinning.

This repo has no reusable workflows, no composite actions, and no release
automation of its own: nothing here is versioned, and there is no runtime
code.
