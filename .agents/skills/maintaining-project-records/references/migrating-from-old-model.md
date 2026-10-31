# Migrating from the old records model

Old model: decision table with ids, worklog, history, learnings files, work-item ids. The new
model replaces it; do not keep both. Do this on its own branch, in one commit per step.

1. Inventory: list `.agents/*.md`, `docs/specs/`, `docs/plans/`, and every place that cites an id
   (`grep` for the id pattern in code, comments, docs, tests).
2. Decisions: for each resolved decision, amend the owning area's requirements document: the
   behaviour, plus the reason, accepted cost and rejected option under "Why it is this way".
   Current behaviour only. Create the document if the area has none. Open decisions become
   questions.md entries. Then delete the decision table.
3. Learnings: a gotcha that is now a rule goes to `.agents/AGENTS.md` (and the review checklist);
   one tied to a spot goes to a code comment; the rest is dropped. Delete the file.
4. Worklog and history: delete both. Git history and commit messages hold them.
5. Todo: strip work-item ids; keep open items only. Move finished rows nowhere.
6. Replace every id in code comments and docs with what the thing is, in plain words.
7. Rewrite handoff.md in the new shape and fix root AGENTS.md imports.
8. Check: grep finds no old ids and no links to deleted files; run the line-width/format check;
   quiz a fresh agent from the repo alone.
