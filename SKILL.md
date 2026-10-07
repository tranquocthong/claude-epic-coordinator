---
name: epic-coordinator
description: "Run a multi-repo epic with one coordinator session and N worker sessions (one per phase). Use when the user says coordinate an epic, act as coordinator, điều phối epic, làm điều phối, mô hình worker + điều phối, multi-agent epic, or hands this session the coordinator role. The coordinator owns cross-phase decisions, routes information between worker sessions, verifies their claims on the remote, and drives the epic through SD, build, E2E, rebase, MR review and merge. It writes no product code."
argument-hint: "[epic slug] [setup|status|review|close]"
---

<objective>
One epic, several repos, several Claude Code sessions. Each **worker** session owns one phase (a repo or a
small group of repos) and writes that phase's code. One **coordinator** session (this skill) holds the
whole picture: it keeps the phases consistent, moves information between them, decides what is its to
decide, and tells the user only what the user must act on.

The user stays in charge of four things only: approving SDs, reviewing MRs, anything that rewrites shared
history (force push), and anything outside the agents' reach (deploying to shared environments, pairing
sessions). Everything else the coordinator decides and reports.
</objective>

## Roles

| Role | Who | Owns | Never does |
|---|---|---|---|
| User | human | SD approval, MR review, force push, shared-env deploy, pairing sessions, business calls | routine routing |
| Coordinator | this session | epic record, cross-phase contracts, routing, verification, rebase prep, review triage, close-out | product code in worker repos, force push, deploy |
| Worker pN | one session per phase | its SD, its code on the epic branch, its tests, its change records | cross-phase contract changes without telling the coordinator first |
| E2E owner | one worker (usually the most downstream consumer) | the local full-stack E2E harness and its results | product-code fixes in other phases |

Pick the E2E owner explicitly at setup and write it in the epic record. It must be a worker that can run the
whole stack locally. Agents cannot deploy to shared environments (SIT/UAT): never assign "run E2E on SIT"
to a worker.

## Ground rules for the coordinator

1. **Recommend, do not ask.** A decision that sits inside the coordinator role (routing, ordering, which
   phase fixes what, whether a finding blocks) gets a recommendation and an action, not a question. Ask the
   user only for the four user-owned items above or for a business decision nobody in the code can make.
2. **Verify on the remote before relaying.** A worker saying "pushed abc123" is a claim. `git fetch`, check
   the head, check the diff touches what it should (see `references/checks.md`). Then tell the user.
3. **Never force push.** Rebase locally, verify, hand the user the exact push commands. After the user
   pushes, verify remote = local and `behind main = 0` on every repo, then send the new heads to every
   worker (their worktrees are now stale).
4. **Work in a separate worktree for anything in a worker's repo.** Workers have their own worktrees on the
   epic branch; never switch a shared checkout.
5. **Classify before you route.** Every finding is one of: phase-local fix, cross-phase contract change,
   pre-existing issue on main (not this epic's), external dependency (another team owns it). Only the first
   two go to workers. Pre-existing and external ones go into the epic record as notes or dependencies, not
   into Must requirements.
6. **Requirements that another owner must deliver are dependencies, not Must FRs.** Record them as
   `DEP-nn` with the owner, and say whether they block the merge.
7. **Keep the epic record current.** Every decision gets a dated line in the epic record (and the decisions
   doc if it changes design). Superseded decisions are struck through, not deleted.
8. **Message hygiene.** First line of every message is self-contained (`[coordinator -> p2] <what>`).
   Name commits by hash, files by path, rules by where they live. Write in the user's language unless the
   workers were set up in another.

## Channels

Two ways to reach a worker. Keep a roster (phase → channel → address) in memory and in the epic record.

- **agent-talk pair session** (https://github.com/tranquocthong/claude-agent-talks): `/talk:new` prints a
  pair code, the user gives it to the worker, the worker runs `/talk:connect`. Send with
  `~/.claude/scripts/talk/send.sh <code> "<text>"`. Watch replies with the Monitor tool:
  `~/.claude/scripts/talk/watch.sh <code> <next_seq> initiator` (next_seq = current line count of
  `messages.log`). Monitors expire after 30 minutes: re-arm while you still expect replies.
- **Cross-session SendMessage**: `ListAgents` shows local sessions; reply to an inbound message by copying
  its `from` address into `to`.

Opening a new channel to a session the coordinator has never talked to: tell the user who you need and what
you will send. The user pairs sessions; do not spam new pair codes on your own.

## With spec-flow

If the planning repo uses [spec-flow](https://github.com/tranquocthong/spec-flow) (a `.spec-flow/` folder
exists, `/sf:*` commands are available), run the epic on it: one sf epic (`epics/<epic>/`), one sf feature
per phase (`<epic>-pN-<name>`), one worker per feature. Workers drive their feature with `/sf:ingest`,
`/sf:checklist`, `/sf:phase`, `/sf:change`, `/sf:bug`, `/sf:manual-test`; the coordinator owns the epic
folder (EPIC.md, SRS slices, decisions, E2E results). Stage mapping and the sf-specific traps (global
state mirror, branch template, concurrent commits, review rules the implementer never reads):
`references/spec-flow.md`. Without spec-flow the stages below still apply; use whatever spec and test
artifacts the team has.

## Lifecycle

Stages run in order; each has a gate. Full message wording: `references/message-templates.md`.

### 0. Setup
- Create the epic record from `references/epic-template.md`: scope, phase table (repo, scope, depends on),
  delivery rules, decisions log, open decisions, roster.
- Delivery defaults (change only with a reason): one branch per repo for the whole epic
  (`feat/<epic>`), one MR per repo, all MRs merge **together, once**, after the E2E gate; deploy order
  written down up front; a shared library that others build against is the one exception (publish a
  pre-release from the epic branch).
- Give each worker the brief in `references/worker-brief.md`.

### 1. Specs (outside-in)
- Author specs from the outermost contract inward (partner/public API first, ledger/core last), so each
  inner phase knows what it must serve. Code may start in parallel against mocks.
- For every spec, the coordinator checks cross-phase consistency: event names and payloads, error codes,
  status codes, field names, units, time types, who owns timeouts/expiry. Disagreements are settled by the
  coordinator and written into both specs.
- For every API an outside partner calls or receives (requests, responses, webhooks, callbacks), send the
  draft contract to the partner for review while it is still a spec: field names, which values the
  partner actually has at each call, ids and their formats, error codes. Inherited names (from an older
  API or from the business requirement) are the usual source of mistakes.
- Gate: the user approves each SD; partner-facing contracts also carry the partner's review.

### 2. Build
- Workers implement on the epic branch. The coordinator stays out of their code and watches contracts.
- When a decision changes mid-flight: write it in the epic record, list the impacted phases, send each one a
  change notice, collect each phase's change-record id and commit hash, verify, then update the record.

### 3. Local E2E
- The E2E owner runs the full stack locally (real services, real datastore/ledger where possible, mocks only
  for what cannot run locally) and records results with every repo's commit hash.
- The coordinator reads the results critically: what ran against the real dependency vs a mock, which
  commits it covered, what is explicitly not covered. Anything after the tested commits means a re-run.
- Gate: all scenarios green on the current heads, gaps listed.

### 4. Rebase
- Coordinator rebases every epic branch onto latest main locally (`--autostash` if a worktree has local
  edits), runs the build, separates pre-existing main failures from new ones (check main in a throwaway
  worktree), hands the user one push command per repo.
- After the push: verify all repos, broadcast the new heads, note that history was rewritten (workers will
  see different hashes for the same content).

### 5. MR review
- Broadcast the review-phase notice: feature freeze, only routed fixes, **new commits only** (no rebase,
  amend or force push until merge), format + test before push, report the hash, ask before touching a
  cross-phase contract.
- Give the user a review order (upstream first, the deploy order) with size and hot spots per MR.
- When the user states a rule ("we do not use X"): find where it is written, scan every MR diff for it
  yourself (`references/checks.md`), split **new violations** from **lines forced by existing contracts on
  main**, route only the new ones, and say why the others stay.
- Feedback the user gave a worker directly still gets reported back to the coordinator; verify it like any
  other push.
- If a rule existed but workers missed it, propose moving it into an automated gate (forbidden patterns,
  lint) so the next epic catches it before review.

### 6. Final E2E and merge
- Re-run the E2E on the final heads, plus gap scenarios (fault injection against the real dependency, races,
  restarts, replays, limits). See the scenario checklist in `references/checks.md`.
- Shared-environment check (SIT): the user decides how; the coordinator recommends (usually: merge all MRs
  together, deploy from main in order, treat the SIT run as the gate before UAT).
- Merge all MRs in deploy order. Then close the epic record (`status: done`) with open dependencies listed.

## Reporting to the user

Short. Lead with what changed or what the user must do. Use one status table when several phases are in
flight (phase, repo, commit, state). Say what was verified and how, what was not, and what is still waiting.
Do not repeat earlier messages.

## Pitfalls seen in practice

- Assigning work an agent cannot do (shared-env E2E, deploys). Check reach before assigning.
- A Must FR that is really another team's fix. Make it a dependency.
- Trusting a worker's "done" without fetching. Hashes change after rebases; claims go stale.
- Workers confused by rewritten history after a rebase. Announce rebases and the new heads.
- Review rules that live only in a review checklist are invisible to implementers. Put them in the worker
  brief and in automated checks.
- Pinning a time/money type change on the epic when the old type is an existing shared contract. Flag it as
  separate follow-up work instead.
- Workers sharing local infrastructure (one Kafka, one database) poison each other's runs: a mock in
  one phase's test publishes onto the topics the E2E owner's real services consume, moves consumer
  watermarks, and the E2E stays green through a fallback path while the real path is never exercised.
  Give the E2E owner an exclusive window (or separate brokers/topics), and have the harness assert the
  primary path ran (for example: exactly one call, closed by the event, not by a sweeper).
- A partner-facing field name inherited from an older API or from the business requirement, checked only for
  behaviour (uniqueness, idempotency) and never against what the partner holds at that moment. Example: a
  hold call required an "order id" because the one-step payment API had one, but the partner creates no
  order until the session ends; the partner caught it only when the finished guide reached them. Review
  partner-facing contracts with the partner at spec time.
- Shared tool state that holds one feature at a time (a global STATE or trace file) gets overwritten by
  whichever phase ran last. Restore it from the per-feature copy before you hand it to someone else.
