# `oarepo_requests` — custom request-type UI

Read this before building a custom request type's UI. **This file corrects
a natural assumption**: there is no registry mapping a request type to a
custom accept/decline-modal form. What actually exists is narrower and
worth understanding precisely before you go looking for something that
isn't there.

## What does NOT exist

Base `invenio_requests`' `RequestAction.js` hardcodes the modal body for
**every** action (accept/decline/cancel/submit/...) to a single generic
"add an optional comment" `RichEditor`. Its only override points are keyed
by **action name**, not by request type:

- `InvenioRequests.RequestAction.layout` — wraps the whole action component.
- `` `RequestActionModal.title.${action}` `` — title text only.
- `InvenioRequests.RequestActionModal.layout` — wraps the whole modal.

`oarepo_requests` does **not** extend this with a per-request-type form/body
registry. If you're looking for "how do I make the accept modal show a
custom form for my request type," that mechanism doesn't exist in this
package — read on for what to do instead.

## What DOES exist: a Python-side Label/Icon registry

Base `invenio_requests` already parametrizes its type-display components
with per-type overridable ids:

```js
// invenio_requests' RequestTypeLabel.js
<Overridable id={`RequestTypeLabel.layout.${type}`}>
  <Label>{type}</Label>
</Overridable>
```

`oarepo_requests/ui/overrides.py` populates that extension point for
OARepo's own request types via **`oarepo_ui`'s Python-side override
registry** (`UIComponentOverride`, endpoint-scoped — not the React
`overrideStore`):

```python
REQUEST_TYPE_LABELS: dict[str, str] = {
    "publish_draft": "LabelTypePublishDraft",
    "new_version": "LabelTypeNewVersion",
    ...
}

def register_request_ui_overrides() -> None:
    for endpoint in REQUESTS_UI_ENDPOINTS:
        for type_id, component in label_components.items():
            current_ui_overrides.add(
                UIComponentOverride(endpoint, f"RequestTypeLabel.layout.{type_id}", component)
            )
```

The components themselves (`components/labels.js`, `components/icons.js`)
are plain functions named `LabelType<X>`/`IconType<X>`. **This is the real,
existing "type string → component" registry** — but it's scoped to
labels/icons/badges shown in request lists and timelines, not to the
action-modal body. If you're adding a new request type, register its
label/icon this way.

## Custom request-creation UX: bypass the action system entirely

For genuinely custom request UX (not just accept/decline of an existing
request, but *creating* one with a bespoke form), the established pattern
is a **fully standalone React widget** that talks to the REST API directly,
never composed inside `RequestActionController`/`RequestAction`/
`RequestActionModal` at all:

```js
// get_access/index.js — GetAccessButton/GetAccessModal, mounted independently
const buildRequestPayload = (reason, groupId) => ({
  request_type: "group_membership",
  topic: { group: groupId },
  payload: { justification: reason },
});

const createResp = await http.post("/api/requests", buildRequestPayload(values.reason, groupId));
const submitLink = createResp.data?.links?.actions?.submit;
if (submitLink) await http.post(submitLink, {});
```

This is a plain Formik form (e.g. a `TextAreaField` for "justification")
mounted independently via `ReactDOM.render` into its own page div (e.g.
`#standalone_submitter_application`), with its own webpack entry — not an
extension of the generic request-action machinery. **Use this pattern**
when a request type needs custom creation UX: build a standalone
form/modal, POST to `/api/requests` with your `request_type`/`topic`/
`payload`, then call the response's `links.actions.submit` link if one is
present, rather than trying to plug into `RequestActionController`.

`GetAccessButton` itself is genuinely **exported and reusable** — a real
repository imports it as `import { GetAccessButton } from
"@js/oarepo_requests/get_access"` and mounts it with deployment-specific
props (`<GetAccessButton groupId="submitters" groupName={...} />`) on a
plain onboarding page unrelated to any record model. See
[wiring.md §7](wiring.md#7-wiring-a-plain-non-model-page-a-smaller-worked-example)
for that full, real wiring example end-to-end (Python
`TemplatePageUIResource` → template → webpack entry → this component).

`tabs.js` in this package is unrelated to any of the above — plain jQuery
persisting a `tab=` query parameter on a Semantic UI tab menu; no React or
registry involved.
