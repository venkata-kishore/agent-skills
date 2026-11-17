# Environment and dev resources

Read on demand, when `~/.agents/AGENTS.md` points here.

- Do not assume tools I have not installed. Check the machine's existing setup before asking me to act.
- Use your own dev resources (your own database container or port), not my other projects'. Ask before resetting dev data; a one-time permission does not extend to later resets.
- Host ports: while a project is only being scaffolded or tried out, its framework's default ports are fine. Once it is a real project, every one of its own dev servers (frontend, backend, the app container; Nuxt, Node, Java or whatever it runs) moves to a small block in the 88xx series (for example 8801 the app, 8802 its backend), so several projects can run at once. Third-party services (databases, caches, mail catchers, object stores) keep their usual ports. A session sees only its own project, so before a project takes a block, check what is listening (`ss -ltn`, `docker ps`) and ask me which block to use; record the chosen ports in the project's committed config and README.
- Do not run expensive environment steps (image builds) before the work needs them.
- Build caches: remove one-off build output (binaries and caches made for a single piece of work, such as those under a scratch folder) when that work is finished. Clean a regular build cache only once it has grown past about 1 GB; check its size first (for Go: `du -sh "$(go env GOCACHE)"`, then `go clean -cache`). Leave dependency caches (Go's module cache and the like) unless I ask.
- Before production, breaking changes and rebaselines are fine. Prefer the cleaner design over compatibility shims.
