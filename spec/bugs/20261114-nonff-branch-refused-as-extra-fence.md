# A behind branch makes the hook refuse the fence ref as one the leases do not call for

**Status:** open

**Estimate:** medium; open design decisions, named below. The third option in Fix keeps the guard and fixes the
message on both paths. It still reproduces on 2026-11-15:
`TestAFencedPushFailsAsTheRulesSayForEachKindOfRemote/a_branch_that_is_not_a_fast-forward` passes, and it pins the
misleading "do not call for" refusal. The fix touches `internal/cli/push.go`, `internal/cli/prepush.go`, the test case
in `push_classify_test.go`, and `docs/reference/pushing.md`. A bug sweep's design critic found these decisions open:

- Where the pusher's behind-branch check sits. It should come after `analyze` in `pushOnce`. It should run only when
  the plan has fence writes, because a push with no writes goes without `--atomic`. There git lands the branches that
  fast-forward and rejects only the behind one, with its own clear message. An ungated check would refuse both. A
  remote tip that is missing locally already fails closed in `analyze`, so the check never sees that case.
- Which condition adds the hook's hint. The Fix option says "the fence refs it sees would be called for by the agent's
  leases". The triage said "the hook's input held no branch update". Under the second condition, the attack in
  `TestAnAgentCannotPushAFenceRefItsLeasesDoNotCallFor` also gets the "branch may be behind" hint. A hand-run push of
  `main feature` with a behind main gets no hint. The lease-based condition avoids both problems. It needs the keys
  passed into `requireFences`, with `fenceref.Parse` and `fenceplan.Covering`.
- The exit status of the new pusher failure. Today a behind branch exits 3 (`ErrRefused`), because the hook prints
  `nitipan:`. Code 3 means an overlapping lease blocks the claim. A plain error would exit 1, which matches git's own
  rejection under `fence=false`. Scripts can see the change.

## Observed

An agent that holds a lease runs `nitipan push origin main` while `origin/main` has moved ahead, so the push is not a
fast-forward. The push fails with:

```
nitipan: git refused to push to origin. The push carries fence refs that the leases you hold do not call for:
refs/nitipan/fence/<root>/src. Only nitipan push writes fence refs, and only for your own keys
```

The fence ref is the one the lease calls for. The real cause is that the branch is behind, and the message does not say
so.

## Cause

Git rejects a non-fast-forward branch locally and leaves it out of the pre-push hook's input. The hook sees only the
fence ref. `runPrePush` in `internal/cli/prepush.go` finds no branch update, so the plan has no write for that fence.
`requireFences`, around line 242, then counts the fence ref as extra and refuses it. The remote is never asked.

The pusher's own plan, built from `ls-remote`, does call for the fence. The two plans disagree because the hook cannot
see the branch.

## Repro

The test case "a branch that is not a fast-forward" in `TestAFencedPushFailsAsTheRulesSayForEachKindOfRemote`, in
`internal/cli/push_classify_test.go`, pins today's behavior. The agent holds a lease on `src`, commits, and the remote
branch advances by a commit under `src` from another clone, which the agent then fetches. `nit push origin main` fails
with the message above and the output "git did not push".

To reproduce by hand, push a behind branch with a held lease and read the hook's message.

## Fix

This needs a design call. The options:

- The hook skips the extra-fence check when no branch ref survives in its input. A push of fence refs alone from an
  agent would then pass the hook, which weakens the guard against an agent pushing an arbitrary fence ref with a huge
  token. The hook would need another way to tell that case from a rejected branch.
- The pusher detects a non-fast-forward before it pushes, by comparing the remote tip with the branch, and fails with
  "the branch is behind, fetch and rebase" without running git. The hook still refuses a hand-run push of the same shape
  with the misleading message.
- Both: the pusher gives the clear message for its own pushes, and the hook words its refusal to name a rejected branch
  as a possible cause when the fence refs it sees would be called for by the agent's leases.

The third option fixes the message in both paths without relaxing the guard. The first weakens a guard, and the second
leaves the hook message wrong for hand-run pushes. NIT-009 M1 does not touch this: a behind branch already ends the
push with no fallback.
