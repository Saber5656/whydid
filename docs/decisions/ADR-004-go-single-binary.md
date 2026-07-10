# ADR-004: Implementation in Go as a Single Static Binary

Date: 2026-07-08
Status: Accepted (confirmed with product owner, 2026-07-08)

## Context

Candidates: Go, Rust, TypeScript/Node, Python. Decisive constraints:

- The hook path (`whydid hook record`) runs at **every prompt**; its startup +
  runtime budget is single-digit milliseconds or users will feel their shell lag.
- Distribution must be a single artifact installable via Homebrew tap and GitHub
  Releases with no runtime dependency (rules out Node/Python for v1 ergonomics).
- Implementation will be executed by lower-capability agents from written issues;
  a small language surface and a strong standard library reduce guessing.

## Decision

- Implement whydid in **Go** (minimum version pinned in issue 01; latest stable at
  scaffold time), released as statically-linked single binaries for
  `darwin/arm64`, `darwin/amd64`, `linux/amd64`, `linux/arm64`.
- Dependency policy: standard library first. Allowed third-party runtime
  dependencies in v1: a TOML parser (`github.com/BurntSushi/toml`). Test-only
  dependency allowed: a PTY helper (`github.com/creack/pty`) for shell
  integration tests. Anything else requires a new ADR.
- Shell assets (zsh/bash snippets, vendored bash-preexec) are embedded with
  `go:embed` so the binary remains the only artifact.
- CLI uses the standard `flag` package with a hand-rolled subcommand dispatcher
  (documented in DESIGN.md §11) instead of cobra, keeping the dependency tree and
  startup cost minimal.

## Consequences

- `whydid hook record` startup (~a few ms) is inside the per-prompt budget;
  Node/Python would not have been.
- Cross-compilation + GoReleaser gives the four release targets from one CI job.
- No cobra/viper conveniences; the CLI surface is deliberately small (7
  subcommands) so the hand-rolled dispatcher stays trivial and testable.
- Contributors need Go, which is common in the CLI-tool ecosystem.
