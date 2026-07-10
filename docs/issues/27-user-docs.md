# Title

User documentation: README, install, privacy

## Summary

Write the public-facing documentation: a full README (English, with the
existing Japanese one-liner preserved), a privacy document mirroring ADR-005 in
user language, and the uninstall/troubleshooting guide.

## Context

Docs are part of v1 completeness (ISSUE_PLAN §1) and carry the privacy promises
publicly (trust surface for an OSS security-sensitive tool). The repo README
currently contains only the Japanese one-line concept, which must be preserved
at the top.

## Scope

- `README.md` (rewrite)
- `docs/PRIVACY.md`
- `docs/TROUBLESHOOTING.md`
- Release-QA checklist section inside `docs/RELEASING.md` (extends issue 26's
  file)

## Detailed Requirements

1. README structure (English): the existing Japanese one-liner kept as the
   first line under the title (with its English translation beside it); demo
   block (the DESIGN §2.2 console example, kept text-only); install (brew /
   tarball+checksum / `go install`); setup for zsh AND bash (`eval` lines +
   `whydid doctor`); configuration quickstart (`whydid config set llm.model …`,
   provider examples: OpenAI, Anthropic, **Ollama** with
   `base_url http://localhost:11434/v1` — model names shown as placeholders
   with a note to check the provider's current models); privacy summary box
   linking docs/PRIVACY.md (bullet the ADR-005 defaults: redaction always on,
   standard payload only, consent, `--show-payload`, no telemetry, never
   executes commands); limitations section (bash ↑-recall degradation, tmux
   needed for output capture, background jobs/E3); links to CONTRIBUTING,
   SECURITY, LICENSE.
2. `docs/PRIVACY.md`: user-language restatement of ADR-005 — exactly what is
   sent (the 8 payload fields with an example), what never leaves the machine,
   what is stored locally (path, retention 72 h, perms), redaction rule
   families with 2 masked examples, how to inspect (`--show-payload`), how to
   turn capture off, and the no-telemetry statement.
3. `docs/TROUBLESHOOTING.md`: symptom→fix table for at least: "no records
   found", "nothing to explain", consent re-prompt, `E_NO_KEY` per provider,
   insecure-endpoint refusal, tmux capture not attributed, bash double-hook
   suspicion (U2), how to fully uninstall (remove eval line, delete state dir +
   config, brew uninstall).
4. Every command/flag/path/env-var mentioned must exist in the shipped binary
   (doc-vs-`--help` review step in Validation).
5. Tone: plain, factual; no marketing superlatives; English throughout (repo
   language policy), Japanese one-liner preserved as noted.

## Acceptance Criteria

- [ ] README renders correctly on GitHub (checked via preview) and every
      install path was executed once as written (brew on macOS, tarball on
      Linux, `go install`) — recorded in the PR.
- [ ] PRIVACY.md field list matches `payload.Payload` (cross-checked against
      the issue-22 scope test's approved list — same 8 keys).
- [ ] TROUBLESHOOTING covers all listed symptoms; uninstall steps verified on a
      test machine.
- [ ] `whydid doctor`, `--show-payload`, and all documented flags exist and
      behave as described (spot-check transcript in PR).
- [ ] Japanese one-liner is intact at the top of README.

## Validation

Docs review + the recorded command transcripts; link checker (manual or CI
misspell/markdown step).

## Dependencies

16, 20, 21, 25, 26.

## Non-goals

Website; screencasts/GIFs (nice-to-have, not v1-blocking); translated full
docs (v2); man page.

## Design References

DESIGN.md §2, §11, §15, §16; ADR-005; ISSUE_PLAN §1/§6.4.
