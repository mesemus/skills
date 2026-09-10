# react-invenio-forms — field components reference

Skips what's obvious from Formik/Semantic UI React knowledge (that
`TextField` renders a text input, that `label` sets a label). Covers only
the behavior you can't guess: exact prop contracts and non-obvious runtime
behavior specific to this library. Read [../forms.md](../forms.md) first for
how these compose into an actual form.

## Shared skeleton

Every leaf field wraps Formik's `Field`/`FastField`, reads its value via the
`fieldPath` prop (a Formik dot/bracket path, e.g.
`"metadata.creators[2].person_or_org.name"` — this is the only addressing
scheme; there's no other way to bind a field), renders a `semantic-ui-react`
`Form.*` primitive, and renders `helpText` as a **sibling**
`<label className="helptext">` placed *after* the field, not inside it or
inside the field's own `label` prop.

## `TextField` / `TextAreaField`

Props: `fieldPath`, `label`, `required`, `disabled`, `helpText`, `optimized`.
`optimized` (default `false`) swaps the underlying Formik `Field` for
`FastField` — see [gotchas.md](gotchas.md) before turning it on.

## `RichInputField`

TinyMCE-backed via an internal `RichEditor`. The non-obvious part:
`inputValue={() => value}` is passed as a **getter function, not a reactive
value** — this is deliberate, to stop TinyMCE re-rendering on every
keystroke. Practically: treat this field as write-back-on-blur (it calls
`setFieldValue`/`setFieldTouched` on blur), not as a normal two-way-bound
controlled input. Don't expect the editor's displayed content to update
just because the Formik value changed elsewhere.

## `SelectField`

Semantic UI `Dropdown`-backed. Two behaviors that aren't guessable from the
prop names:

- **It keeps values that aren't in `options` visible** — if a record's
  existing value isn't part of the currently loaded `options` list (e.g. a
  vocabulary term that's been paginated away), the field still shows it
  rather than silently blanking the selection (`ensureSelectedValuesInOptions`
  internally).
- **Passing your own `onChange` fully replaces the default value-setting
  behavior.** The field does not also call `setFieldValue` for you in that
  case — you must call `formikProps.form.setFieldValue(fieldPath, ...)`
  yourself inside your handler. Forgetting this is a silent bug: the
  dropdown's displayed selection updates, but the Formik value never
  changes, and the bug won't surface until submit.

`allowAdditions` enables free-text entries (used for e.g. subject/keyword
fields that accept both vocabulary terms and free text).

## `RemoteSelectField`

`SelectField` plus autocomplete-from-API. Props: `suggestionAPIUrl`,
`suggestionAPIQueryParams`, `searchQueryParamName` (default `"suggest"`),
`serializeSuggestions` (default maps `{title, id}` → `{text, value, key}` —
override this if your API returns a different shape), `debounceTime`
(default 500ms).

**The debounce is captured once, at component construction** (a lodash
`_debounce` built inside a class field initializer) — changing the
`debounceTime` prop after mount has no effect. If a field needs a
runtime-adjustable debounce, this component isn't sufficient as-is.

## `RadioField`, `BooleanField`, `ToggleField`

There is **no standalone `CheckboxField` export**. A single checkbox is
`BooleanField` (toggles via `setFieldValue(fieldPath, !value)`); a toggle
switch is `ToggleField` (= `RadioField` with `toggle` plus
`onValue`/`offValue`/`onLabel`/`offLabel`, and `optimized: true` by default —
one of the few fields optimized by default, presumably because a toggle's
value is never programmatically derived from a sibling).

## `GroupField`

A thin `Form.Group` wrapper for laying out one row of fields (typically one
row of a repeatable item). The non-obvious part: it flags the whole group as
erroneous with a plain **`error.startsWith(fieldPath)` substring check**
against every error key — not an exact path match. Two field paths that
share a prefix (`metadata.title` vs `metadata.titles`) can cause a false
positive here; don't rely on this for exact isolation between
similarly-named fields. `basic` renders a plain `<div>` instead of
`Form.Group`'s flex layout, used when nesting groups inside each other.

## `AccordionField`

Collapsible section wrapper. Props: `includesPaths` (array of field paths
"owned" by this section), `label`, `severityChecks` (custom
label/description overrides per severity level).

Internally it recursively walks `form.errors`/`form.initialErrors` looking
for entries under any of `includesPaths`, and for each one found that has a
`{message, severity}` shape (see [errors-and-labels.md](errors-and-labels.md)),
buckets it into `info`/`warning`/`error` and renders a count `Label` badge in
the accordion header, plus adds an `"error"` CSS class to the header if any
`error`-severity issue exists anywhere under those paths. This is what
produces the "collapsed section shows how many problems are inside it" UX
used throughout deposit forms — it is not automatic for a section unless you
pass a complete and accurate `includesPaths` list.

## Deprecated prop aliases

Several widgets (`MultiInput`, `BooleanCheckbox`, the `Array`/`ArrayField`
wrapper) accept both a current and a deprecated prop name, resolved with
nullish coalescing: `helpText ?? description`, `labelIcon ?? icon`. Use the
current names (`helpText`, `labelIcon`) in new code — the deprecated aliases
exist only for backward compatibility with older call sites.
