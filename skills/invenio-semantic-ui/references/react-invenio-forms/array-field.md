# react-invenio-forms — `ArrayField` in depth

Read this before building any repeatable group of fields (creators,
identifiers, dates, related works, ...). There is **no `ArrayFieldItem`
component** — this entire pattern is one component (`ArrayField`, exported
also as the overridable-wrapped `Array`) plus a render-prop.

## The render-prop is the whole pattern

`ArrayField` wraps Formik's `FieldArray` and calls its `children` prop as a
function once per existing row:

```jsx
<ArrayField
  fieldPath="metadata.creators"
  label="Creators"
  defaultNewValue={{ person_or_org: { name: "" } }}
  addButtonLabel="Add creator"
>
  {({ arrayHelpers, indexPath, arrayPath }) => (
    <GroupField key={indexPath} basic>
      <TextField
        fieldPath={`${arrayPath}.${indexPath}.person_or_org.name`}
        label="Name"
      />
      <Button icon="close" onClick={() => arrayHelpers.remove(indexPath)} />
    </GroupField>
  )}
</ArrayField>
```

What `ArrayField` does for you: renders the label/help text, iterates
existing values, and renders an "Add" button that pushes `defaultNewValue`.
What it does **not** do: remove/reorder UI — you wire that yourself from
Formik's standard `arrayHelpers` (`push`, `remove`, `swap`, `move`, `insert`,
`replace`, `unshift`) that the render-prop hands you. `move`/`swap` change
every affected row's index-based `fieldPath` — see
[../formik/submission-and-arrays.md](../formik/submission-and-arrays.md) for
why the synthetic `__key` below (not the array index) must be used as the
React `key`.

Always build child `fieldPath`s by interpolating the `arrayPath`/`indexPath`
values the render-prop provides — don't invent a different path convention;
Formik's `getIn`/`setFieldValue` require exact string matches.

## Non-obvious behaviors

- **Every row gets a synthetic, ever-decreasing `__key`** (`nextKey`, starts
  at `-1` and decrements), used purely as the React `key` prop for newly
  pushed rows so they never collide with 0-indexed pre-existing rows. This
  key lives *inside the same object* that becomes your Formik array value —
  **you must strip `__key` before sending values to an API**, or it leaks
  into the payload. `react-invenio-forms` does not strip it for you; a
  consumer's serializer must (e.g. `delete value.__key`).

- **`requiredOptions`** (an array of partial objects) is how "this
  repeatable field must contain at least one row of type X" is enforced —
  not by a validation library. For each entry in `requiredOptions`, if no
  existing row matches it (a `lodash.matches`-style partial comparison),
  `ArrayField` auto-injects a new row seeding `defaultNewValue` merged with
  that required option. The intended pattern is to also mark that injected
  row's inputs `disabled` (and disable its remove button) so the user can't
  accidentally delete a structurally-required row — `ArrayField` itself
  doesn't do the disabling; the consuming field component does, by checking
  whether the current row matches a required option.

- **`showEmptyValue`** — if there are no existing values and no
  `requiredOptions`, seeds exactly one empty row once, tracked via internal
  state (`hasBeenShown`) so removing that lone row doesn't cause it to
  reappear. Useful for fields where showing an initial empty row is better
  UX than showing only an "Add" button.

- **Coarse group-level error flagging**: the outer `Form.Field` gets an
  `error` class if *any* Formik error key starts with the array's
  `fieldPath` — same substring-based approach as `GroupField` (see
  [components.md](components.md#groupfield)), not an exact-match check.

## When to bypass `ArrayField`

For repeatable fields with heavy per-row logic and real performance
sensitivity (RDM's creators/contributors field is the reference example),
it's legitimate to hand-roll Formik's `<FieldArray>` directly with a custom
`shouldComponentUpdate` that deep-compares only the row's own slice of
`form.values`/`errors`, rather than accepting `ArrayField`'s default
re-render behavior. Reach for this only after profiling shows a real
problem — for the vast majority of repeatable fields (identifiers, dates,
related works), the plain `ArrayField` + render-prop pattern above is the
correct default.
