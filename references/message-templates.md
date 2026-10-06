# Message templates

First line is always self-contained: `[coordinator -> <target>] <what this is>`. Translate to the language
the workers use.

## Task handoff

```
[coordinator -> pN] <one-line task>.
Why: <one or two sentences of context>.
Do: <concrete changes, file/class names where known>.
Keep: <what must not change, e.g. wire status codes and bodies>.
Push: new commit on feat/<epic>, format + tests first, report the hash.
```

## Cross-phase change notice (send to every impacted phase)

```
[coordinator -> pX, pY] Decision <date>: <the change in one line>.
Impact on pX: <what pX changes>. Impact on pY: <...>.
Unchanged: <what stays the same, so nobody over-reacts>.
Each phase: open a change record, report change id + commit hash.
```

## Review-phase notice (broadcast)

```
[coordinator -> all phase owners] The epic is now in MR REVIEW.
Rules until merge:
(1) Feature freeze: only fixes routed by the coordinator.
(2) New commits only on feat/<epic>. No rebase, amend or force push; the single rebase onto main happens
    right before merge, run by the coordinator.
(3) Format + tests before every push; report the hash.
(4) Anything touching a cross-phase contract: tell the coordinator BEFORE coding.
(5) Behavior-changing fixes trigger an E2E re-run by the E2E owner.
(6) Self-check new code against <review checklist path>.
```

## Review finding (rule violation)

```
[coordinator -> pN] Review (user): rule "<rule>" (<where it is written>).
Fix: <file/class> - <method> -> <target shape>.
Leave as is: <lines forced by existing contracts on main, and why>.
Wire behavior unchanged. New commit, format + tests, report the hash.
```

## New heads after a rebase

```
[coordinator -> E2E owner, all] All epic branches were rebased onto latest main and pushed (history
rewritten, content unchanged). New heads: <repo> <hash>, ... Sync your worktree:
git fetch && git reset --keep origin/feat/<epic>
```

## E2E re-run request

```
[coordinator -> E2E owner] Re-run the full E2E on current heads.
Why: last run covered <hashes>; since then: <commits per repo>.
Before: sync every worktree, rebuild.
Record: new RESULTS entry with every repo's hash; commit the results; report pass/fail. On failure, report
evidence first, do not fix.
```

## Reply to the user (status)

```
<What changed, one line.> Verified: <how>.

| Phase | Repo | Commit | State |
|---|---|---|---|
| p1 | ... | abc1234 | done |
| p2 | ... | - | waiting |

Your move: <only if the user must act>.
```
