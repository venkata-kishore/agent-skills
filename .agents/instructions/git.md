# Git: beyond commit and push

Read on demand, when `~/.agents/AGENTS.md` points here.

- Fetch and check every branch, including other developers', before deciding where things stand. Never build on or merge another developer's branch without me.
- Delete a branch, local and remote, once it is merged; keep no unneeded branches. Check first: after a normal merge `git branch --contains` lists it, after a squash merge it never will, so check that its PR is merged and the branch tip is the PR's merged head (`gh pr view <n> --json state,headRefOid`). A branch with commits after that, or another developer's branch, needs my go. A temporary branch a skill created for its own helpers is removed by that skill when it is done.
- git's stash is shared by every worktree of a repo: in a shared repo, set changes aside as a patch file or a temporary commit, never a bare stash.
- Restore a file with a path git really resolves (`git show HEAD:./rel/path`) and check the result is not empty.

## Preparing a release

- Set the new version everywhere it is written: build files, lock files, install snippets in the README and the docs. Search for the old version string to find every place.
- Date the changelog section and check it against the commits since the last release.
- Check every pinned version against the registry and update what is behind.
- Check the names, routes, metrics and defaults the docs state against the code.
- Run the full gates on the release commit itself, not on an earlier one.
- Write the PR title and description with the `preparing-changes-for-merge` skill: a title can be what starts the release.
- Open the release PR only on my go.
