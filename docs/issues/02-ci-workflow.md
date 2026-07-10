# Title

CI workflow: tests, lint, and security scans

## Summary

Add `.github/workflows/ci.yml` running the full check suite on every PR and on
pushes to `main`, across ubuntu-latest and macos-latest, plus Dependabot config
for Go modules and GitHub Actions.

## Context

CI is the merge gate for all subsequent issues (ISSUE_PLAN §6.1) and implements
part of the supply-chain posture (DESIGN §12.2 T7, §12.7).

## Scope

- `.github/workflows/ci.yml`
- `.github/dependabot.yml`

## Detailed Requirements

1. Triggers: `pull_request` (all branches) and `push` to `main`.
2. Job matrix: `os: [ubuntu-latest, macos-latest]`; Go from `.go-version` /
   go.mod via `actions/setup-go` with built-in module cache.
3. Steps per OS: `make fmt` (fail on diff), `make vet`, `make lint`
   (golangci-lint pinned to a specific version), `make test` (includes `-race`).
4. Security steps (ubuntu only is acceptable): `govulncheck ./...` and
   `gosec ./...` (gosec runs via golangci-lint if enabled there; a standalone
   step is also fine — pick one, don't run it twice).
5. ubuntu runner: `sudo apt-get install -y zsh` so later shell tests (issue 23)
   run unchanged; harmless before then.
6. All third-party actions pinned to a full commit SHA (not a tag).
7. `dependabot.yml`: weekly `gomod` and `github-actions` update PRs.
8. Workflow permissions: `contents: read` at top level (least privilege).

## Acceptance Criteria

- [ ] A PR with a formatting error fails CI on the fmt step.
- [ ] CI is green on both OSes for the issue-01 skeleton.
- [ ] `govulncheck` and `gosec` steps execute and pass.
- [ ] Every `uses:` reference is SHA-pinned.
- [ ] Dependabot opens config-validating (no schema errors in the Insights tab).

## Validation

Open a scratch PR containing (a) clean code → green, (b) an intentionally
unformatted file → red on fmt. Delete the scratch PR after verification.

## Dependencies

01.

## Non-goals

Release workflow (issue 25); coverage upload services; caching beyond
setup-go defaults; branch-protection settings (repo-admin action, not code).

## Design References

DESIGN.md §12.7, §14 (CI matrix row); ISSUE_PLAN §6.
