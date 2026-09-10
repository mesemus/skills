# react-invenio-forms — labels and the error shape

Read this before writing any custom field component, or before rendering an
error/severity anywhere in a form — the error shape here is not what plain
Formik usage would lead you to expect.

## `FieldLabel` is intentionally minimal

```jsx
<FieldLabel htmlFor={fieldPath} icon={labelIcon} label={label} />
```

Just an optional icon plus text, nothing else — no tooltip, no help text
slot. Every built-in field uses this as its `label` prop. **Help text is
never part of `FieldLabel`** — it's always a separate sibling
`<label className="helptext">{helpText}</label>` rendered after the field.
If you want a hoverable info popup (e.g. to explain what a severity check
means), compose `InvenioPopup` (`react-invenio-forms` →
`elements/accessibility/InvenioPopup`) next to the label yourself; there is
no built-in affordance for it on `FieldLabel`.

## Errors are not always strings

Every field must handle three possible error shapes:

- `undefined` — no error.
- a plain string — a classic Formik validation error.
- `{message, severity, description}` — Invenio's "checks" system, where
  `severity` is `"info" | "warning" | "error"` and `description` may contain
  pre-sanitized HTML with more detail. Not every error is a hard failure —
  `info`/`warning` are used for soft recommendations a user can ignore.

The resolution logic repeated verbatim across every built-in field is:

```js
const computedError =
  error ||
  getIn(errors, fieldPath) ||
  (initialValue === value && getIn(initialErrors, fieldPath));
```

The `initialValue === value` guard is the non-obvious part: `initialErrors`
holds errors the backend returned on a previous failed save. Once the user
edits a field away from its original value, that stale error is
deliberately suppressed — even though Formik's own live validation hasn't
necessarily re-run yet — specifically so the user doesn't see a permanent,
seemingly unfixable error message after they've already changed the value.
**Copy this exact three-way check into any custom field**; omitting it
either crashes on the object error shape or leaves stale errors stuck on
screen after an edit.

**This is not the touched-gated behavior generic Formik guidance
describes.** Plain Formik's `<ErrorMessage>` convention only shows an error
after the field has been `touched` (blurred at least once), which is
commonly cited as the default good-UX behavior. `computedError` above has no
such gate — `error`/`errors` are rendered as soon as they exist, regardless
of `touched`; `touched` only appears in the narrower `initialError`
suppression case above. If you're used to Formik's touched-gated
`<ErrorMessage>` pattern from elsewhere, don't assume it applies to fields
built on this library — errors here can appear before the user has
interacted with the field at all (e.g. immediately after a failed save
populates `initialErrors`, before the field is touched).

## Rendering errors

- **`FeedbackLabel`** (preferred for new fields) — severity-aware: picks an
  icon (`times circle` for `error`, `info circle` otherwise) and, if
  `description` is present, shows it in an `InvenioPopup` via
  `dangerouslySetInnerHTML`. That HTML is expected to already be sanitized
  by the backend — don't route arbitrary untrusted content through it.
  For group-like fields it also recurses into nested sub-paths to find the
  first leaf error to display (there's no attempt to show multiple
  simultaneous errors on one field — showing just the first is a deliberate
  UX simplification).
- **`ErrorLabel`** — older, simpler; string-or-severity only, no recursive
  sub-path lookup. Prefer `FeedbackLabel` unless matching existing code.
- **`ErrorMessage`** — a generic `Message`-based error box, not bound to a
  specific field/fieldPath.

## Section-level error badges

`AccordionField` performs the analogous recursive walk over a whole
section's `includesPaths`, buckets any severity-tagged errors found into
`info`/`warning`/`error` counts, and shows them as badges in the section
header — see [components.md](components.md#accordionfield) for the exact
mechanics.
