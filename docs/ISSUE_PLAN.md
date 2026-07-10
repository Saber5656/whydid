# whydid — v1 Issue Plan

Status: Approved (derived from docs/DESIGN.md; GitHub Issues are generated from
`docs/issues/*.md` and are stale derived artifacts whenever they disagree with
these files).

## 1. v1 completion statement

**When every issue listed in §2 is completed and its Validation section passes,
whydid v1 is complete**: a single static Go binary, installable via the Homebrew
tap / GitHub Releases / `go install`, that (a) records command metadata through
zsh and bash hooks with fail-silent behavior, (b) opportunistically captures real
error output inside tmux, (c) explains the most recent failure via a
user-configured OpenAI-compatible or Anthropic endpoint under the ADR-005 privacy
regime (mandatory redaction, standard payload, first-run consent,
`--show-payload`, zero telemetry), (d) delivers a selected fix by prompt
insertion without ever executing commands (INV-1/INV-2), and (e) ships with the
full test pyramid (unit, provider integration, PTY e2e, security suite) green on
the ubuntu+macos CI matrix and a tag-driven release pipeline. The only remaining
work would arise from the known unknowns in §8 or newly discovered
implementation facts — such discoveries must be filed as new issues and, when
they change behavior, folded back into DESIGN.md first.

## 2. Issues in recommended execution order

| Order | File | Title | Wave |
|---|---|---|---|
| 1 | issues/01-go-scaffold.md | Scaffold Go module, repository layout, and Makefile | 0 |
| 2 | issues/03-community-files.md | Add LICENSE (MIT) and community policy files | 0 |
| 3 | issues/02-ci-workflow.md | CI workflow: tests, lint, and security scans | 0 |
| 4 | issues/04-config-package.md | Config package: TOML schema, defaults, validation, paths | 1 |
| 5 | issues/05-record-store.md | Record store: JSONL session files (append/read/compact/GC) | 1 |
| 6 | issues/06-text-hygiene.md | Text hygiene: redaction engine and ANSI sanitizer | 1 |
| 7 | issues/07-hook-record-command.md | `whydid hook record`: fail-silent ingest path | 2 |
| 8 | issues/08-zsh-integration.md | `whydid init zsh`: hooks, session id, wrapper function | 2 |
| 9 | issues/09-bash-integration.md | `whydid init bash`: vendored bash-preexec integration | 2 |
| 10 | issues/10-tmux-capture.md | tmux OutputProvider: on-demand error-output capture | 2 |
| 11 | issues/11-llm-core.md | LLM core: provider interface, HTTP policy, key resolution | 3 |
| 12 | issues/12-openai-provider.md | OpenAI-compatible provider backend | 3 |
| 13 | issues/13-anthropic-provider.md | Anthropic Messages provider backend | 3 |
| 14 | issues/14-payload-prompt.md | Payload builder and prompt template | 3 |
| 15 | issues/15-response-parser.md | Response parser, validator, and danger tagging | 3 |
| 16 | issues/17-renderer.md | Renderer: explanation/fixes UI on stderr | 4 |
| 17 | issues/18-fix-selection-stdout.md | Fix selection loop and the stdout contract (INV-2) | 4 |
| 18 | issues/19-consent-show-payload.md | First-run consent flow and `--show-payload` | 4 |
| 19 | issues/20-config-command.md | `whydid config` subcommand | 4 |
| 20 | issues/16-explain-command.md | Explain command orchestration (state machine S0–S10) | 4 |
| 21 | issues/21-doctor-command.md | `whydid doctor` and session GC | 4 |
| 22 | issues/22-security-test-suite.md | Security test suite (T1–T9 regression net) | 5 |
| 23 | issues/23-shell-e2e-harness.md | Shell end-to-end PTY test harness | 5 |
| 24 | issues/24-hardening-audit.md | Hardening audit and error-path polish | 5 |
| 25 | issues/25-goreleaser-release.md | GoReleaser config and tag-driven release workflow | 6 |
| 26 | issues/26-homebrew-tap.md | Homebrew tap publishing | 6 |
| 27 | issues/27-user-docs.md | User documentation: README, install, privacy | 6 |

## 3. Dependency table

| Issue | Blocked by | Enables |
|---|---|---|
| 01 | — | everything |
| 02 | 01 | merge gate for all later PRs |
| 03 | — | public-repo hygiene |
| 04 | 01 | 07,10,11,14,19,20,21 |
| 05 | 01 | 07,10,14,16,21 |
| 06 | 01 | 07,10,14,15,17,22 |
| 07 | 05,06 | 08,09,23 |
| 08 | 07 | 09,21,23 |
| 09 | 07,08 | 21,23 |
| 10 | 04,05,06 | 14,16,17 |
| 11 | 04 | 12,13,16,21 |
| 12 | 11 | 16,23 |
| 13 | 11 | 16 |
| 14 | 04,05,06,10 | 15(prompt/schema pairing),16,19 |
| 15 | 06,14 | 16,17 |
| 16 | 10,11,12,13,14,15,17,18,19 | 21(GC call),22,23,24,25 |
| 17 | 06,10,15 | 16,18 |
| 18 | 17 | 16,23 |
| 19 | 04,14 | 16 |
| 20 | 04 | 27 |
| 21 | 04,05,08,09,11 | 24,27 |
| 22 | 06,10,15,16,17,18 | release gate |
| 23 | 08,09,12,16,18 | release gate |
| 24 | 16,21 | release gate |
| 25 | 02,16 | 26,27 |
| 26 | 25 | 27 |
| 27 | 16,20,21,25,26 | v1 done |

## 4. Implementation waves

| Wave | Theme | Issues | Exit criterion |
|---|---|---|---|
| 0 | Repo foundation | 01,02,03 | CI green on a hello-world binary; MIT license present |
| 1 | Core data layer | 04,05,06 | config/store/redact/sanitize packages ≥ 90% covered, golden corpora in place |
| 2 | Shell integration | 07,08,09,10 | manual smoke: failures recorded in zsh+bash; tmux capture returns attributed text |
| 3 | LLM engine | 11,12,13,14,15 | fake-server integration tests pass for both providers; parser survives adversarial corpus |
| 4 | CLI & UX | 17,18,19,20,16,21 | full explain loop works end-to-end against a fake provider on a dev machine |
| 5 | Verification | 22,23,24 | security suite + PTY e2e green in CI matrix |
| 6 | Release | 25,26,27 | `v0.1.0` tag produces installable brew/tarball artifacts; docs complete |

Wave N may start when its issues' listed dependencies are complete — full
completion of wave N−1 is not required (see §3).

## 5. Coverage table (DESIGN.md → issues)

| DESIGN.md section | Issue(s) |
|---|---|
| §2 UX flows & degraded modes | 16,17,18,19,27 |
| §4 repository layout | 01 |
| §5.1 session identity | 08,09 |
| §5.2 zsh snippet | 08 |
| §5.3 bash snippet | 09 |
| §5.4–5.5 transport + hook record | 07 |
| §6 record store | 05 |
| §7 redaction | 06 (rules), 22 (corpus regression) |
| §8 tmux capture | 10 |
| §9.1 state machine + exit codes | 16 |
| §9.2 stdout contract | 18 (+23 e2e proof) |
| §9.3 rendering/selection | 17,18 |
| §9.4 non-interactive | 16 |
| §10.1 provider interface/keys | 11 |
| §10.2 payload | 14 |
| §10.3 prompt template | 14 |
| §10.4 parse/validate/danger | 15 |
| §10.5.1 openai backend | 12 |
| §10.5.2 anthropic backend | 13 |
| §10.6 HTTP policy/errors | 11 (+12,13 conformance) |
| §11.1 CLI dispatcher | 01 (skeleton), 16/19/20/21 (commands) |
| §11.2 env vars | 04,07,16 |
| §11.3 config schema | 04,20 |
| §11.4 key handling | 11,24 |
| §11.5 doctor | 21 |
| §12 security model (T1–T9, INV-1..5) | 22 (suite), plus per-threat issues named in the threat table |
| §13 edge cases E1–E15 | 05,07,08,09,10,15,16 (each issue lists its E-cases) |
| §14 testing strategy | 02,22,23 + per-issue tests |
| §15 release engineering | 25,26 |
| §16 scope boundaries | this file §7–§8 |

Every DESIGN section is owned by at least one issue; no product behavior exists
only in prose outside this mapping.

## 6. Validation strategy (whole product)

1. **Per-issue**: each issue ships unit tests in the same PR; its Validation
   section is the PR review checklist. CI (issue 02) is the merge gate.
2. **Wave gates**: the exit criteria in §4 are verified before starting dependent
   work that would build on unproven behavior.
3. **Cross-cutting invariants** (INV-1..INV-5) get dedicated regression tests in
   issue 22 and are re-proven at the system level by the PTY e2e harness
   (issue 23), which exercises the real zsh/bash → binary → fake-LLM →
   buffer-insertion loop.
4. **Release gate**: wave 5 fully green + manual QA checklist (issue 27) on one
   macOS and one Linux machine, including an Ollama-endpoint smoke test.
5. **No live-LLM tests in CI** — determinism and secrecy; live smoke is manual.

## 7. Deferred to v2 (do not implement in v1 issues)

Rich payload tier (history + git state, opt-in); additional OutputProviders
(iTerm2/kitty/WezTerm/screen); `--fix` scripting mode; offline rules fast-path;
entropy-based redaction; fish/PowerShell; auto-hint mode; malformed-response
retry; cosign/SLSA; homebrew-core; UI-chrome i18n; `allow_insecure` for
non-loopback http; Docker images. (Mirrors DESIGN.md §16.2.)

## 8. Known unknowns (may spawn new issues during implementation)

| ID | Unknown | Trigger to act | Likely action |
|---|---|---|---|
| U1 | tmux segmentation hit-rate in real prompts (powerlevel10k etc.) | e2e/dogfooding shows frequent "could not be attributed" | new issue: marker-assisted segmentation or prompt-pattern config |
| U2 | bash-preexec coexistence (iTerm2 integration, Atuin) | doctor reports or bug reports of double hooks | new issue: detection hardening in 09 |
| U3 | hook latency on slow filesystems (NFS $HOME) | p95 > 15 ms in testing | new issue: async write or tmpfs spool |
| U4 | malformed-JSON rate per provider/model | exit-4 frequency in dogfooding | new issue: single corrective retry (v2 item pulled forward) |
| U5 | redaction false positives annoy users | dogfooding feedback | new issue: rule tuning; never weaken defaults silently |
| U6 | GoReleaser ↔ tap token friction | first release dry-run fails | adjust issue 26 procedure |
| U7 | `json_mode` breaks some OpenAI-compatible gateways | provider 400s with json_mode on | document `llm.json_mode=false` workaround; consider auto-fallback issue |

New issues born from these must follow the same format as docs/issues/*.md and
be added to §2/§3 here first.

## 9. GitHub issue mapping (derived artifacts)

Created 2026-07-08 from the drafts below. If a GitHub issue and its draft file
disagree, the file wins (update the file first, then the issue).

| Draft file | GitHub issue |
|---|---|
| issues/01-go-scaffold.md | [#2](https://github.com/Saber5656/whydid/issues/2) |
| issues/02-ci-workflow.md | [#3](https://github.com/Saber5656/whydid/issues/3) |
| issues/03-community-files.md | [#4](https://github.com/Saber5656/whydid/issues/4) |
| issues/04-config-package.md | [#5](https://github.com/Saber5656/whydid/issues/5) |
| issues/05-record-store.md | [#6](https://github.com/Saber5656/whydid/issues/6) |
| issues/06-text-hygiene.md | [#7](https://github.com/Saber5656/whydid/issues/7) |
| issues/07-hook-record-command.md | [#8](https://github.com/Saber5656/whydid/issues/8) |
| issues/08-zsh-integration.md | [#9](https://github.com/Saber5656/whydid/issues/9) |
| issues/09-bash-integration.md | [#10](https://github.com/Saber5656/whydid/issues/10) |
| issues/10-tmux-capture.md | [#11](https://github.com/Saber5656/whydid/issues/11) |
| issues/11-llm-core.md | [#12](https://github.com/Saber5656/whydid/issues/12) |
| issues/12-openai-provider.md | [#13](https://github.com/Saber5656/whydid/issues/13) |
| issues/13-anthropic-provider.md | [#14](https://github.com/Saber5656/whydid/issues/14) |
| issues/14-payload-prompt.md | [#15](https://github.com/Saber5656/whydid/issues/15) |
| issues/15-response-parser.md | [#16](https://github.com/Saber5656/whydid/issues/16) |
| issues/16-explain-command.md | [#17](https://github.com/Saber5656/whydid/issues/17) |
| issues/17-renderer.md | [#18](https://github.com/Saber5656/whydid/issues/18) |
| issues/18-fix-selection-stdout.md | [#19](https://github.com/Saber5656/whydid/issues/19) |
| issues/19-consent-show-payload.md | [#20](https://github.com/Saber5656/whydid/issues/20) |
| issues/20-config-command.md | [#21](https://github.com/Saber5656/whydid/issues/21) |
| issues/21-doctor-command.md | [#22](https://github.com/Saber5656/whydid/issues/22) |
| issues/22-security-test-suite.md | [#23](https://github.com/Saber5656/whydid/issues/23) |
| issues/23-shell-e2e-harness.md | [#24](https://github.com/Saber5656/whydid/issues/24) |
| issues/24-hardening-audit.md | [#25](https://github.com/Saber5656/whydid/issues/25) |
| issues/25-goreleaser-release.md | [#26](https://github.com/Saber5656/whydid/issues/26) |
| issues/26-homebrew-tap.md | [#27](https://github.com/Saber5656/whydid/issues/27) |
| issues/27-user-docs.md | [#28](https://github.com/Saber5656/whydid/issues/28) |
