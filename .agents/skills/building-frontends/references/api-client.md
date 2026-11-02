# API client and errors

Only stores (or the data layer behind them) use the client. Pages and components call store actions. One client instance per app.

## Client

Check the installed `ky` version first; the option names differ. In ky v2 the base option is `prefix` (slashes are normalized) and hooks receive one state object. In ky v1 it is `prefixUrl`, hooks take positional arguments, and a leading slash on the path throws.

```js
// composables/useApi.js (ky v2 shape)
import ky from "ky";
import { env } from "~/config/env.js";

const AUTH_PATHS = ["api/auth/login", "api/auth/refresh", "api/auth/logout"];
let api = null;
let refreshPromise = null;

export function useApi() {
  if (api) return api;
  const client = ky.create({
    prefix: env.API_BASE,
    credentials: "include",
    hooks: {
      afterResponse: [
        async ({ request, response, retryCount }) => {
          const skip = AUTH_PATHS.some((path) => request.url.includes(path));
          if (response.status !== 401 || skip || retryCount > 0) return;
          refreshPromise ??= refreshToken().finally(() => (refreshPromise = null));
          await refreshPromise;
          return ky.retry({ request: new Request(request), code: "TOKEN_REFRESHED" });
        },
      ],
    },
  });
  api = (url, { body, query, ...rest } = {}) =>
    client(url, {
      ...rest,
      ...(body && { json: body }),
      ...(query && { searchParams: query }),
    }).json();
  return api;
}
```

- One shared refresh across concurrent 401s. Skip the auth endpoints: a failed login must not trigger a refresh.
- Failed refresh: clear auth state, reset every per-user store, redirect to login.
- On a failed request the thrown error carries the parsed body as `err.data`. Read only its `errorCode` (and structured fields such as per-field codes); see below.

## Error codes and the message helper

The backend returns a stable `errorCode` with every error. The UI words the message.

```js
// utils/errorMessage.js (pure, unit-testable)
const MESSAGES = {
  INVALID_CREDENTIALS: "Email or password is incorrect.",
  NAME_TAKEN: "That name is already in use.",
};
const FALLBACK = "Something went wrong. Try again.";

export function messageForError(err) {
  return MESSAGES[err?.data?.errorCode] ?? FALLBACK;
}
```

- Never match `err.message` or any server text, and never show the raw body, `err.message` or a stack trace.
- A refusal the UI wants to word differently gets its own `errorCode` from the backend, not a text match on the client.
- Where the message goes:
  - Form or single action: an inline error beside the field or form.
  - Dashboard action: a toast (`Saved` on success, `messageForError(err)` on failure).
  - Failed load: an error state in the view with a retry. Do not turn it into an empty list.
