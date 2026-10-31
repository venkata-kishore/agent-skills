---
name: maintaining-project-records
description: Use when a repo has an .agents/ folder (decisions, todo, history, questions, learnings, worklog, handoff) and a session starts cold or resumes someone else's work, a session ends, several agent sessions run in parallel, work is deferred, a choice is made, an owner question is answered, a sub-project closes, or a gotcha is found; also when setting up that framework in a new repo or replacing CLAUDE.md with AGENTS.md.
---

# Maintaining project records (.agents/ framework)

## Overview
Project memory lives in the repo, not in chat: a thin root `AGENTS.md` plus `.agents/` records,
one home per kind of fact, updated in the same commit as the change they describe. If the repo's
`.agents/AGENTS.md` "Project records" table differs from this skill, the repo wins.

## Layout
```
AGENTS.md            thin entry: imports + read order. No rules here.
.agents/AGENTS.md    conventions, hard rules, commands, review checklist, records table
.agents/decisions.md D-numbered table: decision, status (open|resolved), why / rejected option
.agents/todo.md      open work items (WI rows) + deferred findings + "later" ideas
.agents/history.md   completed WI rows only
.agents/questions.md items blocked on the owner
.agents/learnings.md gotchas (trap -> what to do)
.agents/worklog.md   dated entries, newest first, last 14 days only
.agents/handoff.md   current state + workstreams table, rewritten on every state change
```
Root `AGENTS.md` carries `@.agents/AGENTS.md` and `@.agents/handoff.md` lines (Claude Code
expands them) plus a plain sentence telling other agents to open `.agents/AGENTS.md`.
Reserved names, leave unused until needed: `.agents/memories/`, `skills/`, `agents/`, `tasks/`,
`mcp-settings.json`. Never create `.agents/agents.md` (collides with AGENTS.md on Windows).

## Where each fact goes (exactly one home)
| Event | Record |
|---|---|
| Chose X over Y (design, library, approach) | decisions.md: new D row with the rejected option and why |
| Owner answers a question | decisions.md D row (or todo line if it is just work); delete it from questions.md |
| Need owner input | questions.md entry: context, a recommendation, alternatives with why/why not (refactor size, time, and trivial bookkeeping are never cons; no filler cons). Ask in the session that raised it |
| Deferred something / finding not fixed | todo.md line only. A deferral is not a decision: no D row |
| Work item finished | move its row from todo.md to history.md |
| Trap that cost time | learnings.md; once it is a rule, move it to .agents/AGENTS.md |
| New durable convention | .agents/AGENTS.md |
| Workstream assigned / commits / done / blocked | rewrite its handoff.md row now (sessions can die) |
| Session ends | worklog.md entry + prune + rewrite handoff.md |

## Cold start (any agent, no prior chat)
1. Read root AGENTS.md read order, then `.agents/handoff.md`.
2. Reconcile with git (`git status`, `git log --oneline -5`, `git worktree list`, `git branch -vv`).
   Git wins; fix handoff first and say what was stale.
3. Take the stream the owner named; with none named you are the lead. Never touch another
   stream's claimed files while it may be active: ask the owner.
Anything a fresh agent needs must be in the repo. Never leave a rule only in chat or in an
agent-specific memory; move it into `.agents/AGENTS.md`.

## Parallel sessions
handoff.md holds a workstreams table: stream, assigned to, where (branch/worktree), claimed
files, state, next step. The lead assigns work (owner OK), collects status (done / in flight /
blockers / files), and keeps all records current; workers share the main checkout by default
(worktrees only for a specific need or on the owner's ask), claim paths via the lead before
editing, touch only claimed files, and report instead of editing `.agents/`. Commits list explicit paths. One
strongest-model review per finished chunk or feature, before commit/merge, not per commit.

## End-of-session routine
1. Add `## YYYY-MM-DD · <branch>` at the top of worklog.md, 1-5 bullets citing D/WI ids.
2. Prune: cutoff = today minus 14 days. Delete every entry dated before the cutoff, in the same
   edit (2026-10-12 keeps 2026-09-28 onward). Git keeps the rest.
3. Rewrite handoff.md: branch, workstreams table, pending owner calls, blockers.
4. Commit record changes with the code they describe.

## Sub-project closeout
Specs and plans (`docs/specs/`, `docs/plans/`) are working documents. When the last task lands:
fold durable facts into `.agents/` (conventions, D rows, leftovers to todo/questions), update the
roadmap and PROJECT docs, then delete the spec and plan. Code comments and docs cite D ids, never
spec sections or plan task numbers.

## Bootstrapping a new repo
Create the layout above; move existing rules into `.agents/AGENTS.md` with the records table and
these lifecycle rules; move existing decision/backlog files in with `git mv`. Remove `CLAUDE.md`:
Claude Code (v2.1.277+) reads root AGENTS.md only when no `CLAUDE.md`, `.claude/CLAUDE.md`, or
`CLAUDE.local.md` exists in the working directory or above. Start sessions at the repo root.

## Common mistakes
- Logging a deferral as a D row as well as a todo line (two homes).
- Adding a worklog entry without pruning old ones.
- Appending to handoff.md instead of rewriting it, or updating it only at session end.
- Keeping a rule only in chat or agent memory, so a fresh agent cannot see it.
- Staging another stream's files from a shared checkout.
- Leaving an answered question in questions.md.
- Keeping a finished spec "for reference" or citing it from code.
