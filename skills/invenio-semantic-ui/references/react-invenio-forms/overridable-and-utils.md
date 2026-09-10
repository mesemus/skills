# react-invenio-forms — overridability layer and utility exports

Read this when you need one of the library's smaller exported helpers, or
when you're confused why a field you're using is already override-aware
without any `<Overridable>` JSX visible at your call site.

## This package's layer on top of `react-overridable`

The generic override mechanism (`OverridableContext`, `Overridable.component`,
`parametrize`, `overrideStore`) lives in the separate `react-overridable`
package — see
[../conventions.md §1](../conventions.md#1-overridable-components--the-central-customization-mechanism)
for that mechanism in full. `react-invenio-forms` adds two conveniences on
top of it, and nearly every field export in
[components.md](components.md) is wrapped with one of them:

- **`showHideOverridable(id, Component)`** — for a component whose override
  ID is fixed and known at export time (i.e. almost every built-in field).
  It does two things beyond plain `Overridable.component(id, Component)`:
  adds a free `hidden` prop (render `null` when `true`), and emits a
  `console.warn` if both `required` and `disabled` are passed together
  (since HTML5 constraint validation is skipped on disabled inputs — a real
  footgun elsewhere in the library that this wrapper alone warns about;
  plain unwrapped fields don't).

  ```js
  export const ResourceTypeField = showHideOverridable(
    "InvenioRdmRecords.DepositForm.ResourceTypeField",
    ResourceTypeFieldComponent
  );
  ```

  Because the wrapping happens at export time, **the field is already
  override-aware wherever it's imported** — you don't need to wrap your
  *usage* of it in `<Overridable>` JSX for its internals to be overridable
  (a consuming app's own `RDMDepositForm`-style component may additionally
  wrap the *usage site* in its own separate `<Overridable id="...">.container`
  for a second, independent override point — that's a choice the consumer
  makes, not something this wrapper does).

- **`showHideOverridableWithDynamicId(Component)`** — for components whose
  override ID isn't known until runtime (custom fields, where the ID comes
  from backend config rather than being hardcoded). The `id` becomes a prop
  supplied by the caller instead of baked in at export time.

- **`parametrizeWithFormContext(Component, propsFactory)`** — like
  `react-overridable`'s `parametrize`, but `propsFactory({existingProps, formValues})`
  also receives the live Formik `values` (via `useFormikContext()`
  internally). Use this instead of plain `parametrize` when an override's
  extra props need to depend on the current form state (e.g. hiding or
  relabeling a field based on another field's current value) — plain
  `parametrize` only supports static props or props derived from the
  wrapped component's own existing props, not from form state.

## HTTP client

```js
import { http, withCancel } from "react-invenio-forms";
```

`http` is a preconfigured `axios` instance (CSRF cookie/header,
`application/vnd.inveniordm.v1+json` Accept header) — use it instead of a
bare `axios` instance for any API call from a form component.
`withCancel(promise)` wraps a promise so it can be cancelled on unmount,
avoiding `setState` on an unmounted component. Both are the same instances
used across the wider ecosystem, not form-specific — see
[../conventions.md §5](../conventions.md#5-http-client) for the standard
usage pattern (`try/catch` + `componentWillUnmount` cancellation).

## Dropdown/option utilities

- **`dropdownOptionsGenerator`**, **`createOption`**, **`mergeOptions`** —
  build/merge Semantic UI `Dropdown` option lists (`{text, value, key}`)
  from vocabulary-shaped API data (`{id, title}` or similar).
- **`ensureSelectedValuesInOptions`** — the function `SelectField` uses
  internally to keep an existing value visible even when it's absent from
  the current `options` array; exported separately in case a custom
  dropdown-based field needs the same behavior.

## Formatters

- **`humanReadableBytes`** — file-size formatting (e.g. for a files list).
- **`toRelativeTime(isoString, locale)`** — "3 days ago"-style relative
  timestamps, locale-aware (pass `i18next.language`, matching the usage
  convention in [../conventions.md §2](../conventions.md#2-i18n)).
