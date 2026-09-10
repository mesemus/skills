# react-invenio-forms — consolidated gotchas

Quick-scan list, each cross-referenced to the file with full detail. These
are the specific mistakes an agent familiar with plain Formik/React but new
to this library is likely to make.

- **A custom `onChange` on `SelectField` fully replaces default value-setting
  behavior** — you must call `formikProps.form.setFieldValue` yourself, or
  the dropdown's displayed value updates while the Formik value silently
  doesn't. → [components.md](components.md#selectfield)

- **`FastField`/`optimized={true}` shallow-compares only its own
  value/error/touched.** Safe for an independent scalar field; a footgun
  inside `ArrayField` rows or for any field whose visibility/validity
  depends on a sibling field. Default to `optimized={false}` unless you've
  verified independence. → [components.md](components.md),
  [../formik/paths-and-state.md](../formik/paths-and-state.md) for why this
  optimization is (and isn't) safe at the `setIn` level.

- **`RemoteSelectField`'s `debounceTime` is fixed at construction time** —
  a later prop change has no effect. → [components.md](components.md#remoteselectfield)

- **`RichInputField`'s `inputValue` is a lazy getter, not a controlled
  value** — treat the field as write-back-on-blur, not two-way-bound.
  → [components.md](components.md#richinputfield)

- **`ArrayField` injects a synthetic `__key` into every row**, living inside
  the same object that becomes your Formik value. Strip it before sending
  values to an API — the library does not do this for you.
  → [array-field.md](array-field.md)

- **`GroupField` and `ArrayField`'s error-highlighting are `startsWith`
  substring checks against error keys, not exact path matches.** Field
  names that share a prefix (`metadata.title` vs `metadata.titles`) can
  produce a false-positive "this group has an error" flag.
  → [components.md](components.md#groupfield), [array-field.md](array-field.md)

- **`required` + `disabled` together silently defeats HTML5 required
  validation** (browsers skip validating disabled inputs). Fields wrapped
  via `showHideOverridable` emit a `console.warn` about this; plain,
  unwrapped fields do not. → [overridable-and-utils.md](overridable-and-utils.md)

- **Every field must tolerate three error shapes**: `undefined`, a plain
  string, or `{message, severity, description}`. Missing the
  `initialValue === value` guard when resolving `initialErrors` leaves a
  stale, seemingly unfixable error on screen after the user edits the
  field. Copy the exact resolution logic from an existing field.
  → [errors-and-labels.md](errors-and-labels.md)

- **`CustomFields` resolves widgets by trying `templateLoaders` in order and
  taking the first match** — a widget name silently resolves to whichever
  loader finds it first, which is *by design* the override mechanism for
  custom fields, but easy to misread as "the wrong component is rendering"
  if you don't know the resolution order. → [custom-fields.md](custom-fields.md)

- **There is no `ArrayFieldItem` and no standalone `CheckboxField` export** —
  don't search for them; the patterns are `ArrayField` + a render-prop, and
  `BooleanField`/`ToggleField`/`RadioField` respectively.
  → [components.md](components.md), [array-field.md](array-field.md)

- **Despite `yup` being a peer dependency, real deposit forms in this
  ecosystem use no `validationSchema` at all** — validation is
  server-authoritative. See
  [../forms.md §8](../forms.md#8-validation-is-server-authoritative--dont-reach-for-yup)
  before adding a Yup schema as a default approach.
