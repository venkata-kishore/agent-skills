# Git: beyond commit and push

Read on demand, when `~/.agents/AGENTS.md` points here.

- Fetch and check every branch, including other developers', before deciding where things stand. Never build on or merge another developer's branch without me.
- Delete only local branches, and only after `git branch --contains` shows them merged. The exception is a temporary branch a skill created for its own helpers, which that skill removes when it is done. Never delete remote branches; report stale ones.
- git's stash is shared by every worktree of a repo: in a shared repo, set changes aside as a patch file or a temporary commit, never a bare stash.
- Restore a file with a path git really resolves (`git show HEAD:./rel/path`) and check the result is not empty.
