---
name: frontend-conventions
version: 3
description: Use when building, modifying, or reviewing frontend code in Vue or React — component structure, hooks/composables, state management, API client, auth/tokens, permissions, forms, performance, security (XSS, input validation), accessibility, naming, error handling, Tailwind, imports, testing scope, or adding new pages/features
agents: [main_agent, general_purpose]
---

# Frontend Conventions

## Runtime & Tooling

| Tool | Purpose |
|------|---------|
| **Bun** | Runtime + package manager (never npm/yarn/node) |
| **Vite** | Bundler (via framework integration) |
| **Tailwind CSS** | Styling — latest major (v4), CSS-first via `@theme` |
| **oxlint** | Linting |
| **oxfmt** | Formatting (printWidth: 100) |

```bash
bun install              # Install dependencies
bun run dev              # Start dev server
bun run lint             # oxlint
bun run fmt              # oxfmt
```

- **JavaScript by default** — prefer `.js`/`.vue` for rapid prototyping unless the project explicitly chooses TypeScript
- Line length: 100 characters (don't wrap unnecessarily)
- Bun workspaces for monorepo shared packages
- **Dates: Luxon `DateTime`** is the default for all date/time work (parsing, formatting, math, zones) — never native JS `Date`
- **No frontend unit tests by default** — the frontend is verified at the e2e level, handled separately and only when asked. Don't scaffold component/unit tests unprompted.

## Supporting References

- Use `vue.md` for Vue composition, Pinia setup stores, component APIs, lifecycle, composables, and page data-fetching details.
- Use `nuxt.md` for Nuxt routing, middleware, runtime config, Nuxt UI, icons, layouts, and third-party scripts.
- Use `react.md` for React function components, hooks, state/data, performance, security, and the Next.js / Vite (VoidZero) setup.

> The sections from **Design Principles** through **Naming Conventions** below are framework-agnostic and apply to both Vue and React. Everything from **Project Structure** onward is the existing Vue/Nuxt stack detail.

## Design Principles

Apply these to every file, function, component, store, composable, and hook — Vue or React.

- **Single responsibility** — one unit does one thing. If you need "and" to describe it, split it. The highest-leverage rule for code that stays readable and testable.
- **Separation of concerns** — keep UI, business logic, data access, and side-effects in different layers. A component that fetches, transforms, validates, and renders is four jobs wearing one hat. (This is why stores own data and pages own loading/error UI.)
- **Composition over inheritance** — build behavior from small units (composables, hooks, services), not deep class hierarchies.
- **DRY, but not prematurely** — remove duplication once a pattern appears a third time. Abstracting after one occurrence usually produces the wrong abstraction; duplication is cheaper than the wrong coupling.
- **KISS, and scope deliberately** — write the simplest thing that meets the requirement today. Thinking ahead is good; don't bake in config flags, abstraction layers, or speculative future-proofing on assumption. When a future need looks likely, raise it and confirm scope with the user before building for it — ask, don't assume.
- **Pure by default** — same input, same output, no side-effects. Push I/O, DOM, network, and globals to the edges (stores, composables, effects). Pure logic is trivial to test and reuse.
- **Explicit over implicit** — name things for what they are. No magic numbers (use `@pkg/common` constants), no hidden globals, no clever one-liners the next reader can't follow.
- **Fail loud, fail early** — validate inputs at boundaries and throw/return meaningful errors. Never swallow errors with an empty `catch {}` — the one sanctioned exception is the documented silent-fetch pattern that sets an empty state.

## Complexity & When to Split

Size is not a line count. A 300-line file that does one cohesive thing is fine; a 40-line function juggling three concerns is not. Decide by what the code *does*, not how long it is.

Split a unit when:

- It has more than one **responsibility** (the SRP "and" test).
- It mixes **concerns** — UI + data-fetching + business rules + formatting in one place.
- Its **branching** is hard to follow — many `if`/`switch`/ternary paths, or deeply **nested** blocks (roughly 3+ levels of `if`/`for`/`try` is a smell — extract or use guard clauses).
- It crosses a **domain boundary** — logic for two features tangled together belongs in two homes.
- It takes too many **parameters** — 3+ positional args → pass one options object; boolean-trap params like `doThing(true, false)` → `{ retry, silent }`.
- You can't **name** it cleanly, or a block grows a comment explaining *what* it does — that block is a function waiting to be named.
- It's hard to **test** in isolation — that usually means it's doing too much.

Techniques: guard clauses / early returns over nested pyramids; extract named functions; move logic out of components into composables/hooks/stores; keep templates declarative (logic in `computed`/derived values, not inline expressions). Treat line length as a *weak* prompt to look closer, never as a hard gate.

## Runtime Performance

Measure before optimizing — avoid premature micro-optimization. These apply to both flavors:

- **Debounce/throttle** high-frequency handlers (scroll, resize, input, search).
- **Lazy-load and code-split** anything not needed for first paint — routes, heavy components, third-party SDKs (`@nuxt/scripts` in `nuxt.md`; `lazy`/`defineAsyncComponent` in the framework references).
- **Memoize real computation, not pass-throughs.** `computed`/`useMemo` are for derived values that actually transform or combine data (filter, sort, map, format). Wrapping a value you return unchanged adds overhead for nothing — use it directly; reach for memoization only when there's a calculation worth caching.
- **Paginate / stream** large data; never return or render unbounded lists. Virtualize long lists.
- **Right data structure** — `Map`/`Set` for lookups and membership, not `Array.includes` in a loop (O(n²)).
- **Batch DOM / state updates**; keep work out of tight loops and render/template paths.

## Security Baseline

Frontend code is the XSS front line and everything in it ships to the public. The flavor references add specifics (`v-html`, `dangerouslySetInnerHTML`).

- **Never trust input.** Validate and narrow every external value (user input, route params, query strings, API responses, third-party SDK callbacks) at the boundary. The codebase uses manual imperative validation (see Form Patterns) — apply it to *shape*, not just presence.
- **Render text as text.** Interpolation (`{{ }}` / `{value}`) auto-escapes — rely on it. Only inject raw HTML (`v-html` / `dangerouslySetInnerHTML`) with content sanitized by a vetted library (DOMPurify), never raw user data.
- **Validate URL schemes** built from user data before binding to `:href`/`:src`/`href`/`src` — block `javascript:`.
- **No secrets in the bundle.** Anything reaching the client is public. `VITE_*`, `NUXT_PUBLIC_*`, and `NEXT_PUBLIC_*` vars ship to the browser — never put API secrets there; secrets stay server-side.
- **Never `eval`, `new Function`, or `setTimeout(string)`** on anything influenced by users.
- **Don't leak internals** to the UI — show the safe extracted error message (see Error Handling), not stack traces or raw server payloads.

## Accessibility

Baseline a11y for every UI (the repo runs `@nuxt/a11y` in dev — see `nuxt.md`).

- **Semantic HTML first** — real `<button>`, `<a>`, `<nav>`, `<label>`, headings in order. Reach for ARIA only when native semantics fall short; a correct native element beats a `role=` patch.
- **Keyboard support** — every interactive element is focusable and operable by keyboard, with a visible focus state and logical tab order. No click-only handlers on non-interactive nodes.
- **Labels & alt text** — inputs have an associated `<label>`; images have `alt`; icon-only buttons have an accessible name (`aria-label`).
- **Color contrast** — meet WCAG AA against the semantic Tailwind tokens; never convey state by color alone.

## Naming Conventions

- **Variables / functions:** `camelCase`. Functions are verb-first (`fetchItems`, `handleSubmit`, `parseConfig`).
- **Booleans** read as predicates (`isLoading`, `hasError`, `canEdit`, `isAuthenticated`).
- **Constants:** `UPPER_SNAKE_CASE` for true module-level constants (`PASSWORD_MIN`); shared ones live in `@pkg/common`. Never hardcode magic numbers.
- **Components:** `PascalCase` files and identifiers (`.vue` / `.jsx`) — see Component Conventions and the framework references for prefixes and domain folders.
- **Composables / hooks:** `useXxx.js`, file name matches the exported function.
- **Stores:** `<domain>.js`, one domain per file.
- **Types / interfaces** (when a project opts into TypeScript): `PascalCase`, no `I` prefix; prefer `unknown` + narrowing over `any`.
- **No abbreviations** beyond universally understood ones (`id`, `url`, `db`). `usrMgr` is not a name; `userManager` is.
- **One concept per name** — don't reuse `data`, `item`, `temp` everywhere.
- **No dead code** — delete commented-out blocks and unused exports rather than shipping them "just in case."

## Project Structure

```
frontend/
  pages/              # Route-based page components
  components/         # Reusable UI components
    <domain>/         # Feature-specific subfolders
  stores/             # State management (one per domain)
  composables/        # Reusable logic hooks (useXxx)
  layouts/            # Page shells (auth, dashboard, admin)
  middleware/         # Route guards
  config/             # Environment, theme config
  assets/
    css/main.css      # Tailwind @theme + global styles
    images/           # Bundler-processed images (imported, optimized, hashed)
  public/             # Served as-is: favicon, logo, robots.txt — never bundler-processed
```

Principles:
- One store per domain (auth, user, subscription, admin)
- Composables for reusable stateful logic
- Domain subfolders in components/ when a feature has 3+ components
- Config isolated in config/ — never inline env vars

## Tailwind CSS (latest — v4)

Use the latest Tailwind major (currently **v4**). Configuration is CSS-first in `assets/css/main.css` via `@theme` / `@theme inline` (no `tailwind.config.js`):

```css
@theme {
  --breakpoint-sm: 768px;
  --breakpoint-tablet: 768px;
  --breakpoint-lg: 1280px;
  --breakpoint-desktop: 1280px;
  --breakpoint-hd: 1440px;

  --font-sans: "Inter", sans-serif;
  --font-heading: "Lexend", sans-serif;
}
```

### Theme & Colors

Establish the theme once; build everything on it. Colors come from the **Tailwind palette referenced as CSS variables** — never hex or `oklch()` literals, and never the `theme()` function.

- Define semantic **role tokens** (`--color-bg`, `--color-surface`, `--color-text`, `--color-text-muted`, `--color-border`, `--color-primary`, ...) in `@theme inline`, each mapped to a palette variable through an intermediate role var. Override the role vars in `.dark` to flip themes.
- **Status roles are tokens too** — `--color-success`, `--color-warning`, `--color-error`, `--color-info`, each mapped to a palette shade — so `text-error`, `bg-success/10`, etc. stay consistent app-wide instead of ad-hoc palette picks per call site.
- **Derived colors are tokens too** — hovers, tints, accents get their own role (a palette shade or `color-mix(...)`), defined and flipped alongside the rest. Never inline a derived color in a component.
- Pick the actual hues/shades from the Tailwind v4 color docs **at implementation time** — don't hardcode color values in this codebase.

```css
:root {
  /* role -> Tailwind palette variable (choose hues/shades from the v4 color docs) */
  --app-bg: var(--color-<hue>-<shade>);
  --app-text: var(--color-<hue>-<shade>);
  --app-primary: var(--color-<hue>-<shade>);
  --app-primary-hover: var(--color-<hue>-<shade>);
}
.dark {
  --app-bg: var(--color-<hue>-<shade>);
  --app-text: var(--color-<hue>-<shade>);
  --app-primary: var(--color-<hue>-<shade>);
  --app-primary-hover: var(--color-<hue>-<shade>);
}
@theme inline {
  --color-bg: var(--app-bg);
  --color-text: var(--app-text);
  --color-primary: var(--app-primary);
  --color-primary-hover: var(--app-primary-hover);
}
```

### Base & Component Classes

Don't re-skin the same element with utility soup at every call site. Theme it once, then reuse.

- **Baseline the HTML elements.** Style base elements (`body`, headings, `a`, form controls) against the role tokens in `@layer base`, so plain markup already looks right and defaults aren't repainted everywhere.
- **Reusable components own their whole style.** A recurring element (button, card, input, badge) gets one component class in `@layer components` that encapsulates everything it needs — built from the role tokens — instead of reassembling `bg-… font-… border-…` utilities wherever it appears.

```css
@layer components {
  .btn {
    @apply inline-flex items-center rounded-md px-4 py-2 font-medium;
    @apply bg-primary text-surface hover:bg-primary-hover;
  }
}
```

In markup, consume role-token utilities and component classes — `bg-bg`, `text-text`, `border-border`, `.btn` — not raw palette utilities (`bg-blue-600`). Reach for a raw palette utility only for a deliberate one-off that isn't a theme role.

### Breakpoints

Use named breakpoints in templates:
- `tablet:` (768px) — tablet and up
- `desktop:` (1280px) — desktop and up
- `hd:` (1440px) — large screens

### Dark Mode

Class-based via `.dark` on `<html>`, managed by the framework's color mode module. The `.dark` block overrides the role tokens, so themed utilities and component classes flip automatically — no `dark:` prefixes needed for themed colors.

## Component Conventions

### Naming
- **PascalCase** file names: `ContentCard.vue`, `BillingHistoryTable.vue`
- **App\*** prefix for layout chrome: `AppFooter.vue`, `AppNavItem.vue`
- **Domain prefix** in subfolders: `logistics/LogisticsStatusBadge.vue`

### Organization
- Root-level: generic, reusable across features
- `components/<domain>/`: feature-specific (3+ components = own folder)
- `components/ui/`: thin wrappers around third-party SDKs

### Communication
- Props down, events up (no event bus, no global state for UI events)
- Runtime prop declarations with defaults
- Emit-based child→parent communication
- Template refs + expose for imperative parent→child control (rare)

## API Client Design

Singleton HTTP client pattern:

```js
// composables/useApi.js
import ky from "ky";
import { env } from "~/config/env.js";

let _api = null;
let _refreshPromise = null;

export function useApi() {
  if (_api) return _api;

  const instance = ky.create({
    prefixUrl: env.API_BASE,
    credentials: "include",
    hooks: {
      afterResponse: [async (request, options, response) => {
        if (response.status === 401) {
          // Mutex: share one refresh across concurrent 401s
          if (!_refreshPromise) {
            _refreshPromise = refreshToken().finally(() => { _refreshPromise = null; });
          }
          await _refreshPromise;
          return ky(request, options); // retry
        }
      }],
      beforeError: [async (error) => {
        try { error.data = await error.response.json(); } catch {}
        return error;
      }],
    },
  });

  _api = (url, opts = {}) => {
    const { body, query, ...rest } = opts;
    return instance(url, {
      ...rest,
      ...(body && { json: body }),
      ...(query && { searchParams: query }),
    }).json();
  };

  return _api;
}
```

Key rules:
- Stores call `useApi()` at initialization, not pages
- Pages call store methods, never `api()` directly
- Error body available as `err.data` (attached in beforeError hook)
- Keep a passthrough list of auth endpoints (login, refresh, logout) that the 401 hook skips — a failed login must not trigger a refresh attempt
- Failed refresh → clear auth state and redirect to login page
- Call paths are relative to `prefixUrl` — **no leading slash** (`api("api/things")`, not `api("/api/things")`); ky throws on a leading slash when `prefixUrl` is set

## Error Handling Strategy

### Three Patterns

**1. Inline error ref** — auth forms, single-action pages:
```js
const error = ref("");
try {
  await auth.login(form);
} catch (err) {
  error.value = err?.data?.error || "Login failed.";
}
```
```html
<AlertComponent v-if="error" color="error" :description="error" />
```

**2. Toast notifications** — dashboard/admin CRUD actions:
```js
try {
  await store.updateThing(id, data);
  toast.add({ title: "Saved", color: "success" });
} catch (err) {
  toast.add({ title: "Failed", description: err?.data?.error || "Something went wrong.", color: "error" });
}
```

**3. Silent store failures** — background/optional data fetches:
```js
async function fetchItems() {
  try {
    const data = await api("api/items");
    items.value = data.items;
  } catch {
    items.value = []; // fail silently, show empty state
  }
}
```

### Error Message Extraction

```js
err?.data?.error || err?.data?.message || err.message || "Something went wrong."
```

## Form Patterns

### State

```js
// Simple form — reactive (won't be reassigned)
const form = reactive({ email: "", password: "" });

// Complex form — ref to object (allows reset via reassignment)
const form = ref({ name: "", items: [] });
```

### Validation (Manual, Imperative)

No validation library. Pre-submit guards:

```js
async function handleSubmit() {
  if (!form.email.trim()) {
    error.value = "Email is required.";
    return;
  }
  if (form.password.length < PASSWORD_MIN) {
    error.value = `Password must be at least ${PASSWORD_MIN} characters.`;
    return;
  }
  // proceed with API call
}
```

Constraints as module-level constants:
```js
const PASSWORD_MIN = 12;
const PASSWORD_MAX = 500;
const NAME_MAX = 200;
```

## Import Conventions

Always explicit — never rely on auto-imports for code clarity:

```js
// Framework APIs
import { ref, computed, onMounted, watch } from "vue";

// Stores
import { useAuthStore } from "~/stores/auth";

// Composables
import { useApi } from "~/composables/useApi.js";

// Components
import PageHeader from "~/components/PageHeader.vue";

// Shared constants
import { SUBSCRIPTION_STATUS, ROLES } from "@pkg/common";

// Dates
import { DateTime } from "luxon";
```

- `~/` alias for project root
- Workspace package for shared constants: `@pkg/common`
- Framework builtins (navigateTo, useCookie) may be auto-imported but explicit is preferred

## Shared Constants

Same workspace package used by backend:

```js
import { SUBSCRIPTION_STATUS, ROLES, PLAN_TIER } from "@pkg/common";
```

- Frozen objects (never TS enums)
- Status conventions: negative=terminal, zero=pending, positive=active
- `status > 0` = alive, `status < 0` = dead
- Never hardcode magic numbers — always reference the constant
- Role and permission flags come from this same package. The backend guard is the security boundary; the frontend uses the identical constants only to hide UI and shape messages. When access rules change, update the backend guard and the frontend visibility together — never just one side.

## Environment Config

```js
// config/env.js
const obj = {};
if (import.meta.dev) {
  obj.API_BASE = "http://localhost:43000";
} else {
  obj.API_BASE = "https://api.example.com";
}
export const env = Object.freeze(obj);
```

- Frozen — no runtime mutation
- Dev/prod branching via `import.meta.dev`
- Only imported by the API client composable
- Third-party keys (OAuth, analytics) via framework's runtimeConfig mechanism, not in this file

## State Management Principles

- **Composition-style stores** (function-based setup, not class/options)
- One store per domain: auth, user, subscription, content, admin
- Singleton API client initialized once per store
- Structure: `state (refs) → derived (computed) → actions (async) → return all`
- Data fetching triggered on mount (SPA mode, no SSR)
- Cross-store calls allowed (e.g., auth store resets subscription store on logout)
- Manual `$reset()` function for cleanup

## Auth & Tokens

Auth is per-project. Confirm the project's scheme before wiring anything — never carry one project's auth mechanism into another.

Default scheme for a new project:

- Short-lived JWT access token + long-lived refresh token used only to mint new access tokens.
- The access token travels as an httpOnly cookie (preferred for browser apps) or a bearer header.
- **No tokens in localStorage** — last resort only, and flag it to the user before doing it.
- 401 → one shared refresh (mutex in the API client) → retry the request; failed refresh → clear auth state and redirect to login.

## Route Guards

Global middleware pattern:

1. **Passthrough** — public routes (`/`, `/auth/*`) skip auth checks
2. **Hydrate** — read auth cookie → if expired, try token refresh
3. **Auth redirect** — if on login page while authenticated → redirect to dashboard
4. **Protected** — if not authenticated → redirect to login
5. **Role guard** — admin routes require staff roles

## Quick Reference

| Need | Pattern |
|------|---------|
| New page | Add file in `pages/`, set layout via page meta |
| New component | PascalCase in `components/` or `components/<domain>/` |
| New store | `stores/<domain>.js` — setup store with ref/computed/actions |
| New composable | `composables/useXxx.js` — returns object with state + methods |
| API call | Store method → `api(url, { method, body, query })` |
| Show error | Inline ref for forms, toast for actions |
| Add constant | Shared package `@pkg/common/constants/` |
| Date/time | Luxon `DateTime` — never native JS `Date` |
| Add breakpoint | `@theme` in `main.css` |
| Add color | Map a palette var to a role token in `@theme inline`; flip in `.dark` |
| Reusable element | Component class in `@layer components`, not repeated utilities |
| Dark mode | Role tokens overridden in `.dark` — auto-applied |

## Common Mistakes

- Calling `useApi()` in pages instead of using store methods
- Hardcoding status numbers instead of importing constants
- Using `VITE_*` env vars directly (use frozen `env` object)
- Putting business logic in components (belongs in stores/composables)
- Relying on auto-imports without explicit import statements
- Wrapping simple values in unnecessary computed (use the ref directly)
- Fetching data in stores without try/catch (unhandled rejections)
- Using event bus or provide/inject for child→parent events (use emits)
- Using native JS `Date` instead of Luxon `DateTime`
- Hardcoding hex/`oklch()` or raw palette utilities (`bg-blue-600`) in components instead of role tokens
- Re-skinning the same element with utility soup instead of a reusable component class
- Changing role access on only one side — backend guard and frontend visibility move together
- Storing auth tokens in localStorage instead of an httpOnly cookie or in-memory state
- Writing frontend unit tests unprompted — frontend verification is e2e-level and on request
