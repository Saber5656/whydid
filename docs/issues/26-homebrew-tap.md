# Title

Homebrew tap publishing

## Summary

Extend the release pipeline to publish a Homebrew formula to the tap repository
`Saber5656/homebrew-tap` on every release, enabling
`brew install saber5656/tap/whydid`.

## Context

Confirmed distribution decision (DESIGN §1.3/§15): self-owned tap, no
homebrew-core submission in v1. The tap-push token is a credential and is
created/stored by the repository owner manually — agents only document the
steps (owner's operating rules).

## Scope

- `brews:` section in `.goreleaser.yaml`
- `docs/RELEASING.md` — release runbook including the one-time manual
  prerequisites and per-release steps

## Detailed Requirements

1. `brews:` config: formula name `whydid`; repository owner `Saber5656`, name
   `homebrew-tap`, branch `main`; directory `Formula`; homepage = repo URL;
   description = "Explains why your last shell command failed and proposes
   fixes"; license `MIT`; install stanza `bin.install "whydid"`; test stanza
   runs `whydid version`; caveats text telling the user to add
   `eval "$(whydid init zsh)"` (or bash) to their rc file.
2. Token: workflow env `HOMEBREW_TAP_GITHUB_TOKEN` read from an Actions secret
   of that name; goreleaser `brews[].repository.token` wired to it. The
   workflow must skip the brew step with a clear warning (not fail the release)
   when the secret is absent, so releases remain possible before the tap
   exists.
3. `docs/RELEASING.md` must contain, as explicit MANUAL (human-owner) steps:
   (a) create the public repo `Saber5656/homebrew-tap` with an empty `Formula/`
   directory; (b) create a fine-grained PAT scoped to ONLY that repo with
   contents read/write; (c) store it as the `HOMEBREW_TAP_GITHUB_TOKEN` secret
   in the whydid repo. Plus the per-release procedure: ensure CI green → create
   annotated tag `vX.Y.Z` → push tag → verify Release assets + tap formula →
   `brew install saber5656/tap/whydid && whydid version` smoke on macOS.
4. Formula correctness across the 4 artifacts (goreleaser generates the
   per-arch blocks automatically — verify in dry-run output).

## Acceptance Criteria

- [ ] `make release-dry` output includes a generated formula file whose sha256
      values match the dry-run archives.
- [ ] First real (or rc) release with the secret configured pushes a working
      formula: `brew install saber5656/tap/whydid` succeeds on a macOS machine
      and `whydid version` prints the tag version (recorded in PR/RELEASING).
- [ ] Release without the secret completes with the documented warning.
- [ ] `docs/RELEASING.md` marks the token/repo steps as manual-owner-only.

## Validation

Dry-run inspection + the first tagged release rehearsal (coordinated with the
owner for the manual prerequisites).

## Dependencies

25.

## Non-goals

homebrew-core; Linuxbrew-specific testing (formula is arch-generic; Linux users
primarily use tarballs/go install); autobump PRs to other package managers.

## Design References

DESIGN.md §15, §1.3; ISSUE_PLAN §4 wave 6; owner rule: secrets are handled by
humans only.
