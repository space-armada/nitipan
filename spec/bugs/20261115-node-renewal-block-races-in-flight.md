# The node lease rig's renewal block lets an in-flight renewal through

**Status:** open

**Estimate:** small; no open design decisions. Two files at most: `internal/node/helpers_test.go` and `internal/node/lease_test.go`.

## Observed

`TestLostToAnotherHolderWritesANotice` in `internal/node/lease_test.go` failed once in a full `make ci` run under `-race`.
The second node's claim was refused: the first lease was still `live` at revision 3, with an expiry five minutes past the
advanced clock. The test passed five times in a row when run alone with `-race -count=5`.

## Cause

`blockRenewals(true)` stores an intercept that answers later `/v1/renew` requests with 503. A renewal that is already past
the intercept check when the block goes up still reaches the broker. Under load, that renewal lands after
`clock.Advance(7 * time.Minute)`, so the broker renews the lease on the advanced clock, and `Sweep` finds nothing to lapse.

This is a missing completion signal in the fixture, not a product bug. A real renewal racing a broker outage behaves the
same way, and that is correct.

## Repro

Run `make ci` on a loaded machine. Not reliable. The test alone does not reproduce it.

## Fix

Make `blockRenewals(true)` wait until no renewal is in flight before it returns. For example, count requests inside the
intercepted handler and wait for the count to drain. Other tests that call `blockRenewals(true)` and then advance the
clock have the same exposure, such as `TestNotFoundRenewalWritesALossNotice`.

## Triage notes

The race is shown from the code. The rig handler in `helpers_test.go` loads `r.intercept`, and only then calls the broker
handler. Nothing orders those two steps against `blockRenewals(true)`, which just swaps the pointer. The broker reads its
clock inside the renew handler, so a renewal that is past the check lands on whatever time it finds.

A deterministic reproduction exists. Install a pass-through intercept that signals once it holds a `/v1/renew` request and
then waits on a gate before returning false. Call `blockRenewals(true)` once the hook has signalled. Advance the fake clock by
7 minutes and release the gate. The held renewal reaches the broker on the advanced clock, and `Sweep` finds nothing to lapse.

Regression test shape: with the hook holding a renewal, call `blockRenewals(true)` in a goroutine. Assert that it has not
returned until the gate opens, then assert that it returns afterwards. Before the fix it returns at once, so the first assertion
fails every time.

Fix shape: count `PathRenew` requests in the rig's server wrapper. Increment before the intercept load and decrement when the
broker handler returns. Make `blockRenewals(true)` store the intercept first and then wait on a mutex and condition variable
until the count is zero. Count renewals only. Other paths, such as the keepalive stream, may stay open for a long time and would
never drain. The counter lives on the shared `rig`, which is correct because `newOtherNode` reuses the same server and tests
call `blockRenewals` on the original rig. Waiting uses a condition variable, not a sleep, per the testing rules. The other
`blockRenewals(true)` callers, including `TestNotFoundRenewalWritesALossNotice`, are fixed by the same change.
