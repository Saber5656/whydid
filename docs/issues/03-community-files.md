# Title

Add LICENSE (MIT) and community policy files

## Summary

Add the MIT license and the standard OSS policy/community files so the public
repository is complete before any release: CONTRIBUTING, SECURITY, CODE_OF_CONDUCT,
and GitHub issue/PR templates.

## Context

The repo is public from day one. Security reporting policy is part of the
security model (DESIGN §12.6). License choice (MIT) was confirmed by the product
owner (DESIGN §1.3).

## Scope

- `LICENSE` — MIT, copyright `2026 Saber5656`
- `CONTRIBUTING.md`
- `SECURITY.md`
- `CODE_OF_CONDUCT.md` — Contributor Covenant v2.1, contact = repository owner
  via GitHub
- `.github/ISSUE_TEMPLATE/bug_report.md`, `feature_request.md`,
  `.github/pull_request_template.md`

## Detailed Requirements

1. `SECURITY.md`: report privately via GitHub Security Advisories ("Report a
   vulnerability" button); acknowledgement target 14 days; supported version =
   latest release only; no bounty. Do not include an email address.
2. `CONTRIBUTING.md` must state: development requires Go (version from
   `.go-version`); run `make fmt lint test` before pushing; PRs into `main` via
   review only; design changes require updating `docs/DESIGN.md` (+ an ADR for
   architecture decisions) in the same PR; issues are drafted under
   `docs/issues/` before being opened on GitHub (repo docs are the source of
   truth, ISSUE_PLAN header).
3. Bug template asks for: whydid version, shell + version, OS, tmux y/n,
   `whydid doctor` output (mention it redacts secrets already).
4. PR template checklist: tests added, docs updated, `make fmt lint test` green,
   no new runtime dependency without an ADR (ADR-004 policy).

## Acceptance Criteria

- [ ] GitHub UI shows the license as MIT on the repo page.
- [ ] "Report a vulnerability" flow is available under the Security tab.
- [ ] New-issue view offers both templates; new-PR view shows the checklist.
- [ ] CONTRIBUTING names the docs-first rule verbatim ("update docs/DESIGN.md
      first").

## Validation

Visual verification on GitHub after merge; markdown lint (CI's misspell) passes.

## Dependencies

None.

## Non-goals

README content (issue 27); funding files; governance docs; CLA.

## Design References

DESIGN.md §1.3, §12.6; ISSUE_PLAN §2.
