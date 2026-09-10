# Invenio forms — deep reference (react-invenio-forms + Formik)

Read this file when writing or editing a deposit/edit form, or any Formik-based
form in the Invenio Semantic UI ecosystem. This file covers the *Invenio
usage* layer — form composition, sections, validation model, the
multi-submit pattern. For `react-invenio-forms`' own component-by-component
contract, see `references/react-invenio-forms/`:

- [react-invenio-forms/components.md](react-invenio-forms/components.md) —
  exact props/behavior of every field component (`TextField`, `SelectField`,
  `RemoteSelectField`, `RichInputField`, `GroupField`, `AccordionField`, ...).
- [react-invenio-forms/array-field.md](react-invenio-forms/array-field.md) —
  `ArrayField`/repeatable-groups in full depth (`requiredOptions`,
  `showEmptyValue`, the `__key` convention, when to bypass it).
- [react-invenio-forms/errors-and-labels.md](react-invenio-forms/errors-and-labels.md) —
  `FieldLabel`, `FeedbackLabel`/`ErrorLabel`, and the exact three-way error
  shape every field must handle.
- [react-invenio-forms/custom-fields.md](react-invenio-forms/custom-fields.md) —
  the `CustomFields` widget-resolution mechanism for backend-configured
  fields.
- [react-invenio-forms/overridable-and-utils.md](react-invenio-forms/overridable-and-utils.md) —
  `showHideOverridable(WithDynamicId)`, `parametrizeWithFormContext`, `http`/
  `withCancel`, and the smaller utility exports.
- [react-invenio-forms/gotchas.md](react-invenio-forms/gotchas.md) —
  consolidated, cross-referenced gotchas for the library itself.
- [react-invenio-forms/ui-primitives.md](react-invenio-forms/ui-primitives.md) —
  the library's non-field exports (`Image`, `FilesList`,
  `GridResponsiveSidebarColumn`, `UserListItemCompact`, `InvenioPopup`, ...).

A deposit-style form's **file upload/management step** is a substantial,
separate subsystem, not a `react-invenio-forms` field — see
[file-upload.md](file-upload.md).

For the handful of Formik-itself behaviors this codebase leans on (not
general Formik knowledge — just the specific, version-confirmed mechanics
behind `enableReinitialize`, the multi-submit-button pattern, and
`FieldArray`), see `references/formik/`:

- [formik/paths-and-state.md](formik/paths-and-state.md) — `getIn`/`setIn`
  path syntax and copy-on-write semantics (why `FastField`'s optimization is
  sound, and isn't, in specific cases).
- [formik/reinitialization.md](formik/reinitialization.md) —
  `enableReinitialize`'s deep-equality check and the real risk of it
  discarding in-progress edits.
- [formik/submission-and-arrays.md](formik/submission-and-arrays.md) —
  `handleSubmit` vs `submitForm`, the `type="button"` requirement, `connect`
  for class components, `FieldArray` helpers.
- [formik/gotchas.md](formik/gotchas.md) — consolidated, cross-referenced
  gotchas for Formik itself.

## 1. The stack for a form

`react-invenio-forms` (peer deps: `formik ^2`, `semantic-ui-react ^2`, `yup`,
`react-overridable`) provides Semantic-UI-flavored wrappers around Formik
fields. A form is:

```jsx
import { BaseForm } from "react-invenio-forms";

<BaseForm
  onSubmit={handleSubmit}
  formik={{
    initialValues: record,     // plain JS object, deserialized from the API record
    enableReinitialize: true,  // needed if initialValues can change after mount
  }}
>
  {/* fields go here */}
</BaseForm>
```

`BaseForm` is nothing but `<Formik onSubmit={...} {...formik}><Form>{children}</Form></Formik>`.
There is no magic beyond this — everything else is composition of field
components inside it.

In the real RDM deposit form this `BaseForm` is one layer inside a larger
stack: `DepositFormApp` builds a Redux store and deserializes the raw API
record via a **serializer schema** (a map of `fieldpath → {serialize,
deserialize}` descriptors, see §8) before it ever reaches `initialValues`.
Don't assume the JSON schema of the record is the same shape as the form's
`initialValues` — a serializer layer sits in between whenever vocabulary
lookups, custom fields, or ID↔object conversions are involved.

`enableReinitialize: true` here isn't a cosmetic default — it's load-bearing
because the draft's URL/PID changes after the first save, and it comes with
a real risk of discarding unsaved edits if the Redux record slice updates
for unrelated reasons while the user is mid-edit. See
[formik/reinitialization.md](formik/reinitialization.md) before changing how
or when that Redux slice updates.

## 2. `fieldPath` — the one convention that matters most

Every field component takes a `fieldPath` prop: a Formik dot/bracket path
into the values object, e.g. `"metadata.creators[2].person_or_org.name"`.
There is no other way to address a field. When building a field inside an
`ArrayField` render-prop, construct child paths by string interpolation from
the `indexPath`/`arrayPath` the render-prop gives you — don't invent your own
separator.

## 3. Core field components (all from `react-invenio-forms`)

All leaf fields share one skeleton: wrap Formik's `Field`/`FastField`, read
`fieldPath`, render a `semantic-ui-react` `Form.*` component, and render
`helpText` as a **sibling** `<label className="helptext">` after the field
(not inside the field itself). The field set is `TextField`,
`TextAreaField`, `RichInputField`, `SelectField`, `RemoteSelectField`,
`RadioField`/`BooleanField`/`ToggleField`, `GroupField`, and
`AccordionField` (see §7 for the last one) — exact props and non-obvious
runtime behavior (e.g. `SelectField`'s `onChange` override responsibility,
`RemoteSelectField`'s fixed-at-mount debounce, `RichInputField`'s
lazy-getter value) are in
[react-invenio-forms/components.md](react-invenio-forms/components.md).

## 4. `FieldLabel` and error rendering

`FieldLabel` is a field's `label` prop — deliberately just an icon + text,
never the place for help text (help text is always a separate sibling
`<label className="helptext">`). Field errors are not always plain
strings — every field must resolve `undefined` | string |
`{message, severity, description}` (see §6). Full detail on both, including
`FeedbackLabel`/`ErrorLabel` and the exact error-resolution logic to copy
into a custom field, is in
[react-invenio-forms/errors-and-labels.md](react-invenio-forms/errors-and-labels.md).

## 5. Repeatable groups — `ArrayField` (there is no `ArrayFieldItem`)

`ArrayField` wraps Formik's `FieldArray` and calls its `children` prop as a
**render-prop function** — that's the entire "ArrayFieldItem" pattern:

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

`ArrayField` renders the "Add" button and label/help text for you; you wire
remove/reorder UI yourself from Formik's `arrayHelpers`. Two behaviors worth
knowing before using it: it injects a synthetic `__key` into every row that
**must be stripped before submitting to an API**, and `requiredOptions` is
how "this field must contain at least one row of type X" is enforced (by
auto-injecting a disabled, pre-filled row) instead of via a validation
library. Full behavior — `requiredOptions`, `showEmptyValue`, the `__key`
convention, and when it's legitimate to bypass `ArrayField` for a
hand-rolled `FieldArray` — is in
[react-invenio-forms/array-field.md](react-invenio-forms/array-field.md).

## 6. Errors are not always strings

Every field must tolerate three error shapes: `undefined`, a plain string, or
`{message, severity, description}` (Invenio's "checks" system, severity ∈
`info | warning | error`). The canonical resolution logic, repeated in every
built-in field, is:

```js
const computedError =
  error ||
  getIn(errors, fieldPath) ||
  (initialValue === value && getIn(initialErrors, fieldPath));
```

The `initialValue === value` guard exists so a stale backend-returned error
(surfaced via `initialErrors` after a failed save) disappears the moment the
user edits that field away from its original value — copy this check in any
custom field, or edited fields will show an unfixable-looking permanent
error. Rendering (`FeedbackLabel`/`ErrorLabel`) and the rationale for the
guard are covered in full in
[react-invenio-forms/errors-and-labels.md](react-invenio-forms/errors-and-labels.md).

`AccordionField` does the analogous recursive walk over a whole section's
`includesPaths` to badge-count `info`/`warning`/`error` counts in the section
header — see §7.

## 7. Sections — `AccordionField` + `includesPaths`

Deposit-style forms are organized as a flat sequence of `AccordionField`
blocks (not separate section components with their own state):

```jsx
<AccordionField
  includesPaths={["metadata.title", "metadata.resource_type"]}
  label="Basic information"
>
  <ResourceTypeField fieldPath="metadata.resource_type" options={...} />
  <TextField fieldPath="metadata.title" label="Title" required />
</AccordionField>
```

`includesPaths` lists every field path that belongs to the section.
`AccordionField` walks `form.errors`/`form.initialErrors` for those paths,
buckets any severity-tagged errors, and shows count badges + turns the
accordion header red if any `error`-severity issue exists inside — this is
the standard "collapsed section shows it has a problem" UX. Define the
`includesPaths` lists in one config object near the top of the form file so
they're easy to keep in sync with the fields actually rendered.

## 8. Validation is server-authoritative — don't reach for Yup

Despite `yup` being a peer dependency of `react-invenio-forms`, **the real
RDM deposit form has no `validationSchema` and no Yup schema at all.**
Client-side "requiredness" is just the HTML5 `required` prop on fields (plus
the `requiredOptions`/pre-filled-row trick in §5 for repeatable fields).
Validation is otherwise delegated to the backend:

1. Save/publish dispatches an API call.
2. On a validation failure, the backend returns a flat list of
   `{field, messages, severity, description}` errors.
3. A serializer converts that list into a nested object matching Formik's
   error shape: `_set(deserializedErrors, e.field, {message, severity, description})`.
4. The submit handler calls `formikBag.setErrors(deserializedErrors)`.

Follow this model for new forms in this ecosystem: define `required` at the
field level for basic UX, and treat the backend's response as the source of
truth for everything else, mapping its error list back onto `fieldPath`s. If
a form has no backend endpoint yet, using a Yup `validationSchema` passed via
`formik={{ validationSchema }}` on `BaseForm` still works technically (it's
plain Formik), but it is not the idiomatic pattern in this codebase.

This deliberately diverges from generic Formik guidance you may know from
elsewhere ("use Yup/Zod schema validation instead of manual validate
functions" is standard advice for a plain Formik app) — that advice assumes
a form with no separate backend validation layer. This ecosystem always has
one, and duplicating validation rules into a client-side Yup schema on top
of it is redundant and prone to drifting out of sync with the backend's
actual rules.

**Record → form value mapping** goes through per-field serializer
descriptors, not the raw JSON schema, e.g. a vocabulary field:
`deserialize`: `{id: "publication", title: {...}}` → `"publication"` (the
plain string a `SelectField` needs); `serialize`: `"publication"` →
`{id: "publication"}`. If you add a field backed by a vocabulary, write a
matching pair of small serialize/deserialize functions rather than binding
the dropdown directly to the raw record shape.

## 9. Multiple submit intents from one Formik form

Formik supports exactly one `onSubmit`. When a form needs several distinct
actions (Save draft / Publish / Preview / Delete), the pattern is a small
React Context that records *which* button was clicked immediately before
delegating to Formik's single submit handler:

```jsx
const FormSubmitContext = React.createContext({ setSubmitContext: undefined });

// Save button
const { setSubmitContext } = useContext(FormSubmitContext);
const handleSave = (event) => {
  setSubmitContext("SAVE");
  formik.handleSubmit(event);
};

// the form's single onSubmit
function onFormSubmit(values, formikBag) {
  switch (submitContext.actionName) {
    case "SAVE": return saveAction(values);
    case "PUBLISH": return publishAction(values);
    // ...
  }
}
```

Reach for this whenever a form needs more than one action button — it's the
established idiom here, not a one-off hack. Every button other than the
implicit default submit button must have `type="button"`, and class-based
buttons typically need Formik's own `connect` HOC alongside react-redux's —
see [formik/submission-and-arrays.md](formik/submission-and-arrays.md) for
why both of those matter here.

## 10. Custom / dynamically-configured fields

For metadata fields whose existence and labels are only known at
deploy-time/backend-config-time (Invenio-RDM's "custom fields"), use
`CustomFields` rather than hand-writing each one:

```jsx
<CustomFields
  config={customFieldsUI}       // backend-supplied field list + widget names
  record={record}
  templateLoaders={[
    (widget) => import(`@templates/custom_fields/${widget}.js`), // site override
    (widget) => import(`your-package/src/deposit/customFields`), // package built-ins
    (widget) => import("react-invenio-forms"),                    // generic fallback
  ]}
  fieldPathPrefix="custom_fields"
/>
```

Each backend-declared field names a `ui_widget` (e.g. `"Imprint"`);
`CustomFields` tries each `templateLoaders` entry **in order** and uses the
first one that resolves a component — this is how a deployment can supply a
bespoke widget for a given name without forking the package that defines the
default. Sub-field labels/placeholders come from the backend config object,
not hardcoded strings, so the same widget can render arbitrary
backend-defined labels. Full resolution-order mechanics:
[react-invenio-forms/custom-fields.md](react-invenio-forms/custom-fields.md).

## 11. Gotchas checklist

Two things specific to *how Invenio builds forms* with this library:

- Don't reach for a Yup `validationSchema` by default — see §8.
- A form needing more than one submit action needs the shared-context
  pattern in §9; Formik itself only supports one `onSubmit`.

For the library's own gotchas (`FastField` re-render pitfalls, `SelectField`
custom-`onChange` responsibility, `ArrayField`'s `__key` stripping,
`GroupField`'s substring-based error check, `required`+`disabled`
interaction, and the full three-way error shape), see the consolidated,
cross-referenced list in
[react-invenio-forms/gotchas.md](react-invenio-forms/gotchas.md).
