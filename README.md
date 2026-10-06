# epic-coordinator

A Claude Code skill for running a multi-repo epic with **one coordinator session and N worker sessions**,
one worker per phase.

Workers write code for their phase. The coordinator writes no product code. Its job is to:

- keep the phases' contracts consistent,
- move information between sessions,
- verify every claim on the git remote,
- decide what is its to decide,
- tell you only what you must act on.

You keep four things: approving specs, reviewing MRs, force pushes, and work the agents cannot reach
(deploying to shared environments, pairing sessions).

```
                 you (approve specs, review MRs, push, deploy)
                                  |
                          coordinator session
                 (epic record, contracts, routing, verify)
              /            |              |              \
       worker p1      worker p2      worker p3      worker p4
      (repo a, b)    (repo c, lib)  (repo d, E2E)    (repo e)
```

## What it covers

- Roles and ground rules: recommend instead of asking, verify on the remote, never force push, classify
  findings before routing them.
- Lifecycle stages with gates: setup, specs (outside-in), build, local E2E, rebase, MR review, final E2E
  and merge.
- Channels between sessions: [agent-talk](https://github.com/tranquocthong/claude-agent-talks) pair
  sessions or cross-session `SendMessage`.
- Runs on [spec-flow](https://github.com/tranquocthong/spec-flow) when present: one sf epic, one sf
  feature per phase. See `references/spec-flow.md`.
- Templates: the epic record, the worker brief, and the messages for handoffs, change notices, the
  review-phase notice, review findings and E2E re-runs.
- Verification snippets: branch state across repos, checking a reported commit, MR sizing, scanning MR
  diffs for a rule, telling apart pre-existing failures on main, gap scenarios for the final E2E.

## Install

```bash
git clone https://github.com/tranquocthong/claude-epic-coordinator ~/.claude/skills/epic-coordinator
```

Optional companions:

```
/plugin marketplace add tranquocthong/spec-flow
/plugin install sf@claude-spec-flow
```

Also install agent-talk from https://github.com/tranquocthong/claude-agent-talks.

## Use

1. Open one session per phase plus one coordinator session.
2. In the coordinator session, tell Claude: "act as coordinator for epic `<slug>`" (the skill triggers on
   that, or run `/epic-coordinator <slug> setup`).
3. The coordinator writes the epic record and a brief for each worker. Paste each brief into its worker
   session and pair the sessions it asks for.
4. From then on, talk to the coordinator. Send it your review feedback, questions and decisions. It routes
   them to the workers and reports back with what it verified.

## Files

| File | Content |
|---|---|
| `SKILL.md` | Roles, ground rules, channels, lifecycle, reporting, pitfalls |
| `references/spec-flow.md` | Stage-by-stage mapping to `/sf:*` commands and sf-specific traps |
| `references/epic-template.md` | Epic record: phases, roster, delivery rules, decisions, dependencies |
| `references/worker-brief.md` | What to paste into each worker session (and the E2E owner) |
| `references/message-templates.md` | Handoff, change notice, review notice, review finding, re-run, status |
| `references/checks.md` | Bash snippets for verification and the final E2E gap checklist |

## License

MIT
