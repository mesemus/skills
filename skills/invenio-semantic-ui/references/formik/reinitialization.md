# Formik — `enableReinitialize` and the initial-error/-touched lifecycle

Read this before relying on `enableReinitialize`, and before assuming a
Redux- or API-sourced `initialValues` prop is safe to update at will while a
form is mounted — the real deposit form architecture in this ecosystem
(`initialValues` sourced from Redux, `enableReinitialize: true`, see
[../forms.md §1](../forms.md#1-the-stack-for-a-form)) depends on exactly the
details below.

## The reinitialize check is deep-equal, not reference-equal

`enableReinitialize` defaults to `false`. When `true`, on every render
Formik compares the *previous* `initialValues` prop against the *current*
one with a deep-equality check (`lodash.isEqual`), not a reference (`===`)
check:

```js
if (isMounted && !isEqual(initialValues.current, props.initialValues)) {
  if (enableReinitialize) {
    initialValues.current = props.initialValues;
    resetForm();
  }
}
```

Two consequences:

- A brand-new object with identical deep content (e.g. a fresh object
  literal built fresh on every parent render) does **not** trigger a reset —
  you don't need to memoize `initialValues` purely to avoid spurious resets.
- **Any actual content change to `initialValues` — even one unrelated to
  what the user is doing — does trigger a full `resetForm()`.** In an
  architecture where `initialValues` is read from a shared store (Redux, as
  in this codebase's `DepositBootstrap`), if that store slice updates for
  *any* reason while the user has unsaved edits in a field that hasn't yet
  round-tripped back into the store, the reset silently **discards those
  in-progress edits** — `resetForm()` replaces `values` wholesale with the
  new `initialValues`, it does not merge. Only update the store slice that
  feeds `initialValues` as a deliberate consequence of a save/publish
  action completing, not from unrelated background dispatches, or gate
  `enableReinitialize` off if that's not achievable.

`initialErrors`, `initialTouched`, and `initialStatus` are each reinitialized
independently, with their own `!isEqual(...)` guard against the *previous*
value of that specific prop — updating `initialValues` alone does not by
itself refresh `initialErrors`/`initialTouched` unless those props also
changed in the same render.

## `resetForm()` falls back to cached, not empty, error/touched state

Calling `resetForm()` with no arguments resets `errors`/`touched`/`status`
to whatever was cached from the **previous** reset (`initialErrors.current`/
`initialTouched.current`), not to empty objects, and not necessarily to the
component's *current* `initialErrors`/`initialTouched` props if those
weren't part of what triggered this particular reset. Concretely: if you
want a reinitialize (e.g. after a successful save) to also clear stale,
already-resolved errors, you must pass fresh `initialErrors`/`initialTouched`
props alongside the new `initialValues` in the same render — don't assume
`enableReinitialize` clears errors just because it resets values.

## Why this matters for backend-driven errors

`initialErrors` is the mechanism the deposit form's error-deserialization
step plugs into: a failed save's `{field, messages, severity}` list gets
converted into a Formik-shaped error object and set via
`formikBag.setErrors(...)` (see
[../forms.md §8](../forms.md#8-validation-is-server-authoritative--dont-reach-for-yup)) —
which is a live `errors` update, not `initialErrors`. `initialErrors` itself
is for errors known *before* the user has interacted with the form at all
(e.g. re-rendering a form that failed validation on the previous page
load). Every built-in field's error-resolution logic explicitly reads both
`errors` and `initialErrors` and picks whichever applies (see
[../react-invenio-forms/errors-and-labels.md](../react-invenio-forms/errors-and-labels.md))
— conflating the two, or assuming only one of them is ever populated, will
miss real error states.
