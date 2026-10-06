# Running the epic on spec-flow

[spec-flow](https://github.com/tranquocthong/spec-flow) (`/sf:*`) gives every phase a spec-first lifecycle:
SRS -> SD -> checklist -> implement -> manual-test -> regression. The coordinator model sits on top of it:
**one sf epic, one sf feature per phase, one worker per feature.**

Install: `/plugin marketplace add tranquocthong/spec-flow` then `/plugin install sf@claude-spec-flow`, and
`/sf:init` once in the planning repo.

## Layout

The planning repo (the one holding `.spec-flow/`) is shared by the coordinator and every worker.

```
.spec-flow/
  epics/<epic>/
    EPIC.md            # the epic record (coordinator owns it; see epic-template.md)
    srs/               # one SRS slice per phase: <epic>-p1-<name>.md, ...
    decisions/         # cross-phase design decisions (source of truth until SDs exist)
    state/             # environment state (SIT/UAT/prod) the epic depends on
    assets/            # mocks, scripts, context docs (business context only, not contracts)
    e2e-local/         # the epic-level E2E harness + RESULTS.md (E2E owner)
  specs/<epic>-pN-<name>/
    SD.md  CONTEXT.md  CHECKLIST.yaml  VERIFICATION.md  STATE.md  trace.json  changes/
```

Feature names: `<epic>-p<N>-<short-name>` so `epic-show` and the specs folder sort by phase.

## Stage mapping

| Stage | Coordinator | Worker (phase feature) |
|---|---|---|
| 0 Setup | `flow-tools epic-new --name <epic>`; write EPIC.md, `decisions/`, one SRS slice per phase in `srs/`. For one big SRS, `/sf:split` proposes the phase grouping (the user approves it). | - |
| 1 Specs | Review every SD for cross-phase consistency before the user approves it. May author a slice's SD itself (`/sf:ingest`) and hand implementation to a worker. | `/sf:ingest .spec-flow/epics/<epic>/srs/<slice>.md --epic <epic>` -> SD draft; fix TODOs; user approves; `/sf:checklist` |
| 2 Build | Watch contracts; route decisions. | `/sf:phase <feature>` (implements task by task, stops at the manual-test gate) |
| 2b Decision changes | Write the decision in EPIC.md + `decisions/`; update the affected SRS slices; send change notices. | Spec-first change: `/sf:change` (developer-initiated) or `/sf:resync` (the SRS slice changed). Each produces a change record; report its id + commit. |
| 2c Bugs | Classify: phase-local, cross-phase, pre-existing, external. | `/sf:bug` (reproduce first, then fix and regress) |
| 3 Verify | Read each phase's VERIFICATION and the epic RESULTS critically. | `/sf:manual-test <feature>` (smoke + regression from CHECKLIST.yaml). E2E owner also runs `e2e-local/`. |
| 4-6 Rebase, review, merge | As in SKILL.md. Review rules live in `.spec-flow/project-author.md` (Code review checklist). | New commits only; re-run `/sf:manual-test` when behavior changed. |
| Close | Set `status: done` in EPIC.md, list open dependencies; `/sf:status` per feature should show done. | - |

## spec-flow specifics that bite in a multi-session epic

- **Global state mirror.** `.spec-flow/STATE.md` and `.spec-flow/trace.json` hold ONE active feature. Every
  worker running sf commands in the same planning repo moves the mirror to its own feature. The durable
  state is per feature (`specs/<feature>/STATE.md`, `specs/<feature>/trace.json`). Read those, not the mirror;
  restore the mirror with `flow-tools state-update --feature <feature>` when needed, and expect the
  `ACTIVE FEATURE SWITCHED` warning.
- **`trace-build` drops the repo scope.** On a multi-repo project re-run
  `flow-tools trace-repos --feature <feature> --set "<repos>"` after every `trace-build`.
- **Branching.** sf's per-SD branch template (`feat/{feature}`) does not fit one branch per repo for the
  whole epic. Manage `feat/<epic>` branches yourself (or configure `branching` accordingly) and skip
  `branch-ensure` for epic features.
- **Concurrent commits in the planning repo.** Several sessions commit there at once: commit straight to
  main, never `--amend`, and check the largest existing change-record id before creating a new one (ids are
  counted from files and can collide between sessions).
- **Review rules are invisible to implementers.** `project-author.md` is read by the SD author and by
  reviewers, not by the implementing agent. Put each rule that can be grepped into
  `config.verify.forbiddenPatterns` so the verify gate in `/sf:phase` stops it, and list the rest in the
  worker brief.
- **Requirements owned by another team.** In the SRS slice, make them `Won't (this epic)` plus a
  dependency (`DEP-nn`) with the owner, not a Must FR the worker cannot deliver.
- **Epic E2E is not a feature checklist.** Each phase's CHECKLIST covers that phase. The cross-service E2E
  lives under `epics/<epic>/e2e-local/` and is owned by one worker; its RESULTS entries carry every repo's
  commit hash.
