# Nuxt Patterns

## Configuration

```js
// nuxt.config.js
export default defineNuxtConfig({
  ssr: false,  // Pure SPA — no server-side rendering
  devServer: { port: 43001 },

  modules: [
    "@pinia/nuxt",       // State management
    "@nuxt/ui",          // Component library
    "@nuxt/scripts",     // Third-party script loading
    "@nuxt/a11y",        // Accessibility auditing (dev only)
    "@nuxt/hints",       // Dev-only performance/security/a11y hints
  ],

  colorMode: { classSuffix: "" },  // .dark class (not .dark-mode)

  icon: {
    serverBundle: "local",
    clientBundle: { scan: true },
    collections: ["lucide", "icon-park-outline"],
  },

  runtimeConfig: {
    public: {
      googleClientId: "",  // populated via NUXT_PUBLIC_GOOGLE_CLIENT_ID
    },
  },
});
```

Key points:
- SPA mode means no `useFetch`/`useAsyncData` for SSR hydration — use `onMounted` instead
- Color mode uses bare `.dark` class (no suffix)
- Icon collections bundled locally via `@iconify-json/*` packages

## File-Based Routing

```
pages/
  index.vue                    → /
  login.vue                    → /login
  dashboard.vue                → shell (renders <NuxtPage>)
  dashboard/
    index.vue                  → /dashboard
    profile.vue                → /dashboard/profile
    content/
      index.vue                → /dashboard/content
      [id].vue                 → /dashboard/content/:id
  admin.vue                    → shell (renders <NuxtPage>)
  admin/
    index.vue                  → /admin
    users.vue                  → /admin/users
    billing.vue                → shell (renders <NuxtPage>)
    billing/
      index.vue                → /admin/billing
      plans.vue                → /admin/billing/plans
```

### Conventions
- **Dynamic params**: `[id].vue` → accessed via `useRoute().params.id`
- **Nested routes**: parent `.vue` file contains just `<NuxtPage />`, children go in matching folder
- **Layout selection**: `definePageMeta({ layout: "auth" })` in `<script setup>`

## Layouts

```
layouts/
  default.vue      # Public pages — minimal shell
  auth.vue         # Centered card for login/signup flows
  dashboard.vue    # Sidebar + header + main content area
  admin.vue        # Admin variant (different accent color)
```

Usage in pages:
```js
definePageMeta({ layout: "auth" });
```

Pages without explicit layout meta get the framework default. Dashboard and admin pages inherit their layout from the parent shell page.

## Global Middleware

```js
// middleware/auth.global.js
export default defineNuxtRouteMiddleware((to, from) => {
  const auth = useAuthStore();

  // 1. Public routes — skip auth
  if (to.path === "/" || to.path.startsWith("/auth/")) return;

  // 2. Hydrate if needed
  if (!auth.isAuthenticated) {
    auth.hydrateUser();  // read cookie
    if (!auth.isAuthenticated) {
      return auth.refreshToken().catch(() => navigateTo("/login"));
    }
  }

  // 3. Auth pages — redirect if already logged in
  if (["/login", "/signup"].includes(to.path)) {
    if (auth.isAuthenticated) return navigateTo("/dashboard");
    return;
  }

  // 4. Protected — redirect to login
  if (!auth.isAuthenticated) return navigateTo("/login");

  // 5. Admin guard
  if (to.path.startsWith("/admin") && !auth.isStaff) {
    return navigateTo("/dashboard");
  }
});
```

Key patterns:
- `.global.js` suffix → runs on every navigation
- Return `navigateTo(path)` to redirect
- Return nothing to allow navigation
- Store hydration happens in middleware (read cookie first, then try refresh)

## Auto-Imports vs Explicit

Nuxt auto-imports these — usable without import:
- `navigateTo`, `definePageMeta`, `defineNuxtRouteMiddleware`
- `useCookie`, `useRoute`, `useRouter`, `useRuntimeConfig`
- `useHead`, `useNuxtApp`

When explicit import is needed (for clarity or linting):
```js
import { useToast } from "#imports";
import { navigateTo } from "#imports";
```

**Project convention**: Always explicitly import components and stores. Auto-imports are acceptable for framework utilities only.

## Third-Party Script Loading

Use `@nuxt/scripts` for lazy-loading external SDKs:

```js
import { useScript } from "@nuxt/scripts";

const { onLoaded } = useScript("https://checkout.razorpay.com/v1/checkout.js", {
  trigger: "manual",
});

async function openCheckout(options) {
  await onLoaded();  // loads script on first call, cached after
  return new Promise((resolve) => {
    const rzp = new window.Razorpay({
      ...options,
      handler: (response) => resolve(response),
      modal: { ondismiss: () => resolve(null) },
    });
    rzp.open();
  });
}
```

- `trigger: "manual"` — script loads only when `onLoaded()` is awaited
- Use for payment SDKs, OAuth libraries, analytics that shouldn't block page load

## Runtime Config

For third-party keys that vary by environment:

```js
// nuxt.config.js
runtimeConfig: {
  public: { googleClientId: "" }
}
```

Populated automatically from env vars matching `NUXT_PUBLIC_<KEY>`:
```bash
NUXT_PUBLIC_GOOGLE_CLIENT_ID=abc123
```

Accessed in components:
```js
const config = useRuntimeConfig();
const clientId = config.public.googleClientId;
```

Use for: OAuth client IDs, analytics keys, feature flags. **Not for**: API base URL (use the `config/env.js` pattern instead).

## UI Component Library (Nuxt UI)

### Component Usage

All components prefixed with `U`:
```html
<UButton label="Save" color="primary" @click="save" />
<UInput v-model="form.email" placeholder="Email" />
<UAlert color="error" :description="errorMessage" />
<UIcon name="i-lucide-check" />
<UModal v-model:open="showModal">...</UModal>
<USelect v-model="selected" :items="options" />
```

### Icons

Three icon sets, in priority order:
1. `i-lucide-*` — primary (most icons)
2. `i-icon-park-outline-*` — fallback for missing lucide icons
3. `i-tabler-*` — brand logos only (instagram, whatsapp)

Always use `<UIcon>`:
```html
<UIcon name="i-lucide-plus" class="size-4" />
<UIcon name="i-lucide-trash-2" class="size-4 text-red-500" />
```

### Toast Notifications

```js
import { useToast } from "#imports";
const toast = useToast();

// Success
toast.add({ title: "Profile updated", color: "success" });

// Error
toast.add({ title: "Failed to save", description: err?.data?.error, color: "error" });

// Info
toast.add({ title: "Processing...", color: "info" });
```

### Theme Customization

In `app.config.js`:
```js
export default defineAppConfig({
  ui: {
    colors: { primary: "emerald", neutral: "zinc" },
    button: { slots: { base: "justify-center px-4 py-2.5" } },
    input: { defaultVariant: { size: "md" } },
  },
});
```

Size consistency: theme every form control's slots once (`button`, `input`, `checkbox`, `formField`, `select`, `selectMenu`, `textarea`, `inputNumber`) so control sizing is uniform app-wide — never re-size controls at call sites. Mixed control sizes in one toolbar means a slot override is missing.

### Color Mode

```html
<UColorModeButton />  <!-- Built-in toggle -->
```

Or manually:
```js
const colorMode = useColorMode();
colorMode.preference = "dark"; // "light" | "dark" | "system"
```

## Page Transitions

Defined in `main.css`:
```css
.page-enter-active,
.page-leave-active {
  transition: opacity 250ms ease;
}
.page-enter-from,
.page-leave-to {
  opacity: 0;
}
```

Nuxt applies these automatically to `<NuxtPage>` transitions.

## Quick Reference

| Need | Pattern |
|------|---------|
| New page | `pages/<path>.vue` + `definePageMeta({ layout })` |
| Dynamic route | `pages/[param].vue` → `useRoute().params.param` |
| Nested routes | Parent `.vue` with `<NuxtPage />` + child folder |
| Set layout | `definePageMeta({ layout: "dashboard" })` |
| Redirect | `return navigateTo("/path")` in middleware |
| Read cookie | `useCookie("name")` |
| Env var | `useRuntimeConfig().public.key` (for NUXT_PUBLIC_*) |
| Load SDK | `useScript(url, { trigger: "manual" })` |
| Toast | `useToast().add({ title, color })` |
| Icon | `<UIcon name="i-lucide-xxx" />` |
