# React Patterns

Generic React (function components + hooks), plus a Next.js section and a
Vite/VoidZero setup. Apply **with** the universal layer in `SKILL.md` (design
principles, complexity, runtime performance, security, naming). House defaults
carry over: **JavaScript by default** (`.jsx`; TypeScript opt-in), Bun, oxlint,
oxfmt, explicit imports, `~/` alias, `@pkg/common` for shared constants.

React is the same shape as the Vue side with different primitives: **hooks are
the analogue of composables**, and the "store owns data, page owns loading/error
UI" rule becomes "container owns data, presentational component renders props".

## Components

- **Function components only** — no class components.
- One component per file. PascalCase file name matching the component
  (`ContentCard.jsx`, `BillingHistoryTable.jsx`). Same `App*` prefix for layout
  chrome and domain prefix in subfolders as the Vue side.
- **Composition over configuration** — small components combined via `children`
  beat one component with fifteen boolean props.
- **Container / presentational split when it earns its keep** — keep
  data-fetching and orchestration separate from pure components that take props
  and render. Don't force the split on trivial UI.
- **Lift state only as high as needed** — co-locate with the component that uses
  it; lift to the nearest common ancestor when shared. Global state is a last
  resort, not a default.

Standard ordering inside a component (mirrors the `<script setup>` order in
`vue.md`):

```jsx
// 1. Imports
import { useState, useMemo, useEffect } from "react";
import { useAuth } from "~/hooks/useAuth.js";
import { SUBSCRIPTION_STATUS } from "@pkg/common";

// 2. Component declaration + props (one object, destructured)
export default function UserPanel({ userId, onUpdated }) {
  // 3. Hooks: context/store → state → derived → effects
  const auth = useAuth();
  const [loading, setLoading] = useState(false);
  const [items, setItems] = useState([]);
  const hasItems = useMemo(() => items.length > 0, [items]);

  // 4. Handlers
  async function handleSubmit() { /* ... */ }

  // 5. Early returns (guard clauses) for loading / error / empty
  if (loading) return <Spinner />;

  // 6. JSX
  return <section>{/* ... */}</section>;
}
```

## Hooks

- **Rules of Hooks:** call hooks at the top level only — never inside
  conditions, loops, or nested functions, and only from components or custom
  hooks.
- **`useEffect` synchronizes with external systems; it does not derive state.**
  If a value can be computed from props/state, compute it during render (or with
  `useMemo`) — do not mirror it into state via an effect.
- **Correct dependency arrays.** Missing deps cause stale closures; extra deps
  cause loops. Don't disable the lint rule to "fix" it.
- **Clean up effects** (subscriptions, timers, listeners, `AbortController`) in
  the returned cleanup function — the analogue of `onUnmounted`.
- **Custom hooks (`useXxx`)** extract reusable stateful logic — React's primary
  reuse mechanism, the analogue of Vue composables. File name matches the
  exported hook; always return an object (or a stable tuple), never bare state.

```js
// hooks/useItems.js — reuses the shared ky client (see SKILL.md API Client)
import { useState, useEffect } from "react";
import { useApi } from "~/hooks/useApi.js";

export function useItems(params) {
  const api = useApi();
  const [items, setItems] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    api("api/items", { query: params, signal: controller.signal })
      .then((data) => setItems(data.items))
      .catch(() => setItems([])) // silent background fetch → empty state
      .finally(() => setLoading(false));
    return () => controller.abort(); // cleanup
  }, [params]);

  return { items, loading };
}
```

> `useApi()` returns a stable module-level singleton, so it's safe to omit from
> the dependency array (and harmless to include). Don't silence the
> `exhaustive-deps` lint rule to clear a deps warning — fix the dependency.

## State & Data

- **Local UI state:** `useState` / `useReducer`.
- **Derived values:** compute in render or `useMemo` — never duplicate into
  state. (Same rule as preferring `computed` over a mirrored `ref`.)
- **Server data:** use a query library (TanStack Query) — don't hand-roll
  caching/loading/error in `useEffect`. The store-owns-data idea becomes the
  query cache owning server state.
- **Global client state:** keep it minimal. `Context` for low-frequency values
  (theme, current user) — but Context re-renders all consumers, so keep contexts
  small and split by concern. For cross-feature mutable state reach for a
  lightweight store (Zustand), **one store per domain**, mirroring the Pinia
  rule on the Vue side.
- Keep inputs controlled; derive, don't duplicate.

## API Client & Error Handling

The singleton `ky` client in `SKILL.md` (API Client Design) is
framework-agnostic. In a React project it lives in `hooks/useApi.js` and is
consumed from a custom hook or a query function — same `useApi()`, same
`{ method, body, query }` call shape, same `err.data` error body.

Reuse the house error strategy verbatim:

- **Error message extraction:**
  `err?.data?.error || err?.data?.message || err.message || "Something went wrong."`
- **Three patterns:** inline error state for forms; toast for CRUD actions;
  silent fallback to an empty state for background fetches.

```jsx
// Inline error state — auth forms / single-action pages
const [error, setError] = useState("");
async function handleLogin() {
  try {
    await auth.login(form);
  } catch (err) {
    setError(err?.data?.error || "Login failed.");
  }
}
// {error && <Alert color="error" description={error} />}
```

## Performance

- **Stable keys** in lists — a unique id, never the array index for
  dynamic/reorderable lists (index keys cause subtle state bugs). Same rule as
  `:key` on `v-for`.
- **Memoize deliberately, not reflexively:**
  - `React.memo` a component only when it re-renders often with the same props
    and rendering is non-trivial.
  - `useMemo` for derived values that actually transform or combine data (not
    for a value returned unchanged); `useCallback` for callbacks passed to
    memoized children or used as effect deps.
  - If the project uses the **React Compiler**, most of this is automatic —
    don't over-memoize.
- **Code-split** routes and heavy components with `lazy` + `Suspense`.
- Don't create new objects/arrays/functions inline as props to memoized children
  (it defeats the memo). Keep expensive work out of render.

## Security

Universal security rules live in `SKILL.md` (URL-scheme validation, no secrets
in the bundle); this is the React-specific item.

- **JSX auto-escapes** — interpolating `{userInput}` is safe. The hole is
  **`dangerouslySetInnerHTML`**: use it only with DOMPurify-sanitized content,
  never raw user data.

## File / Folder Conventions

Same structure as `SKILL.md` Project Structure, with React's naming:

```
frontend/
  pages/ or app/      # routes (app/ for Next App Router)
  components/
    <domain>/         # feature-specific (3+ components = own folder)
  hooks/              # useXxx.js  (React's composables/)
  stores/             # one per domain (Zustand) — optional
  config/
  assets/
```

- Components `PascalCase.jsx`; hooks `useXxx.js`; co-locate `*.test.jsx`.
- Explicit imports, `~/` alias, `@pkg/common` for shared constants — never
  hardcode magic numbers (the negative/zero/positive status convention applies).

## Next.js

For App-Router projects:

- **Server Components by default.** Add `"use client"` only when a component
  needs state, effects, browser APIs, or event handlers. Push client boundaries
  as low in the tree as possible.
- **Secrets and data access stay server-side** — Server Components, Route
  Handlers, and Server Actions. `NEXT_PUBLIC_*` is public; everything else is
  not. Sanitize anything crossing the server→client boundary.
- **Data fetching:** fetch in Server Components or Route Handlers; for
  client-side server state, use TanStack Query.
- The Vue side runs SPA mode (`ssr: false`), so its "always fetch in
  `onMounted`" rule is Vue-SPA-specific and does **not** carry over — in Next,
  prefer fetching on the server.

## Vite + VoidZero setup (framework-free React)

For a client-only React SPA (the React analogue of the Nuxt-SPA setup). Same
toolchain as `SKILL.md` **Runtime & Tooling** — Bun, Vite, oxlint, oxfmt, run via
`bun run dev|lint|fmt`. VoidZero is the Vite + Oxc (`oxlint`/`oxfmt`) toolchain.

- **JavaScript by default** (`.jsx`); opt into TypeScript per project.
- Routing via a router library (`react-router` or a file-based router);
  code-split routes with `lazy` + `Suspense`.
- Reuse the `ky` client and the frozen `config/env.js` pattern from `SKILL.md` —
  with plain Vite, branch on `import.meta.env.DEV` (the Vue side's
  `import.meta.dev` is a Nuxt global, not standard Vite).

## Patterns to Avoid

- Class components in new code.
- Derived state stored via `useEffect` instead of computed in render.
- Index keys on dynamic/reorderable lists.
- Giant "god" components; prop-drilling five levels deep (lift to context/store).
- Mutating state directly — always set a new reference.
- Business logic inside JSX (belongs in hooks/handlers/stores).
- Over-memoizing everything; memoizing cheap values.
- Hand-rolling fetch caching in `useEffect` when a query library is available.
- Secrets in `NEXT_PUBLIC_*` / `VITE_*`; `dangerouslySetInnerHTML` on user data.

## Quick Reference

| Need | Pattern |
|------|---------|
| New component | `PascalCase.jsx` in `components/` or `components/<domain>/` |
| New hook | `hooks/useXxx.js` — returns an object with state + methods |
| Reusable logic | Custom hook (`useXxx`), not an HOC or render prop |
| Server data | Query library (TanStack Query), not `useEffect` caching |
| Global state | `Context` (small, by concern) or Zustand store per domain |
| API call | `useApi()` from `hooks/useApi.js` → `api(url, { method, body, query })` |
| Show error | Inline state for forms, toast for actions |
| Derived value | Compute in render / `useMemo` — never mirror into state |
| Code-split | `lazy(() => import(...))` + `<Suspense>` |
| Raw HTML | DOMPurify-sanitized only, via `dangerouslySetInnerHTML` |
