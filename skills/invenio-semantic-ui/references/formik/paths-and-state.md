# Formik — path addressing and immutability (`getIn`/`setIn`)

Formik itself is a well-known, thoroughly-documented library — this
reference (and its siblings in this directory) skips its general API and
covers only the behaviors, confirmed against the installed version
(**2.4.9**), that this codebase's conventions directly depend on.

## `fieldPath` is Formik's own path syntax, not a react-invenio-forms invention

Every `fieldPath` string used throughout this ecosystem
(`"metadata.creators[2].person_or_org.name"`) is parsed by Formik's own
`toPath` (dot segments, `[n]` array indices) inside `getIn`/`setIn`, `Field`,
`FastField`, and `useField`. There is no other path syntax understood
anywhere in the stack — when building a child path inside an `ArrayField`
render-prop, you must produce exactly this format (see
[../react-invenio-forms/array-field.md](../react-invenio-forms/array-field.md)).

```js
import { getIn, setIn } from "formik";
```

Both are exported directly and used outside of Formik's own internals —
e.g. react-invenio-forms's field error-resolution logic calls
`getIn(errors, fieldPath)` directly (see
[../forms.md §6](../forms.md#6-errors-are-not-always-strings)).

## `getIn(obj, path, def)`

Walks the path segment by segment; returns `def` the moment any intermediate
segment is missing or falsy, or if the final value is `undefined`. Because
it stops at the first falsy intermediate (not just missing), `getIn(values,
"metadata.creators[0].name")` returns `def` just as readily when
`metadata.creators` is `undefined` as when it's `[]` with no index `0` —
don't rely on the distinction between "path doesn't exist yet" and "an
ancestor happens to be falsy" when reading nested form state this way.

## `setIn(obj, path, value)` — copy-on-write, not a deep clone

```js
setIn(obj, path, value)
```

returns a **new** object with only the objects/arrays *along the path*
shallow-copied — sibling branches keep their original references untouched.
Two consequences worth knowing:

- **If the value at `path` is already strictly equal (`===`) to `value`,
  `setIn` returns the *original* `obj` reference unchanged** — no new object
  is created at all. This is exactly what makes `FastField`'s
  shallow-compare optimization sound in the cases it *is* sound: if nothing
  actually changed, Formik's own state update is a no-op at the reference
  level, so a component bailing out on an unchanged prop reference isn't
  missing a real update. It's also exactly why `FastField` becomes a
  liability for anything whose relevant state lives *outside* its own
  field's value/error/touched (a sibling field, a parent array mutation) —
  `setIn` changing some *other* path's reference doesn't touch this field's
  own reference at all, so `FastField` has no way to know it should
  re-render. See
  [../react-invenio-forms/gotchas.md](../react-invenio-forms/gotchas.md).
- **Setting a value to `undefined` deletes the key** rather than storing
  `key: undefined`. `setFieldValue(fieldPath, undefined)` removes the
  property from the values object entirely. This matters if any code
  downstream distinguishes "key absent" from "key present with an undefined
  value" (e.g. a required-field check using the `in` operator, or a
  `JSON.stringify`-based diff/dirty check) — clearing a field this way
  changes the object's shape, not just its value.
