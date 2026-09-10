# Formik — consolidated gotchas

Quick-scan list of the Formik-level (not react-invenio-forms-level; see
[../react-invenio-forms/gotchas.md](../react-invenio-forms/gotchas.md) for
those) behaviors most likely to cause a subtle bug in this codebase, each
cross-referenced to the file with full detail.

- **`enableReinitialize` compares `initialValues` by deep equality, and
  resets `values` wholesale (not a merge) whenever it differs.** If
  `initialValues` is sourced from a shared store that can update for
  reasons unrelated to the user's current edits, `enableReinitialize` can
  silently discard in-progress, unsaved changes. →
  [reinitialization.md](reinitialization.md)

- **`resetForm()` restores `errors`/`touched` from cached values, not from
  empty objects** — a reinitialize doesn't automatically clear stale errors
  unless you also pass fresh `initialErrors`/`initialTouched` in the same
  render. → [reinitialization.md](reinitialization.md)

- **Every non-primary action button inside a Formik `<Form>` needs an
  explicit `type="button"`.** Without it, browsers default to
  `type="submit"`, and Formik's own dev-mode check will warn — but the real
  bug is a duplicate/unintended form submission firing alongside your
  button's own handler. → [submission-and-arrays.md](submission-and-arrays.md)

- **`handleSubmit`/`submitForm` only `console.warn` on an unhandled
  rejection from `onSubmit` — they never surface it in the UI.** Catch and
  handle submission errors explicitly inside `onSubmit` itself. →
  [submission-and-arrays.md](submission-and-arrays.md)

- **`setIn(obj, path, undefined)` deletes the key rather than storing
  `undefined`.** Code that distinguishes "key missing" from "key present but
  undefined" (an `in` check, a naive `JSON.stringify` diff) will see a
  shape change, not just a value change, when a field is cleared via
  `setFieldValue`. → [paths-and-state.md](paths-and-state.md)

- **`FieldArray`'s `move`/`swap`/`insert` change every affected row's
  index-based `fieldPath`.** Anything caching an index-derived id/ref
  outside Formik's own state must be recomputed after a reorder, not reused
  — use a stable per-row key (like `ArrayField`'s synthetic `__key`), never
  the raw array index, for React's `key` prop on reorderable rows. →
  [submission-and-arrays.md](submission-and-arrays.md)

- **`connect()` from `formik` is a HOC for class components, distinct from
  the `useFormikContext()` hook** — and it throws if rendered outside a
  `<Formik>`/`<Form>` tree. Class-based action buttons in this codebase
  typically stack it with react-redux's `connect` on the same component;
  that's the established pattern, not something to refactor away by
  default. → [submission-and-arrays.md](submission-and-arrays.md)

- **`isInitialValid` is deprecated** (Formik's own dev-mode warns on it) —
  use `initialErrors` and/or `validateOnMount` instead if you need to know a
  form's validity before the user interacts with it.
