---
name: frontend-conventions
description: "Use when building, modifying or reviewing frontend code in Vue, Nuxt or React: component structure, composables and hooks, stores and state, API client, errors, auth, permissions, forms, design tokens and Tailwind, accessibility, performance, XSS and input validation, naming, imports, tests, or adding pages and features."
---

# Frontend Conventions

## Tooling

- Bun for runtime and packages (never npm, yarn or node). Vite via the framework. Tailwind CSS v4
  (CSS-first, `@theme`). oxlint for linting, oxfmt for formatting (printWidth 100).
- Commands: `bun install`, `bun run dev`, `bun run lint`, `bun run fmt`.
- JavaScript by default (`.js`, `.vue`, `.jsx`); TypeScript only when the project chooses it.
- Dates: Luxon `DateTime` for all parsing, formatting, math and zones. Never native `Date`.
- Bun workspaces for shared packages.

## References

- `references/vue.md`: Vue composition, Pinia setup stores, props and emits, composables, page loading.
- `references/nuxt.md`: Nuxt routing, layouts, middleware, runtime config, Nuxt UI, icons, scripts.
- `references/react.md`: React components, hooks, state and data, Next.js, Vite setup.
- `references/tailwind.md`: token setup, base and component classes, breakpoints, dark mode, contrast.
- `references/api-client.md`: the `ky` client, 401 refresh, error codes, the message helper.
- `references/auth.md`: open when wiring sign-in, tokens, refresh, sign-out or route guards.
- `references/forms.md`: open when building or validating a form or showing field errors.

Open the framework file for the project's stack. Open the others when you touch that topic.
Sign-in changes must reset per-user stores (see Data and state); the rest of auth is in `auth.md`.

## Design principles

- One responsibility per unit. If you need "and" to describe it, split it.
- Separate UI, business logic, data access and side effects. Stores own data, pages own loading and
  error UI. Compose small units; no class hierarchies.
- Remove duplication at the third occurrence, not the first.
- Build the simplest thing for today's requirement; ask the owner before building for a likely
  future need.
- Pure by default; push I/O, DOM and network to the edges (stores, composables, effects).
- No magic numbers: use the shared constants package.
- Fail loud. Validate at boundaries. Never swallow errors with an empty `catch {}`.

## When to split

Decide by what the code does, not its length. Split when a unit:

- has more than one responsibility, or mixes UI, data fetching, business rules and formatting;
- nests 3 or more levels of `if`/`for`/`try` (use guard clauses or extract);
- takes 3 or more positional parameters (pass an options object; no boolean flags like
  `doThing(true, false)`);
- cannot be named cleanly, or a block needs a comment saying what it does (extract it);
- is hard to test in isolation.

## Performance

Measure before optimizing.

- Debounce high-frequency handlers. Lazy-load and code-split what first paint does not need.
- Memoize real computation, not values returned unchanged. Paginate large data; virtualize long
  lists. Use `Map` and `Set` for lookups.

## Security

- Validate the shape of every external value (input, route params, API responses, SDK callbacks).
- Render text as text. Raw HTML only after a vetted sanitizer (DOMPurify).
- Validate URL schemes built from user data before binding to `href` or `src` (block `javascript:`).
- Everything in the bundle is public. Secrets stay server-side. `VITE_*`, `NUXT_PUBLIC_*` and
  `NEXT_PUBLIC_*` ship to the browser.
- Never `eval`, `new Function` or `setTimeout(string)` on user input.
- Hiding UI by permission is cosmetic; the server is the boundary. When access rules change, change
  the server guard and the UI together.

## Accessibility (WCAG 2.2 AA)

- Semantic HTML first; ARIA only where native semantics fall short.
- Everything interactive works by keyboard, with a visible focus state and logical tab order.
- Every control has an accessible name: a `label`, or `aria-label` for icon-only buttons. Images
  have `alt`.
- A field error that appears after an action is associated with its field and announced; no blanket
  `role="alert"` (details in `references/forms.md`).
- Never convey state by colour alone.

## Naming

- `camelCase` variables; verb-first functions (`fetchItems`); boolean predicates (`isLoading`).
- `UPPER_SNAKE_CASE` for module constants; shared ones live in the shared package.
- `PascalCase` component files. `useXxx.js` for composables and hooks, file name equals export.
  Stores are `<domain>.js`.
- TypeScript projects: `PascalCase` types, no `I` prefix, `unknown` plus narrowing over `any`.
- No abbreviations beyond `id`, `url`, `db`. One concept per name (not `data` or `temp` everywhere).
- Delete dead code and commented-out blocks.

## Structure and imports

```
frontend/
  pages/  components/<domain>/  stores/  composables/ (or hooks/)  layouts/
  middleware/  config/  assets/css/main.css  assets/images/  public/
```

- One store per domain. A feature with 3 or more components gets its own `components/<domain>/`
  folder. `components/ui/` holds thin wrappers over third-party SDKs.
- Env config lives in a frozen `config/env.js`, imported only by the API client.
- Always explicit imports, never auto-imports for your own code. `~/` is the project root alias.
  Shared constants come from the shared workspace package.

## Design tokens

- Every design value (colour, radius, font, spacing, breakpoint) comes from a token. No literal hex
  or `oklch()`, no `theme()`, no raw palette utilities like `bg-blue-600` in markup.
- Each semantic colour role gets its own token (`link`, `link-hover`, `success`, `error`), so theming
  and dark mode change one variable.
- Contrast checks use the exact palette step each role renders with, per mode. Recompute them when a
  theme redraws only some steps.
- Base elements in `@layer base`; recurring elements get one `@layer components` class.
- Defer frontend-wide passes such as dark mode until enough UI exists, then do one pass.
- Setup and examples: `references/tailwind.md`.

## Data and state

- Pages and components call store actions only. They never import the HTTP client or call `fetch`.
  Only stores (or the data layer behind them) talk to the client.
- One store per domain, written as a setup store: state, derived, actions, return all.
- Refresh a store on defined events (mount of the owning view, a mutation, a sign-in change, an
  explicit refresh), not on every navigation.
- Every per-user store resets when the signed-in user changes (sign-out, user switch, token loss).
  Cross-store resets are allowed (auth resets the others).
- An action that writes state after an `await` first checks the signed-in user did not change
  while it waited; if it did, drop the result.
- Every list or detail view handles loading, error and empty states as three different states.
  A failed fetch is an error state with a retry, never an empty list.

## Errors

- The UI chooses its message from the server's `errorCode`, through one pure helper
  (`messageForError(err)`), with a generic fallback for unknown codes. Never match message text and
  never show the raw body, `err.message` or a stack trace.
- The backend must return a stable `errorCode` on every error (requirement, not an assumption).
  A refusal the UI needs to word differently gets its own `errorCode`.
- Where the message goes: an inline error next to a form, a toast for a dashboard action, an error
  state in the view for a failed load.
- Helper, client and example: `references/api-client.md`.

## Shared constants and statuses

- Constants come from the shared package, which is generated from the backend. Frozen objects, never
  TS enums. Never hardcode a status number.
- Status scheme (same as the database): `0` base marker, `1` draft, `2` ended normally, `3`
  archived, live states above `3`, negative values removable.
- Use named predicates from the shared constants ("is live", "is removable"), not ad hoc comparisons.
  Role and permission flags come from the same package.

## Tests

Ask the owner which test levels to write unless the project's conventions already say. When unit
tests are wanted, the place for them is pure helpers that hold UI decisions (error code to message,
status predicates, validators). Do not scaffold component or e2e tests unasked.

## Common mistakes

- HTTP client or `fetch` called from a page or component.
- `err.message`, the raw body or matched message text shown; a failed fetch turned into an empty list.
- A per-user store surviving a user change; a write after `await` with no sign-in check.
- Hardcoded status numbers, literal colours or palette utilities; `VITE_*` used directly.
- Business logic in components; auto-imports for own code; native `Date`.
