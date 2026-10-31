# React

Function components and hooks, plus Next.js and a Vite setup. Apply with `SKILL.md`. Hooks are the analogue of Vue composables. House defaults carry over: JavaScript (`.jsx`), Bun, oxlint, oxfmt, explicit imports, `~/` alias, shared constants package.

## Components

- Function components only, one per file, `PascalCase.jsx`. `App*` prefix for layout chrome.
- Composition over configuration: small components combined through `children`, not one component with fifteen boolean props.
- Split container (data, orchestration) from presentational (props in, markup out) only when it earns its keep.
- Keep state as low as possible; lift to the nearest common ancestor when shared.
- Order inside a component: imports, props (one destructured object), hooks (context or store, state, derived, effects), handlers, early returns for loading, error and empty, JSX.

## Hooks

- Call hooks at the top level only, from components or custom hooks.
- `useEffect` synchronizes with external systems. It does not derive state: compute during render or with `useMemo`.
- Correct dependency arrays; do not disable the lint rule. Clean up subscriptions, timers and `AbortController` in the returned function.
- Custom hooks `useXxx` are the reuse mechanism; file name equals export; return an object.

## State and data

- Local UI state: `useState` / `useReducer`. Keep inputs controlled. Never mirror derived values into state.
- Context for low-frequency values (theme, current user); keep contexts small.
- Cross-feature client state: one store per domain (for example Zustand), with the same rules as `SKILL.md`: refresh on defined events, reset when the signed-in user changes, check the sign-in after an `await` before writing.
- Components and hooks call store actions only, never the HTTP client or `fetch`.
- Server data may use a query library (TanStack Query) only if the queries and mutations still go through the one data layer that owns the API client and the reset-on-user-change rule, and the query cache is cleared when the signed-in user changes. Do not hand-roll caching in `useEffect`.
- Components render loading, error and empty states. A failed load is an error state, not an empty list.

```jsx
const [error, setError] = useState("");
async function handleLogin() {
  try {
    await authStore.login(form);
  } catch (err) {
    setError(messageForError(err));
  }
}
// {error && <p id="login-error">{error}</p>}
```

Messages come from `messageForError` and the server `errorCode` (`references/api-client.md`). The client lives in `hooks/useApi.js` and is used by the data layer only.

## Performance

- Stable keys: unique id, not the index, for dynamic lists.
- `React.memo` only for components that re-render often with equal props and render non-trivially. `useMemo` for real computation; `useCallback` for callbacks given to memoized children or used as effect deps. With the React Compiler, do not over-memoize.
- Code-split routes and heavy components with `lazy` + `Suspense`.
- No inline objects, arrays or functions as props to memoized children.

## Security

JSX auto-escapes `{userInput}`. The hole is `dangerouslySetInnerHTML`: DOMPurify-sanitized content only. `NEXT_PUBLIC_*` and `VITE_*` are public.

## Next.js (App Router)

- Server Components by default. Add `"use client"` only for state, effects, browser APIs or event handlers, as low in the tree as possible.
- Secrets and data access stay in Server Components, Route Handlers and Server Actions. Sanitize what crosses to the client. Server-side fetching is the Next default; the Vue SPA rule "fetch in `onMounted`" does not carry over.
- Server Components and Route Handlers are the data layer for server paths. Client components still call store actions only.

## Vite setup (client-only SPA)

Same toolchain as `SKILL.md`. A router library (`react-router` or a file-based router), routes code-split with `lazy` + `Suspense`. Reuse the `ky` client and frozen `config/env.js`, branching on `import.meta.env.DEV` (`import.meta.dev` is a Nuxt global, not Vite).

## Folder layout

`pages/` or `app/` (Next App Router), `components/<domain>/`, `hooks/`, `stores/` (optional), `config/`, `assets/`. Components `PascalCase.jsx`, hooks `useXxx.js`.

## Avoid

Class components; derived state in effects; index keys on dynamic lists; god components and deep prop-drilling; direct state mutation; business logic in JSX; over-memoizing; secrets in public env vars; `dangerouslySetInnerHTML` on user data.
