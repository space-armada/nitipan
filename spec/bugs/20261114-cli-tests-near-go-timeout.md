# The `internal/cli` test package runs close to go's 10-minute test timeout

**Status:** open

**Estimate:** large; non-trivial design decisions. The Fix section lists four options and records no choice. Splitting
the package moves test helpers and changes the layout. Retiring tests needs a judgment about which contracts another
test already owns. Raising `-timeout` is ruled out by the testing standard.

**Verified:** 2026-11-15. `go test -race -count=1 ./internal/cli` passed in 560.4 s on the current build, about 40 s under the 600 s limit. The bug still reproduces as a margin problem.

## Observed

`make test` runs `go test -race ./...` with no `-timeout`, so each package gets go's default of 10 minutes. On this
macOS checkout, a whole `make ci` took about 450 s before the fence-fallback work. `internal/cli` alone took 501 s
after the porcelain classifier, and 568 s to 587 s after the fence modes. One run during the fence-mode work hit the 600 s limit
and failed with a timeout panic, though no test had failed. A rerun passed at 568 s after the new rig tests shared one
rig.

A slower machine, or a few more e2e tests, will fail `make ci` with no test at fault.

## Cause

The package holds most of the e2e tests. Each rig test re-runs the test binary as `nit` through shell scripts against
local bare remotes, and the wide tests start httptest brokers and nodes. They run under `-race`. The package is one
unit for go's timeout, so its whole wall time counts against the limit.

## Repro

Run `make ci` and read the `internal/cli` line of the test output. Compare it with 600 s.

## Fix

This needs a design call. Raising `-timeout` hides the cost and does not fix it, per the testing standard. The options:

- Audit the package's e2e tests with `/test-audit`, and retire the ones another test already proves
- Split the e2e tests into their own package, so the unit tests and each e2e group get their own budget
- Share rigs and fixtures across table rows, as the fence-mode hook table now does
- Adopt a per-file budget for the package, per the testing standard's "Per-File Budgets"
