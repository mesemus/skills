# Formik — submission mechanics, `connect`, and `FieldArray` helpers

Covers the specific Formik behaviors behind this ecosystem's multi-action
submit-button pattern and its class-component-heavy codebase. See
[../forms.md §9](../forms.md#9-multiple-submit-intents-from-one-formik-form)
for the pattern itself; this file is why that pattern works the way it does.

## `handleSubmit(e?)` — the event is optional and only conditionally used

```js
handleSubmit(e) {
  if (e && e.preventDefault) e.preventDefault();
  if (e && e.stopPropagation) e.stopPropagation();
  // dev-mode: warn if triggered by a <button> with no explicit `type`
  submitForm().catch((reason) => {
    console.warn("Warning: An unhandled error was caught from submitForm()", reason);
  });
}
```

- It's safe to call `formik.handleSubmit(event)` from a plain button
  `onClick` (not just a `<form onSubmit>`) — this is exactly what every
  action button in the multi-submit pattern does. Calling it with no
  argument at all also works.
- **Dev-mode warning on ambiguous buttons**: if the currently-focused
  element when submission fires is a `<button>` with no explicit `type`
  attribute, Formik warns, because browsers default an untyped `<button>`
  inside a `<form>` to `type="submit"`. Concretely: **every action button
  inside a Formik `<Form>` other than the intended default submit button
  must have `type="button"` explicitly**, or clicking it can trigger *both*
  your handler *and* a native duplicate form submission via the button's
  implicit `type="submit"`.
- `handleSubmit` swallows submission errors down to a **`console.warn`
  only** — it does not surface them in the UI or re-throw anywhere you'd
  catch by default. Any error your `onSubmit` throws or rejects with must be
  handled explicitly inside `onSubmit` itself (e.g. via
  `formikBag.setErrors(...)`, as in
  [../forms.md §8](../forms.md#8-validation-is-server-authoritative--dont-reach-for-yup)) —
  Formik will not display it for you.
- `submitForm()` is the promise-returning equivalent with no event handling
  — use it directly (e.g. from an `async` handler) when there's no DOM event
  to pass through.

## `connect(Component)` — the class-component escape hatch

```js
import { connect } from "formik";
```

A HOC, not a hook: it renders `Component` inside a `FormikConsumer` and
injects the current formik bag as a `formik` prop, throwing if there's no
ancestor `<Formik>`/`<Form>`/`<Field>` in the tree. This is the tool for
reaching the formik bag from a **class component** (which can't call
`useFormikContext()`). Given how much of this codebase is still
class-based (see
[../conventions.md §4](../conventions.md#4-component-organization)), action
buttons are typically **double-`connect`ed**: react-redux's `connect` (for
Redux-derived props like loading/action state) stacked with formik's
`connect` (for the `formik` bag), both applied to the same class component.
If a class component needs both Redux state and the formik bag, this
double-wrapping is the established pattern here, not a workaround to avoid.

## `FieldArray` render-prop helpers

`push, remove, swap, move, insert, replace, unshift` (this version has no
`pop` or other helpers) — all immutable, implemented on top of `setIn` (see
[paths-and-state.md](paths-and-state.md)), so each call produces a new array
rather than mutating in place. The practical implication: after a `move` or
`swap`, every row's **index-based `fieldPath` changes**, even though the row
objects themselves didn't. Anything that cached an index-derived value
outside of Formik's own state (a DOM element id, a ref keyed by the old
index, a locally memoized computation keyed by `fieldPath`) must be
recomputed after a reorder, not reused — `ArrayField`'s own synthetic
`__key` (see
[../react-invenio-forms/array-field.md](../react-invenio-forms/array-field.md))
exists specifically to give React a stable `key` prop that survives this
kind of index churn; don't substitute the array index itself as a React
`key` for repeatable rows that support reordering or removal.
