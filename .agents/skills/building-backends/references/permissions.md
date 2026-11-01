# Bitmask permission model

Default for apps that need roles beyond "logged in". Per-domain bitmasks checked in code: O(1), no database read per request. Change it only when the project already uses another model.

- Split the product into permission domains (content, users, billing, settings, whatever exists). One integer bitmask per domain.
- Cap a domain at 31 flags so the mask stays a signed 32-bit integer and ports across languages (JS bitwise operators, Java `int`, a Postgres `integer`). A domain that outgrows 31 flags splits in two.
- Name flags ENTITY_ACTION (`BILLING_VIEW`, `CONTENT_CREATE`), never ACTION_ENTITY, so sorted output groups by resource.
- Tier flags per capability: `X_VIEW` (read) < `X_CREATE` (mutate) < `X_MANAGE` (elevated). A role gets the lowest tier that suffices.
- A frozen role map takes a role id to `{ DOMAIN: mask }`. It is never mutated at runtime. The super-admin role has every bit in every domain.
- Refuse any change that would leave no holder of the top role, and say why.
- Enforcement is dual: server guards are the security boundary. The frontend imports the same flag and role constants from the shared constants package only to hide UI and shape messages. Change both sides together.

Check shape (pseudocode):

```text
FLAGS.CONTENT_VIEW   = 1 << 0
FLAGS.CONTENT_CREATE = 1 << 1
FLAGS.CONTENT_MANAGE = 1 << 2

canDo(role, domain, flag) = ROLE_PERMISSIONS[role][domain] & flag != 0   (false for an unknown role)
```
