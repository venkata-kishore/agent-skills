# Nuxt

## Configuration

```js
// nuxt.config.js
export default defineNuxtConfig({
  ssr: false,
  devServer: { port: 43001 },
  modules: ["@pinia/nuxt", "@nuxt/ui", "@nuxt/scripts", "@nuxt/a11y", "@nuxt/hints"],
  colorMode: { classSuffix: "" },
  icon: {
    serverBundle: "local",
    clientBundle: { scan: true },
    collections: ["lucide", "icon-park-outline"],
  },
  runtimeConfig: { public: { googleClientId: "" } },
});
```

- Pure SPA (`ssr: false`): no `useFetch` / `useAsyncData` hydration; fetch in `onMounted` through store actions.
- `classSuffix: ""` gives the bare `.dark` class. Icon collections come from `@iconify-json/*`.
- `@nuxt/a11y` and `@nuxt/hints` are dev-only auditing.

## Routing and layouts

File-based routing under `pages/`. `[id].vue` gives `useRoute().params.id`. A nested route's parent file contains only `<NuxtPage />`, children live in the matching folder.

Layouts live in `layouts/` (`default`, `auth`, `dashboard`, `admin`). Pick one with `definePageMeta({ layout: "auth" })`; children inherit the parent shell's layout.

## Global middleware

`middleware/auth.global.js` runs on every navigation. Return `navigateTo(path)` to redirect, return nothing to allow. Order: public routes pass, hydrate from the auth cookie (else try refresh), redirect signed-in users away from login and signup, redirect signed-out users to login, then the role check for `/admin`.

```js
export default defineNuxtRouteMiddleware((to) => {
  const auth = useAuthStore();
  if (to.path === "/" || to.path.startsWith("/auth/")) return;
  if (!auth.isAuthenticated) {
    auth.hydrateUser();
    if (!auth.isAuthenticated) return auth.refreshToken().catch(() => navigateTo("/login"));
  }
  if (["/login", "/signup"].includes(to.path)) {
    return auth.isAuthenticated ? navigateTo("/dashboard") : undefined;
  }
  if (!auth.isAuthenticated) return navigateTo("/login");
  if (to.path.startsWith("/admin") && !auth.isStaff) return navigateTo("/dashboard");
});
```

The middleware only guards routes. It does not refetch domain data on every navigation. The auth store resets every per-user store when the signed-in user changes. The role check is a convenience; the server enforces access.

## Auto-imports

Nuxt auto-imports framework utilities (`navigateTo`, `definePageMeta`, `defineNuxtRouteMiddleware`, `useCookie`, `useRoute`, `useRouter`, `useRuntimeConfig`, `useHead`, `useNuxtApp`). Always import your own components, stores and composables explicitly. For a framework utility the linter needs explicit, use `import { x } from "#imports"`.

## Third-party scripts

Load external SDKs lazily with `@nuxt/scripts`. `trigger: "manual"` loads on the first `await onLoaded()`; use it for payment, OAuth and analytics SDKs.

```js
import { useScript } from "@nuxt/scripts";
const { onLoaded } = useScript("https://example.com/sdk.js", { trigger: "manual" });
async function openSdk() {
  await onLoaded();
  /* use window.Sdk */
}
```

## Runtime config

Per-environment third-party keys (OAuth client IDs, analytics keys, flags) go in `runtimeConfig.public`, filled from `NUXT_PUBLIC_<KEY>` env vars and read with `useRuntimeConfig().public.key`. They are public. Not for the API base URL (that is `config/env.js`) and never for secrets.

## Nuxt UI

- Components are prefixed `U` (`UButton`, `UInput`, `UAlert`, `UIcon`, `UModal`, `USelect`).
- Icons: `<UIcon name="i-lucide-plus" class="size-4" />`. Order of sets: `i-lucide-*`, then `i-icon-park-outline-*`, then `i-tabler-*` for brand logos only.
- Toast: `useToast().add({ title, color })`. On failure, `description` is `messageForError(err)`, never the raw error text.

```js
try {
  await thingStore.updateThing(id, data);
  toast.add({ title: "Saved", color: "success" });
} catch (err) {
  toast.add({ title: "Failed", description: messageForError(err), color: "error" });
}
```

- Icon-only buttons need an accessible name (`aria-label`). A `UAlert` or toast that reports a field error must still be tied to the field (see `SKILL.md`).
- Theme in `app.config.js` (`ui.colors`, slot overrides). Theme every form control's slots once (`button`, `input`, `checkbox`, `formField`, `select`, `selectMenu`, `textarea`, `inputNumber`) so sizing is uniform; mixed sizes in one toolbar mean a missing slot override. Never re-size at call sites. Colours still come from tokens (`references/tailwind.md`).
- Colour mode: `<UColorModeButton />`, or `useColorMode().preference = "dark" | "light" | "system"`.

## Page transitions

Define `.page-enter-active`, `.page-leave-active` (opacity transition) and `.page-enter-from`, `.page-leave-to` (opacity 0) in `main.css`; Nuxt applies them to `<NuxtPage>`.
