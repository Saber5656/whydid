# Title

GoReleaser config and tag-driven release workflow

## Summary

Add `.goreleaser.yaml` and `.github/workflows/release.yml` so that pushing an
annotated `v*` tag produces GitHub Release artifacts (4 platform tarballs +
checksums) after the full test suite passes.

## Context

DESIGN §15 fixes the release mechanics; supply-chain expectations are in §12.7
(tag-driven CI builds only, checksums published). merge ≠ release: only tags
publish.

## Scope

- `.goreleaser.yaml`
- `.github/workflows/release.yml`
- `Makefile` target `release-dry` (goreleaser release --snapshot --clean)

## Detailed Requirements

1. Builds: `main: ./cmd/whydid`, `CGO_ENABLED=0`, `-trimpath`, ldflags setting
   `internal/version.{Version,Commit,Date}`; targets exactly
   darwin/arm64, darwin/amd64, linux/amd64, linux/arm64.
2. Archives: `tar.gz`, name template
   `whydid_<version>_<os>_<arch>`, containing the binary, LICENSE, README.md.
3. `checksums.txt` (sha256) generated and attached.
4. Changelog: GoReleaser's default commit-list grouped by `feat|fix|docs|chore`
   prefixes (best effort, DESIGN §15); no external changelog tooling.
5. Workflow: trigger `push: tags: ['v*']`; jobs: (a) run the identical checks as
   ci.yml (call `make fmt vet lint test` + e2e), then (b) `goreleaser release`
   with `GITHUB_TOKEN` (built-in) — permissions `contents: write` on job (b)
   only. All actions SHA-pinned.
6. The workflow must fail (and publish nothing) if any test step fails.
7. `release-dry` target lets a maintainer verify the config locally without a
   tag; document it in CONTRIBUTING (one line, this PR).
8. Do NOT include the Homebrew section yet (issue 26 adds `brews:` — keeping
   this issue independently mergeable and testable without tap credentials).
9. Version sanity: goreleaser `snapshot.name_template` includes `-dev`;
   `./whydid version` from a snapshot build shows the snapshot version.

## Acceptance Criteria

- [ ] `make release-dry` on a dev machine produces 4 archives + checksums under
      `dist/` and the binaries run (`./dist/…/whydid version`).
- [ ] A `v0.0.1-rc1` tag on a scratch fork/branch run publishes a GitHub
      (pre-)release with 4 assets + checksums.txt (verified once, then the tag
      deleted; note the verification in the PR).
- [ ] `sha256sum -c checksums.txt` passes against downloaded assets.
- [ ] Release workflow has `contents: write` only where needed; ci.yml remains
      read-only.
- [ ] Tag on a red test suite publishes nothing.

## Validation

Snapshot build in CI (add a `release-dry` step to ci.yml on ubuntu only, without
publishing) + the one-time rc-tag rehearsal.

## Dependencies

02, 16 (a releasable product exists; e2e from 23 should be green before the
first real tag — enforced by wave gating, not by this config).

## Non-goals

Homebrew tap (26); signing/SLSA (v2); homebrew-core; Docker images; nightly
builds.

## Design References

DESIGN.md §15, §12.7; ADR-004.
