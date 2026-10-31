# Vue 3

## Script setup

`<script setup>` only: no Options API. Order inside it: imports, store and composable setup, props
and emits, reactive state, computed, functions, lifecycle.

```js
import { ref, computed, onMounted } from "vue";
import { useThingStore } from "~/stores/thing.js";

const thingStore = useThingStore();
const props = defineProps({ userId: { type: String, required: true } });
const emit = defineEmits(["updated"]);
const loading = ref(false);
const hasItems = computed(() => thingStore.items.length > 0);
onMounted(loadData);
```

Assign a store to a named variable (`const thingStore = ...`) before calling it. Pages do not
import the API client.

## Reactivity

- `ref()` for primitives, reassignable objects and template refs; `reactive()` for form state that
  is never reassigned; `computed()` for derived values; `watch` for specific sources.
- Destructuring `reactive()` or a store breaks reactivity. Use `toRefs(state)`, or
  `storeToRefs(store)` for state and getters. Actions destructure directly.

## Pinia setup stores

Always the setup form: state refs, computed, plain `async function` actions, return everything flat.
Only the store calls the API client.

```js
export const useThingStore = defineStore("thing", () => {
  const api = useApi();
  const items = ref([]);
  const isEmpty = computed(() => items.value.length === 0);

  async function fetchItems(params) {
    const auth = useAuthStore();
    const userId = auth.user?.id;
    const data = await api("api/things", { query: params });
    if (auth.user?.id !== userId) return;
    items.value = data.things;
  }

  function $reset() {
    items.value = [];
  }

  return { items, isEmpty, fetchItems, $reset };
});
```

- Every per-user store has `$reset()`, called when the signed-in user changes.
- An action that writes after an `await` checks the signed-in user is unchanged first (above).
- A store refreshes on defined events (see `SKILL.md`), not on every route change.
- Store actions throw on failure. The caller turns the error into UI with `messageForError`.

## Composables

`useXxx.js` exports `useXxx()` and returns an object (never a bare ref). A module-level `ref` makes
a singleton shared across components; a function-scoped one is per instance.

## Component API

- Runtime prop declarations with types and defaults; `defineEmits` with kebab-case event names.
- Props down, events up. No event bus, no `provide`/`inject` for events (injection only), no
  `$parent`, never mutate props.
- `defineExpose` plus a template ref only for rare imperative control (`focus`, `clear`).

## Lifecycle

`onMounted` for fetching, DOM access and listeners. `onUnmounted` for cleanup (intervals,
listeners, subscriptions). Options API names do not carry over: `beforeMount` is `onBeforeMount`,
and the `<script setup>` body replaces `created`.

## Page data loading

The store owns data; the page owns loading and error UI. Always `onMounted` (SPA mode).

```js
const loading = ref(true);
const errorMessage = ref("");

async function loadData() {
  loading.value = true;
  errorMessage.value = "";
  try {
    await thingStore.fetchItems({ page: 1 });
  } catch (err) {
    errorMessage.value = messageForError(err);
  } finally {
    loading.value = false;
  }
}
```

Render three states: loading, error (with a retry calling `loadData`) and empty.

## Performance

- Stable `:key` on every `v-for`: a unique id, not the index, for dynamic lists.
- Never `v-if` and `v-for` on one element: filter in a `computed`, or wrap in a `<template>`.
- `v-show` for frequent toggles, `v-if` for rare ones.
- `shallowRef` / `shallowReactive` for large structures; `v-once` / `v-memo` for static subtrees.
- `defineAsyncComponent` (with `<Suspense>` where a fallback is needed) for heavy components.
- `computed` over template methods (computed caches).

## Security

- `v-html` only with DOMPurify-sanitized or fully trusted content; default to `{{ }}`.
- Validate dynamic `:href` / `:src` built from user data (block `javascript:`).
- Validate API response shape; do not assume it.
- `VITE_*` and `NUXT_PUBLIC_*` are public. Real secrets stay server-side.
