# Forms

- State: in Vue, `reactive` for a simple form that is never reassigned, a `ref` to an object when the form is reset by reassignment. In React, controlled inputs.
- No validation library. Run manual imperative checks before submit, on shape and not just presence. Limits are module-level constants (`PASSWORD_MIN`, `NAME_MAX`), not inline numbers.
- The server validates again; client checks are for feedback only.
- A field error that appears after an action is shown beside its field, associated with it (`aria-describedby`, `aria-invalid`) and announced: a live region, or move focus to the error. Do not put `role="alert"` on everything. Every control has an accessible name (a `label`).
- Server-side field failures come back as per-field `errorCode`s and go through `messageForError` (`references/api-client.md`), never raw server text.

```js
async function handleSubmit() {
  fieldErrors.email = form.email.trim() ? "" : "Email is required.";
  fieldErrors.password = form.password.length >= PASSWORD_MIN ? "" : `Use at least ${PASSWORD_MIN} characters.`;
  if (fieldErrors.email || fieldErrors.password) return;
  try {
    await auth.login(form);
  } catch (err) {
    formError.value = messageForError(err);
  }
}
```
