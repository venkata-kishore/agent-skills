# Performance, security and accessibility

Open when building a feature that handles user input, renders user data, or adds interactive UI, and when reviewing for these.

## Performance

Measure before optimizing.

- Debounce high-frequency handlers (input, scroll, resize).
- Lazy-load and code-split what first paint does not need.
- Memoize real computation, not values returned unchanged.
- Paginate large data and virtualize long lists.
- Use `Map` and `Set` for lookups.

## Security

- Validate the shape of every external value: user input, route params, query strings, API responses, SDK callbacks.
- Render text as text. Interpolation (`{{ }}` / `{value}`) escapes; rely on it. Raw HTML (`v-html`, `dangerouslySetInnerHTML`) only after a vetted sanitizer such as DOMPurify.
- Validate URL schemes built from user data before binding to `href` or `src` (block `javascript:`).
- Everything in the bundle is public. Secrets stay server-side. `VITE_*`, `NUXT_PUBLIC_*` and `NEXT_PUBLIC_*` ship to the browser.
- Never `eval`, `new Function` or `setTimeout(string)` on user input.
- Hiding UI by permission is cosmetic; the server is the boundary. When access rules change, change the server guard and the UI together.
- Wider rules (sessions, CSRF, headers, uploads): `securing-applications`.

## Accessibility (WCAG 2.2 AA)

- Semantic HTML first; ARIA only where native semantics fall short.
- Everything interactive works by keyboard, with a visible focus state and a logical tab order.
- Every control has an accessible name: a `label`, or `aria-label` for icon-only buttons. Images have `alt`.
- A field error that appears after an action is associated with its field and announced. No blanket `role="alert"` (details in `forms.md`).
- Never convey state by colour alone.
- Modals trap focus and return it to the trigger on close. Respect `prefers-reduced-motion`.
