# Title

Scaffold Go module, repository layout, and Makefile

## Summary

Create the Go module `github.com/Saber5656/whydid`, the package skeleton from
DESIGN.md §4, a minimal CLI dispatcher with `version` and `help`, and a Makefile
with the standard developer targets.

## Context

Every other issue adds code into this skeleton. The dispatcher is hand-rolled
(no cobra) per ADR-004; startup cost of the binary matters because
`whydid hook record` runs at every shell prompt.

## Scope

- `go.mod` (module `github.com/Saber5656/whydid`, Go version = latest stable at
  implementation time, recorded in go.mod and in `.go-version`)
- `cmd/whydid/main.go` — calls `internal/cli.Run(os.Args[1:])`, `os.Exit` with
  the returned code
- `internal/cli/` — dispatcher, global `-h/--help/--version`, usage text; stub
  registrations for future subcommands returning "not implemented" (exit 5)
- `internal/version/` — `Version`, `Commit`, `Date` vars settable via ldflags,
  defaulting to `dev`
- `Makefile` — targets: `build`, `test` (with `-race`), `lint`
  (golangci-lint run), `fmt` (gofmt -l check), `vet`, `clean`
- `.golangci.yml` — enable: govet, staticcheck, errcheck, gosimple, ineffassign,
  unused, gofmt, misspell, gosec
- `.gitignore` for Go artifacts

## Detailed Requirements

1. `internal/cli.Run(args []string) int` parses the first non-flag argument as
   the subcommand; registry is `map[string]func(args []string) int`. Unknown
   subcommand → usage on stderr, return 2 (DESIGN §11.1).
2. No subcommand → dispatch to `explain` stub (this is the default command).
3. `whydid version` and `whydid --version` print
   `whydid <Version> (<Commit> <Date>)` to stdout, return 0.
4. Exit-code constants defined once in `internal/cli/exitcodes.go` exactly as
   DESIGN §9.1: 0,1,2,3,4,5 with named constants (`ExitOK`, `ExitNothing`,
   `ExitConfig`, `ExitProvider`, `ExitBadResponse`, `ExitInternal`).
5. A `panic` recovery wrapper in `Run` converts panics to exit 5 with a one-line
   stderr message (no stack trace by default; full trace when `WHYDID_DEBUG=1`).
6. No third-party imports in this issue.

## Acceptance Criteria

- [ ] `make build` produces `./whydid`; `./whydid version` prints the dev version.
- [ ] `./whydid nosuchcmd` prints usage to stderr and exits 2.
- [ ] `./whydid` (no args) exits 5 with "not implemented" (until issue 16 lands).
- [ ] `make test`, `make lint`, `make vet`, `make fmt` all pass.
- [ ] `go.mod` has zero `require` entries.
- [ ] Binary startup (`./whydid version`) completes in < 20 ms on the dev machine
      (informal check; guards ADR-004 latency budget).

## Validation

Unit tests for the dispatcher: routing, unknown command, `--version`, panic→5.
Run `make test lint` locally; CI wiring arrives in issue 02.

## Dependencies

None (first issue).

## Non-goals

Any real subcommand behavior; CI; release config; third-party deps.

## Design References

DESIGN.md §4, §9.1 (exit codes), §11.1; ADR-004.
