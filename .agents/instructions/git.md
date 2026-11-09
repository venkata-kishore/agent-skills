# Git: beyond commit and push

Read on demand, when `~/.agents/AGENTS.md` points here.

- Fetch and check every branch, including other developers', before deciding where things stand. Never build on or merge another developer's branch without me.
- Delete a branch, local and remote, once it is merged; keep no unneeded branches. Check first: after a normal merge `git branch --contains` lists it, after a squash merge it never will, so check that its PR is merged and the branch tip is the PR's merged head (`gh pr view <n> --json state,headRefOid`). A branch with commits after that, or another developer's branch, needs my go. A temporary branch a skill created for its own helpers is removed by that skill when it is done.
- git's stash is shared by every worktree of a repo: in a shared repo, set changes aside as a patch file or a temporary commit, never a bare stash.
- Restore a file with a path git really resolves (`git show HEAD:./rel/path`) and check the result is not empty.
