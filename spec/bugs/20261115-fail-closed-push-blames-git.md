# A fail-closed `nitipan push` says git refused, but git never ran

**Status:** open

## Observed

`failClosedPush` in `internal/cli/push.go` words both of its errors as "git refused to push, because ...". It runs when
`analyze` cannot work out which paths a push touches. One example is a remote tip that is missing from the clone, so
`RemoteTouched` cannot walk the commits. `pushOnce` returns before `gitPush`, so git is never asked. The refusal is
nitipan's.

The hook's matching message in `internal/cli/githook.go`, "git refused to update a branch or HEAD", is accurate there.
A hook's refusal does make git refuse. The pusher reused the wording where it no longer holds.

## Repro

Hold a lease, then let the remote branch advance from another clone without fetching it. Run `nitipan push origin main`.
The error says git refused to push. No `git push` ran.

Found by the bug sweep's design critic during triage of
[20261114-nonff-branch-refused-as-extra-fence](20261114-nonff-branch-refused-as-extra-fence.md). The code path is
confirmed. The repro was not run by hand.

## Fix

Say that nitipan refused the push, for example "nitipan did not push, because ...". Keep the wrapped cause. A test
that pins the message must change with it.
