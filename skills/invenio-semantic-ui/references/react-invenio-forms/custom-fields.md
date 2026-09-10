# react-invenio-forms — `CustomFields` (dynamically-configured fields)

Read this when a form needs fields whose existence, labels, and widget type
are only known at deploy/backend-config time (Invenio's "custom fields"
concept) rather than hardcoded in the frontend.

## Why this exists

A deployment can declare extra metadata fields (e.g. "Imprint", "Journal",
"Meeting") in backend configuration without the frontend package needing to
know about them ahead of time. `CustomFields` is the generic runtime loader
that makes this possible — it is a fundamentally different authoring model
than hand-composing `TextField`/`ArrayField` yourself, and you should reach
for it only when fields are genuinely backend-driven, not as a general
alternative to writing fields directly.

```jsx
<CustomFields
  config={customFieldsUI}          // backend-supplied: [{ field, ui_widget, props... }, ...]
  record={record}
  templateLoaders={[
    (widget) => import(`@templates/custom_fields/${widget}.js`), // 1: site/instance override
    (widget) => import("your-package/src/deposit/customFields"),  // 2: package's own built-ins
    (widget) => import("react-invenio-forms"),                    // 3: generic fallback
  ]}
  fieldPathPrefix="custom_fields"
/>
```

## Widget resolution order — the part that isn't obvious

Each backend-declared field names a `ui_widget` string (e.g. `"Imprint"`).
`CustomFields` (via the internal `importWidget`/`loadWidgetsFromConfig`
helpers) tries each entry in `templateLoaders` **in the order given**, and
uses the **first one that successfully resolves a component**:

```js
for (const loader of templateLoaders) {
  try {
    const module = await loader(widgetName);
    component = module.default ?? module[widgetName];
    if (component) break;
  } catch {
    continue; // this loader doesn't have it — try the next
  }
}
```

This is the mechanism that lets a specific deployment override one widget
(by placing a matching file under its own `templateLoaders[0]` path) without
forking the package that defines the default implementation — the same
`ui_widget` name resolves to a different component purely based on which
loader finds it first. When adding a new built-in custom-field widget to a
package, put it where that package's own loader entry expects it; when
overriding one for a specific site, add a same-named module earlier in the
`templateLoaders` list rather than editing the shared package.

## Value shape and labels

Sub-field labels, placeholders, and help text for a custom field come from
the **backend-supplied config object** passed as that field's props — not
hardcoded strings in the widget component. This is why the same widget
component can render arbitrary backend-defined labels/help text across
different deployments: the widget is a template, the config supplies the
copy.

Vocabulary-backed custom fields follow the same `{id} ↔ plain string`
serialize/deserialize convention as regular vocabulary fields (see
[../forms.md §8](../forms.md#8-validation-is-server-authoritative--dont-reach-for-yup)),
and the generic custom-field serializer strips the `__key` property (see
[array-field.md](array-field.md)) before values reach the API, for any
custom field built on `ArrayField`.
