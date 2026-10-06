# Verification snippets and checklists

Set these once per epic:

```bash
EPIC_BRANCH=feat/<epic>
REPOS="/path/to/repo-a /path/to/repo-b /path/to/repo-c"   # every repo in the epic
```

## Branch state on every repo

```bash
for r in $REPOS; do
  git -C "$r" fetch -q origin main "$EPIC_BRANCH"
  echo "$(basename "$r") head=$(git -C "$r" rev-parse --short "origin/$EPIC_BRANCH")" \
       "local=$(git -C "$r" rev-parse --short "$EPIC_BRANCH" 2>/dev/null)" \
       "ahead=$(git -C "$r" rev-list --count "origin/main..origin/$EPIC_BRANCH")" \
       "behindMain=$(git -C "$r" rev-list --count "origin/$EPIC_BRANCH..origin/main")"
done
```

After a user push: `head == local` and `behindMain == 0` on every repo.

## Verify a worker's reported commit

```bash
git -C "$r" fetch -q origin "$EPIC_BRANCH"
git -C "$r" log --oneline -3 "origin/$EPIC_BRANCH"          # the hash is there, on top
git -C "$r" show --stat --format='%h %s' <hash>             # touches what the worker said, nothing else
git -C "$r" show <hash> | grep -E '^[+-] ' | head -40       # read the actual change
```

## MR size and hot spots (for the review order)

```bash
for r in $REPOS; do
  echo "--- $(basename "$r")"
  git -C "$r" rev-list --count "origin/main..origin/$EPIC_BRANCH"
  git -C "$r" diff --shortstat "origin/main...origin/$EPIC_BRANCH"
  git -C "$r" diff --dirstat=files,5 "origin/main...origin/$EPIC_BRANCH" | head -10
done
```

## Scan MR diffs for a rule (added lines only)

Adjust the pattern to the rule. Example for a Java team: response wrappers, zone-less time types, raw HTTP
clients, schedulers.

```bash
PATTERN='ResponseEntity|LocalDateTime|OffsetDateTime|java\.util\.Date|@Scheduled|WebClient\.(builder|create)'
for r in $REPOS; do
  echo "=== $(basename "$r")"
  git -C "$r" diff -U0 "origin/main...origin/$EPIC_BRANCH" -- . ':(exclude)**/src/test/**' \
    | awk -v p="$PATTERN" '/^\+\+\+ /{f=substr($0,7);next} /^\+/ && $0 ~ p {print f": "substr($0,1,160)}'
done
```

Then, for every hit, decide:
- **New violation**: the epic introduced it. Route it to the owning phase.
- **Forced by an existing contract**: the type or shape already exists on main (an existing DTO, entity or
  wire field). Check with `git -C "$r" grep -n '<symbol>' origin/main`. Leave it and record it as follow-up.

Timestamp columns in new migrations:

```bash
git -C "$r" diff -U0 "origin/main...origin/$EPIC_BRANCH" -- '*.sql' | grep -iE '^\+.*(timestamp|date)'
```

Scheduled jobs: open each new one and confirm it is safe on several pods (row locks with SKIP LOCKED, an
advisory lock, or a distributed lock library).

## Pre-existing failures

When a build fails after a rebase, prove whether main already fails:

```bash
git -C "$r" worktree add /tmp/main-check origin/main
( cd /tmp/main-check && <build and test command> )
git -C "$r" worktree remove /tmp/main-check
```

Pre-existing failures go to the repo owner, not to the phase worker. Warn the user that the MR's CI will be
red for that reason.

## E2E gap scenarios (final run)

Run against the real dependency wherever possible and check every service's state, not only the API
response:

- Fault injection in front of the real dependency (a small proxy that fails one operation) to drive retry
  and give-up paths that a mock covered before.
- Projections and derived balances after partial operations.
- The exact-boundary case (commit equal to the reserved amount, limit exactly reached).
- Idempotent replays: same request id with the same payload, and with a different payload.
- Races on one resource with a real barrier (not background `curl &`): commit vs release, commit vs expiry.
- Kill and restart each stateful service mid-flow; recovery jobs must finish without double effects.
- Replayed or duplicated events (reset a consumer offset); no double counting downstream.
- Limit and quota rejections end to end.
- Record what could not run locally and why. Never fake a scenario to make it green.
