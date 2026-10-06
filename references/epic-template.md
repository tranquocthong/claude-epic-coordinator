# Epic record template

Keep this file in the planning repo (for example `epics/<epic>/EPIC.md`). The coordinator owns it.

```markdown
id: <epic>
status: active            # active | done
created: <YYYY-MM-DD>

## Scope

<2-4 sentences: what the epic delivers, for whom, what stays generic (no partner/customer names in code).>

Design source of truth: `decisions/<topic>.md` until the SDs exist.

## Phases

Specs are authored outside-in (outermost contract first). Code may run in parallel against mocks.

| Phase | Repo(s) | Scope | Depends on |
|---|---|---|---|
| p1-<name> | <repo-a>, <repo-b> | <what this phase delivers> | contract of p2 |
| p2-<name> | <repo-c>, <shared-lib> | ... | p4 |
| p3-<name> | <repo-d> | ... | p2 (event contract) |
| p4-<name> | <repo-e> | ... | DEP-01 (owner: <team>) |

## Roster

| Phase | Session | Channel | Address |
|---|---|---|---|
| coordinator | <name> | - | - |
| p1 | <name> | agent-talk | pair `<code>` |
| p2 | <name> | agent-talk | pair `<code>` |
| p3 (E2E owner) | <name> | SendMessage | `<address>` |
| p4 | <name> | agent-talk | pair `<code>` |

## Delivery (decided <date>)

- One branch per repo for the whole epic: `feat/<epic>`. One MR per repo. All MRs merge together, once,
  after the E2E gate.
- Code work happens in a git worktree on `feat/<epic>`, never by switching a shared checkout.
- Exception: `<shared-lib>` publishes a pre-release from `feat/<epic>` so other repos can build; the final
  release ships with the one-time merge.
- Deploy order at merge: <shared-lib> -> <repo-x>, <repo-y> -> <repo-z> -> ...
- Full local E2E for the whole epic is owned by <phase>.
- Rebase onto latest main once, right before the merge. No history rewrite during MR review.

## Decisions

Decided <date>: <decision>. (Superseded decisions: ~~strike through~~ and point to the new line.)

## Dependencies

| Id | What | Owner | Blocks merge? |
|---|---|---|---|
| DEP-01 | <fix another team owns> | <team> | no |

## Open decisions

1. <question> - recommendation: <...>
```
