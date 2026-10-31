# Vue 3 Patterns

## Script Setup

`<script setup>` is the only pattern — no Options API, no `setup()` function return.

Standard ordering within `<script setup>`:
```js
// 1. Imports
import { ref, computed, onMounted, watch } from "vue";
import { useAuthStore } from "~/stores/auth";
import { useApi } from "~/composables/useApi.js";
import { SUBSCRIPTION_STATUS } from "@pkg/common";

// 2. Store/composable initialization
const auth = useAuthStore();
const api = useApi();
const route = useRoute();
const toast = useToast();

// 3. Props and emits
const props = defineProps({ userId: { type: String, required: true } });
const emit = defineEmits(["updated", "deleted"]);

// 4. Reactive state
const loading = ref(false);
const items = ref([]);
const form = reactive({ name: "", email: "" });

// 5. Computed
const hasItems = computed(() => items.value.length > 0);

// 6. Functions
async function loadData() { ... }
function handleSubmit() { ... }

// 7. Lifecycle
onMounted(loadData);
```

## Reactivity

| Use | When |
|-----|------|
| `ref()` | Primitives, objects that may be reassigned, template refs |
| `reactive()` | Form state objects that won't be reassigned |
| `computed()` | Derived values (automatically cached) |
| `watch(source, cb)` | React to specific value changes |
| `watchEffect(cb)` | Auto-track all reactive deps used inside |

```js
// ref — can reassign .value
const user = ref(null);
user.value = await fetchUser();

// reactive — mutate properties directly
const form = reactive({ name: "", email: "" });
form.name = "new value";

// computed — derived, cached
const fullName = computed(() => `${user.value.first} ${user.value.last}`);
```

**Destructuring `reactive()` (or a store) breaks reactivity** — it flattens
values into disconnected plain locals:

```js
const state = reactive({ count: 0 });
let { count } = state;            // ❌ count is a detached copy, won't update
const { count } = toRefs(state);  // ✅ stays reactive as a ref

// Pinia: storeToRefs for state/getters, destructure actions directly
const store = useThingStore();
const { items, isEmpty } = storeToRefs(store); // ✅ reactive
const { fetchItems } = store;                  // ✅ actions are plain functions
```

## Pinia Setup Stores

Always use the composition/setup form:

```js
import { defineStore } from "pinia";
import { ref, computed } from "vue";
import { useApi } from "~/composables/useApi.js";

export const useThingStore = defineStore("thing", () => {
  const api = useApi();

  // State
  const items = ref([]);
  const total = ref(0);
  const current = ref(null);

  // Derived
  const isEmpty = computed(() => items.value.length === 0);

  // Actions
  async function fetchItems(params) {
    const data = await api("api/things", { query: params });
    items.value = data.things;
    total.value = data.total;
  }

  async function createItem(body) {
    const data = await api("api/things", { method: "POST", body });
    items.value.unshift(data.thing);
    total.value++;
    return data.thing;
  }

  function $reset() {
    items.value = [];
    total.value = 0;
    current.value = null;
  }

  // Return everything
  return { items, total, current, isEmpty, fetchItems, createItem, $reset };
});
```

Conventions:
- `useApi()` called once at the top
- All state as `ref()`
- Actions are plain `async function` (not arrow functions)
- Explicit `$reset()` for logout cleanup
- Return all state, computed, and actions in flat object
- Store variable pattern: `const thingStore = useThingStore()` — assign to named variable before calling methods

## Composables

Named `useXxx` — encapsulate reusable stateful logic:

### Module-Level Singleton (shared state)

```js
// composables/useSidebarState.js
import { ref } from "vue";

const isExpanded = ref(JSON.parse(localStorage.getItem("sidebar") ?? "true"));

export function useSidebarState() {
  function toggle() {
    isExpanded.value = !isExpanded.value;
    localStorage.setItem("sidebar", JSON.stringify(isExpanded.value));
  }
  return { isExpanded, toggle };
}
```

### Per-Instance (component-scoped)

```js
// composables/useFileValidation.js
import { ASSET_TYPE } from "@pkg/common";

export function useFileValidation({ maxSize = 100 * 1024 * 1024 } = {}) {
  const allowedMimes = Object.values(ASSET_TYPE).flat();

  function validate(file) {
    if (file.size > maxSize) return { valid: false, error: "File too large" };
    if (!allowedMimes.includes(file.type)) return { valid: false, error: "Unsupported type" };
    return { valid: true, error: null };
  }

  return { validate, allowedMimes, maxSize };
}
```

Rules:
- Module-level `ref()` for singletons (state shared across components)
- Function-scoped for per-instance use
- Always return an object (not a bare ref)
- Name matches file: `useXxx.js` exports `useXxx()`

## Component API

### Props

```js
const props = defineProps({
  title: { type: String, required: true },
  items: { type: Array, default: () => [] },
  loading: { type: Boolean, default: false },
  status: { type: Number },
});
```

### Emits

```js
const emit = defineEmits(["file-selected", "item-deleted", "form-submitted"]);

// Usage
emit("file-selected", file);
emit("item-deleted", itemId);
```

Event names: kebab-case (matches template `@event-name` convention).

### Expose (for imperative parent access)

```js
// Child
const inputRef = ref(null);
function focus() { inputRef.value?.focus(); }
function clear() { /* ... */ }
defineExpose({ focus, clear });

// Parent
const childRef = ref(null);
childRef.value?.focus();
```

Use sparingly — prefer props/emits for normal data flow.

### Template Refs

```js
const containerRef = ref(null);
onMounted(() => {
  containerRef.value?.scrollTo(0, 0);
});
```

## Lifecycle

| Hook | Use For |
|------|---------|
| `onMounted` | Data fetching, DOM access, event listeners |
| `onUnmounted` | Cleanup (intervals, listeners, subscriptions) |
| `onBeforeUnmount` | Cleanup before DOM removal |

```js
onMounted(async () => {
  await store.fetchData();
  loading.value = false;
});

onUnmounted(() => {
  clearInterval(pollInterval);
});
```

Options API names don't carry over: use `onBeforeMount` instead of `beforeMount`; `created` has no direct equivalent — the `<script setup>` body runs at that point.

## Event Handling

### Template Events

```html
<button @click="handleClick">Save</button>
<input @input="onInput" @keydown.enter="onSubmit" />
<ChildComponent @item-selected="onItemSelected" />
```

### Child → Parent

```js
// Child component
const emit = defineEmits(["credential", "dismiss"]);
emit("credential", tokenValue);

// Parent template
<GoogleSignInButton @credential="handleGoogleLogin" />
```

### Patterns to Avoid

- Event bus (use store or props/emits)
- `provide`/`inject` for event communication (use for dependency injection only)
- `$parent`/`$children` access
- Mutating props (emit an event instead)

## Page Data Fetching Pattern

```js
const store = useThingStore();
const loading = ref(true);

onMounted(loadData);

async function loadData() {
  loading.value = true;
  try {
    await store.fetchItems({ page: 1 });
  } catch (err) {
    toast.add({ title: "Failed to load", color: "error" });
  } finally {
    loading.value = false;
  }
}
```

- Always `onMounted` (SPA mode, no SSR async data)
- Loading state managed in the page, not the store
- Store owns the data, page owns the loading/error UI state
- `finally` block ensures loading resets even on error

## Performance

Universal runtime-performance rules live in `SKILL.md`; these are the
Vue-specific levers.

- **Stable `:key` on every `v-for`** — a unique id, never the array index for
  dynamic/reorderable lists (index keys cause state bugs on reorder/insert).
- **Never put `v-if` and `v-for` on the same element** — precedence and
  performance trap. Filter in a `computed`, or wrap the loop in a `<template>`
  with the `v-if` inside.
- **`v-show` vs `v-if`** — `v-show` for frequent toggles (keeps the node, flips
  CSS); `v-if` for branches that mount rarely.
- **`shallowRef` / `shallowReactive`** for large structures that don't need deep
  reactivity (big lists, third-party instances) — avoids deep-proxy cost.
- **`v-once` / `v-memo`** for static or rarely-changing subtrees.
- **Lazy-load** routes and heavy components with `defineAsyncComponent` (plus
  `<Suspense>` where a fallback is needed); code-split by route.
- **`computed` over methods in templates** — `computed` caches; a method re-runs
  every render. Keep logic out of inline template expressions (they re-evaluate
  on every change-detection pass).

## Security

Universal security rules live in `SKILL.md`; these are the Vue-specific items.

- **`v-html` is the XSS hole.** Bind only DOMPurify-sanitized or fully trusted
  content to it — never raw user input. Default to `{{ }}`/text binding, which
  auto-escapes.
- **Validate dynamic `:href` / `:src`** built from user data — block
  `javascript:` and other unexpected schemes.
- **Validate API response shape** — don't assume the server payload's structure;
  the manual validation in Form Patterns applies to incoming data too.
- **`VITE_*` / `NUXT_PUBLIC_*` env vars are public** — they ship to the browser.
  No secrets there; use the `runtimeConfig` pattern in `nuxt.md` for
  per-environment public keys and keep real secrets server-side.
