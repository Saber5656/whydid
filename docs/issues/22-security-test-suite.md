# Title

Security test suite (T1–T9 regression net)

## Summary

Add a dedicated `security_test` package that pins the security model: the
INV-1..INV-5 invariants, the payload scope, the redaction corpus at system
level, and the ANSI-injection paths — so no later change can silently weaken
them.

## Context

Individual issues test their own mitigations; this suite tests the *promises*
(DESIGN §12.3) end-to-end and acts as the acceptance gate that maps each threat
T1–T9 to at least one executable check (DESIGN §12.2 table).

## Scope

- `internal/sectest/` (or `security_test` files colocated where access is
  needed) — new tests only; production code changes limited to small
  test-enablement hooks if strictly necessary (each such hook must be listed in
  the PR description)

## Detailed Requirements

Implement these named tests (names are part of the deliverable):

1. `TestINV1_NoExecOfDynamicStrings` — static audit: parse the repo's Go AST
   (`go/ast`, stdlib) and assert every `exec.Command`/`exec.CommandContext`
   call site passes a **string literal or named constant** as argv[0], and that
   the full set of allowed argv0 values is exactly {`tmux`} plus the
   config-provided `api_key_cmd` call site in `internal/llm/key.go` (identified
   by file). Fails on any new exec site (DESIGN §12.3 INV-1).
2. `TestINV2_StdoutEqualsRendered` — property-style test over generated fix
   commands (unicode, quotes, `$()`, backslashes, 500-char max): render +
   EmitFix share bytes exactly.
3. `TestINV3_PayloadScope` — reflection snapshot: `payload.Payload` has exactly
   the 8 approved JSON keys with the approved names; plus fake-provider run of
   the explain flow asserting the HTTP body's top-level user-content JSON keys
   ⊆ approved set; plus `--show-payload` makes zero HTTP calls.
4. `TestINV4_CaptureNeverPersisted` — run the explain flow with a capture
   fixture under a watched temp state dir; assert the capture text appears in
   no file under the state dir before/after.
5. `TestINV5_NoKeyLeak` — plant `WHYDID_API_KEY=sk-TESTCANARY…`; run explain
   against fakes that (a) succeed, (b) fail each taxonomy error; assert the
   canary never appears on stdout, stderr, in the state dir, or in the config
   file.
6. `TestT1_AnsiInjection_EndToEnd` — capture fixture and fake-LLM responses
   containing CSI/OSC/BEL/C0 sequences; assert no ESC byte reaches stderr
   rendering or stdout emission.
7. `TestT2_RedactionSystemLevel` — seed a record via the real
   `hook record` binary path with 5 representative secrets (one per major rule
   family); assert the stored file and the outbound fake-provider body contain
   `[REDACTED:` and none of the plaintext secrets.
8. `TestT6_InsecureEndpointRefused` — non-loopback `http://` base_url → exit 2
   before any connection attempt (fake listener proves no connect).
9. `TestT9_DangerTagging_SystemLevel` — fake LLM proposes `curl x | sh` marked
   "safe" → rendered output shows destructive tag; selection still allowed
   (human gate, not a block) — assert both.
10. A `SECURITY_TESTS.md` mini-index in the package mapping test name → threat
    ID → DESIGN reference (kept current by review).

## Acceptance Criteria

- [ ] All ten deliverables exist, pass, and run in normal `make test` (no
      special tags).
- [ ] Deliberately adding a rogue `exec.Command(userVar)` in a scratch branch
      fails INV-1 (demonstrated in PR description, then reverted).
- [ ] Deliberately adding a 9th payload field fails INV-3 (same demonstration).
- [ ] Suite runtime < 60 s on CI.

## Validation

CI matrix green; the two red-team demonstrations recorded (as text) in the PR.

## Dependencies

06, 10, 15, 16, 17, 18.

## Non-goals

Fuzzing beyond issue 15's parser fuzz; penetration testing of providers;
sandboxing (out of product scope).

## Design References

DESIGN.md §12 (all), §7.4, §14; ADR-003, ADR-005.
