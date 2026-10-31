---
name: delegating-to-codex
description: For coding agents other than Codex. Delegate implementation work to OpenAI Codex CLI workers (gpt-6.1-sol, high effort), in parallel git worktrees when the work splits, then review, integrate and verify the result, or run a read-only brainstorm or plan. Use only when the user explicitly asks in chat to use Codex, e.g. "use codex for X" or "have codex do X". Never on your own initiative.
compatibility: Not for Codex itself; disable it there with a [[skills.config]] entry (enabled = false) in ~/.codex/config.toml. Needs a shell, git, jq and a logged-in Codex CLI. Sends repository content to OpenAI.
metadata:
  verified-with: "codex-cli 0.166.0, git 2.52.0"
  verified-on: "2026-10-31"
---

# Codex orchestration

You orchestrate; Codex workers write the code. You keep the decisions, the review and the final say. The user's request to use Codex approves sending this repo to OpenAI for that task; a new task needs a new request.

## Step 1: Check the CLI

1. Run `codex exec --help` and `codex exec resume --help`.
2. Confirm the flags used below still exist: `-C`, `-m`, `-c`, `-s`, `--json`, `-o`, `-` (prompt from stdin), and resume by id or `--last`. If one was renamed, use the new spelling. If one is gone, tell the user.
3. If the model is rejected, ask; do not switch models. Never use `--dangerously-bypass-approvals-and-sandbox`.

## Step 2: Plan

1. `R=$(git rev-parse --show-toplevel)` and `BASE=$(git -C "$R" rev-parse HEAD)`. Not a git repo: stop and ask.
2. `MODEL=gpt-6.1-sol`. Use `$MODEL` in every command below.
3. If `git -C "$R" status --porcelain` is not empty, ask the user to commit or stash first. Workers start from `HEAD` and must not mix with their work.
4. Split the task into units. Units with no shared files and no ordering run in parallel, at most 4 at once. Dependent units go in a later wave. Tell the user the split.
5. Choose a work directory `W` outside the repo: your scratch directory or `mktemp -d`.

## Step 3: Isolate and brief each unit

1. Create its worktree: `git -C "$R" worktree add -b codex/<slug> "$W/wt/<slug>" "$BASE"`, then `mkdir -p "$W/runs/<slug>"`. In later waves, use `codex/integrate` in place of `"$BASE"`.
2. Install dependency folders (`node_modules`, `.venv`) in the worktree if the build needs them.
3. Write `$W/runs/<slug>/brief.md`. Codex reads AGENTS.md but no other agent's instruction files, so include (inline the content of any file the repo's AGENTS.md imports with @, since Codex does not expand imports):
   - Goal and acceptance criteria.
   - Files in scope. Files owned by other units are off-limits.
   - Project and user rules for code: match the surrounding style, use the formatter, make the smallest change, follow the project's test policy, add no dependencies unless needed, never mention other projects, and write no comments that restate the code (see `writing-code-comments`).
   - Limits: no git state changes, stay in this directory, no destructive commands, no other agents.
   - Finish with a summary of changes, commands run and open issues.

## Step 4: Run the wave

Start one block per unit, then wait. Runs take minutes; use your agent's background mode if it has one.

```bash
( codex exec -C "$W/wt/<slug>" -m "$MODEL" -c 'model_reasoning_effort="high"' -s workspace-write \
    --json -o "$W/runs/<slug>/last.md" - < "$W/runs/<slug>/brief.md" \
    > "$W/runs/<slug>/events.jsonl" 2> "$W/runs/<slug>/stderr.log"
  echo $? > "$W/runs/<slug>/exit" ) &
wait
```

A non-zero `exit`, or a `turn.failed` event, means the run failed. Read `stderr.log`, report it, and retry at most once.

## Step 5: Review each unit

Start as soon as a unit finishes.

1. Read `last.md`, then the full diff: `git -C "$W/wt/<slug>" add -A && git -C "$W/wt/<slug>" diff --cached HEAD`.
2. Check scope, conventions and test quality. Look for secrets and debug leftovers.
3. Run the formatter, build and tests in the worktree.
4. Fix small issues yourself. For bigger ones, write `feedback.md` and resume the worker, at most 2 rounds:

```bash
cd "$W/wt/<slug>" && codex exec resume <thread_id> -m "$MODEL" -c 'model_reasoning_effort="high"' \
  -c 'sandbox_mode="workspace-write"' --json -o "$W/runs/<slug>/last.md" - < "$W/runs/<slug>/feedback.md" \
  > "$W/runs/<slug>/events-2.jsonl" 2> "$W/runs/<slug>/stderr-2.log"
```

The thread id is `jq -r 'select(.type=="thread.started") | .thread_id' "$W/runs/<slug>/events.jsonl" | head -1`. Or use `--last` from that worktree.

## Step 6: Integrate

1. Once: `git -C "$R" worktree add -b codex/integrate "$W/wt/integrate" "$BASE"`.
2. For each unit, export a patch and apply it to the integration worktree. Resolve conflicts yourself.

```bash
git -C "$W/wt/<slug>" diff --cached --binary HEAD > "$W/runs/<slug>/patch.diff"
git -C "$W/wt/integrate" apply --3way "$W/runs/<slug>/patch.diff"
```

3. Run the full formatter, build and tests in the integration worktree.
4. If there is another wave, commit this one there, then go back to step 3:
   `git -C "$W/wt/integrate" -c user.name=codex-orchestrator -c user.email=codex@localhost commit -q --no-verify -m "wave N"`
   (`--no-verify`: this is a throwaway commit in a temporary worktree deleted in Step 7, and hooks would run against half-integrated code.)
5. Apply the result to the user's tree without staging. Never use `--3way`, `--index` or `--cached` on the user's tree.

```bash
git -C "$W/wt/integrate" add -A && git -C "$W/wt/integrate" diff --cached --binary "$BASE" > "$W/combined.diff"
git -C "$R" apply --check "$W/combined.diff" && git -C "$R" apply "$W/combined.diff"
```

## Step 7: Clean up and report

1. For every worktree you created, including `integrate`: `git -C "$R" worktree remove --force "$W/wt/<slug>" && git -C "$R" branch -D codex/<slug>`. Delete only the `codex/*` branches this skill created.
2. Report: what each unit's worker did and what you fixed, any conflicts, and build and test results (with output on failure). Leave the changes uncommitted unless the user asks.

## Brainstorming and planning

When the user wants ideas or a plan and no code changes, skip the worktrees and set `R` and `W` as in Step 2 first, then run one read-only session:

```bash
codex exec -C "$R" -m "$MODEL" -c 'model_reasoning_effort="high"' -s read-only \
  -o "$W/last.md" - < "$W/brief.md"
```

- If `-C` points outside a git repo, add `--skip-git-repo-check`.
- If the user names a model, check it appears in `~/.codex/models_cache.json` before using it in place of `$MODEL`. If it is not listed, ask.
