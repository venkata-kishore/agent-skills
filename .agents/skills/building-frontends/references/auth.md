# Auth and route guards

Auth is per-project. Confirm the project's scheme before wiring anything; never carry one project's mechanism into another.

Default for a new project:
- Short-lived access token plus a long-lived refresh token that only mints access tokens.
- The access token travels as an httpOnly cookie (preferred for browsers) or a bearer header.
- Never localStorage for tokens. It is a last resort; flag it to the user first.
- On 401, one shared refresh (mutex in the API client), then retry. A failed refresh clears auth state, resets every per-user store and redirects to login. Client code: `references/api-client.md`.

Per-user stores:
- Every per-user store resets when the signed-in user changes (sign-out, user switch, token loss). The auth store triggers the resets.
- An action that writes after an `await` checks the signed-in user did not change while it waited.

Route guard order:
1. Public routes pass.
2. Hydrate auth (cookie, else refresh).
3. Signed-in users on login or signup redirect to the dashboard.
4. Signed-out users redirect to login.
5. Role check for staff or admin areas.

Cookie sessions need `SameSite` plus a CSRF token or an `Origin` check on state-changing requests (see `securing-applications`).

Guards guard routes only; they do not refetch domain data on every navigation. Hiding UI by role is cosmetic: the server is the boundary, and server and UI access rules change together. Nuxt middleware example: `references/nuxt.md`.
