# Title

Record store: JSONL session files (append/read/compact/GC)

## Summary

Implement `internal/store`: the per-session JSONL command-record files under the
state dir, with atomic appends, tolerant reads, size-capped compaction, and
time-based garbage collection.

## Context

This is whydid's only persistent data (besides config). Its schema and hygiene
rules are normative in DESIGN §6; threat T3 (secrets at rest) is partially
mitigated here via permissions and GC (redaction itself happens in issue 07).

## Scope

- `internal/store/record.go` — `Record` struct + JSON tags exactly per DESIGN §6.2
- `internal/store/store.go` — `Append`, `ReadAll`, `LatestFailure`, `Compact`, `GC`
- `internal/store/path.go` — `SessionFile(stateDir, sessionID)` with session-id
  sanitization

## Detailed Requirements

1. `Record` fields/tags: `v, seq, cmd, exit, cwd, shell, start_ts, end_ts, tty,
   tmux_pane, cmd_truncated` (types per DESIGN §6.2 table). `v` written as `1`.
2. Session id sanitization: accept only `[A-Za-z0-9._-]{1,64}`; anything else →
   error (session ids come from the environment; path traversal must be
   impossible).
3. `Append(dir, sid, rec)`:
   a. ensure dir chain `<state>/sessions` exists with 0700 (explicit `MkdirAll`
      + `Chmod`), file opened `O_CREATE|O_WRONLY|O_APPEND`, `Chmod(0600)`;
   b. `seq` = last valid record's seq + 1 (read tail; on unreadable file start
      at 1);
   c. marshal to a single line + `\n`, one `Write` call;
   d. after write, if `stat.Size() > 256*1024` → `Compact` to newest 50 records
      via temp file (0600) + `rename`.
4. `ReadAll(dir, sid)`: parse line-by-line; skip lines that fail to unmarshal or
   have `v != 1` (forward compat, torn writes — DESIGN §6.2/§6.4); cap: read at
   most the last 512 KiB of the file.
5. `LatestFailure(records, n)`: scan the newest `n` (=10 at call sites) records
   newest-first; skip records whose `cmd` first token is `whydid`; return the
   first with `exit != 0`; also report whether the newest non-skipped record
   succeeded (explain S1 distinguishes "nothing to explain" from "no records" —
   DESIGN §9.1).
6. `GC(dir, maxAge)`: delete `sessions/*.jsonl` with mtime older than maxAge
   (72 h at call sites); ignore errors per-file; return count deleted.
7. No goroutines, no locks (documented single-writer trade-off DESIGN §6.4).

## Acceptance Criteria

- [ ] Append→ReadAll round-trips all fields; file is 0600, dir 0700 regardless
      of umask (test with umask 0).
- [ ] A file containing a torn half-line among valid lines yields the valid
      records only.
- [ ] Compaction triggers at >256 KiB and leaves exactly the newest 50 records,
      order preserved, atomically (old or new content visible, never partial).
- [ ] `LatestFailure` skips `whydid …` records and returns the documented
      distinction for "newest succeeded".
- [ ] Session id `../../etc/passwd` is rejected.
- [ ] GC deletes only files older than the threshold.

## Validation

Unit tests with `t.TempDir()`; a fuzz-style test appending 10k records asserting
reader tolerance and compaction bounds; `make test` with `-race`.

## Dependencies

01.

## Non-goals

Redaction (06/07 apply it before calling `Append`); reading records of other
sessions; any locking/daemon.

## Design References

DESIGN.md §6 (all), §9.1 S1, §12.2 T3, §13 E7/E8.
