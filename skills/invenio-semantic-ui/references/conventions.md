# Cross-cutting conventions in the Invenio Semantic UI ecosystem

Read this file for the conventions that apply to *any* component in this
ecosystem, not just forms or search: the overridable-components mechanism,
i18n, HTTP calls, and notification/confirmation UI.

## 1. Overridable components — the central customization mechanism

Invenio deliberately avoids letting downstream sites fork core packages to
customize the UI. Instead, (almost) any component can be swapped by ID via
`react-overridable`. Learn this before writing anything meant to be reused
across deployments — it's the single most load-bearing convention in this
codebase.

**Two ways a component exposes an override point:**

1. **Whole-component**, at export time:
   ```js
   export default Overridable.component("InvenioAdministration.AdminDetailsView", AdminDetailsView);
   ```
   Anyone importing this component gets the override-aware wrapper for free —
   no special JSX needed at the call site.

2. **Inline slot**, wrapping default markup as `children` inside `render()`:
   ```jsx
   <Overridable id="InvenioAppRdm.Deposit.CreatorsField.container" record={record}>
     <CreatorsField fieldPath="metadata.creators" />
   </Overridable>
   ```
   If `id` isn't in the override map, the wrapped `children` render unchanged
   (with any extra props merged in); if it is, the override renders instead,
   receiving the child's original props merged with whatever extra props were
   passed to `<Overridable>`.

**ID convention**: dotted, namespaced strings,
`<Namespace>.<Component>.<slot>.<subslot>`, e.g.
`InvenioAppRdm.Deposit.AccordionFieldBasicInformation.container`,
`InvenioAdministration.SearchResultItem.actions.container`. IDs can also be
built at runtime for per-item overrides, e.g. `` `RequestActionButton.${action}` ``.
There's no central registry file listing every ID — discover them by reading
the component you want to override, or with the dev-mode helper below.

**How overrides get registered and provided:**

```js
// somewhere that runs before the app mounts
import { overrideStore } from "react-overridable";
overrideStore.add("InvenioAppRdm.Deposit.LicenseField.container", MyLicenseField);
```

```jsx
// the app's entrypoint
import { OverridableContext, overrideStore } from "react-overridable";
ReactDOM.render(
  <OverridableContext.Provider value={overrideStore.getAll()}>
    <MyApp />
  </OverridableContext.Provider>,
  domContainer
);
```

Packages typically ship an intentionally-empty override-registration module
(a "mapping" file) that a downstream deployment shadows with its own version
to inject real overrides — the shipped default has zero overrides, all
default components render as-is.

**Debugging tip**: call `window.reactOverridableEnableDevMode()` in the
browser console to overlay every overridable region on the page with a
clickable, copyable ID badge — the fastest way to find the exact ID to
override without reading source.

`parametrize(Component, extraProps)` is the companion helper for injecting
fixed (or props-derived) extra props into a component when registering it as
an override, e.g. binding a page-specific value (like the current community
object) into an otherwise generic component before registering it.

## 2. i18n

Each Python package ships its own `i18next` singleton
(`assets/translations/<package>/i18next.js`, aliased as
`@translations/<package>/i18next`), configured with `keySeparator: false,
nsSeparator: false` — meaning **the translation key is the literal English
string**, gettext-style, not a dotted lookup key.

```jsx
import { i18next } from "@translations/invenio_requests/i18next";

i18next.t("Copy to clipboard");
i18next.t("{{count}} results found", { count: total });

<Trans defaults="Opened {{relativeTime}} by" values={{ relativeTime }} />
```

Rules that follow from static-analysis string extraction
(`i18next-scanner`, configured per-package):

- Arguments to `i18next.t()` must be **literal string constants** — no
  building the key by concatenation. Interpolating *values* via `{{var}}`
  placeholders is fine and is the standard way to inject dynamic content.
- Use `<Trans>` (not `t()`) when the translatable string contains inline
  markup or mixed React content.
- **Import the i18next instance from the same package you're editing** —
  each package's instance only contains that package's translation catalog;
  copy-pasting a component across packages without updating this import is a
  common, silent mistake (strings render as their raw English key untranslated).
- Always translate `aria-label`s alongside visible text, even for icon-only
  buttons.

## 3. Entrypoint / bootstrap shape (context for understanding component props)

You won't usually write these entrypoints (wiring is covered elsewhere), but
recognizing the shape explains where a component's props come from. Apps
mount with React 16's `ReactDOM.render` (no `createRoot`, no `.tsx`
anywhere in this ecosystem) into a DOM node that a Jinja template renders,
reading initial data either from a `dataset` attribute:

```js
const config = JSON.parse(document.getElementById("...").dataset.someKey);
```

or from a hidden input's JSON value:

```js
const getInputFromDOM = (name) => {
  const el = document.getElementsByName(name);
  return el.length && el[0].hasAttribute("value") ? JSON.parse(el[0].value) : null;
};
record={getInputFromDOM("deposits-record")}
```

A single page can mount several independent React trees into several DOM
nodes (e.g. a landing page mounting up to five separate apps). When a
component's data looks like it "comes from nowhere," look for the sibling
data attribute or hidden input on its mount point.

## 4. Component organization

- **No TypeScript anywhere in this ecosystem** — `PropTypes` +
  `.defaultProps` on both classes and function components is the norm.
- **Class components are still the majority** of existing code (roughly 8:1
  over hooks-based function components in the packages surveyed). Expect and
  be comfortable with `class X extends Component { state = {...}; render() {} }`,
  including `static contextType = SomeContext` for context consumption in
  classes. Writing new code as a function component with hooks is fine and
  arguably preferable, but don't be surprised by or feel compelled to rewrite
  legacy class components you're only editing.
- **Container/presentational split** is a named, recurring pattern:
  a `*Controller`/`*Context`-holding class component owns state and an API
  client and exposes both via a Context provider; a plain, often
  `Overridable`-wrapped, function component renders the UI and receives
  everything as props (e.g. `RequestActionController` /
  `RequestActionButton`). Reach for this split once a component mixes
  "fetch and manage state" with "render markup."

## 5. HTTP client

`react-invenio-forms` exports a preconfigured `axios` instance as `http`,
with Invenio's CSRF/content-type defaults baked in:

```js
import { http } from "react-invenio-forms";

await http.get("/api/records");
await http.post(url, payload);
```

Use `http` instead of a bare `axios` instance so CSRF cookies/headers and
the `application/vnd.inveniordm.v1+json` accept header are handled
consistently.

For any async call triggered from a component that might unmount before the
promise resolves (a common source of "setState on unmounted component"
warnings), wrap it with the also-exported `withCancel`:

```js
this.cancellable = withCancel(http.post(url, payload));
try {
  const response = await this.cancellable.promise;
} catch (error) {
  if (error !== "UNMOUNTED") this.setState({ error });
} finally {
  this.setState({ loading: false });
}
// componentWillUnmount:
this.cancellable?.cancel();
```

Normalize API errors before displaying them with a small helper mirroring
the common pattern: `error?.response?.data?.message || error?.message`.

## 6. Notifications and confirmation dialogs

There is **no toast/snackbar library** in this ecosystem (no react-toastify,
no custom Toastr) — use one of the two established patterns instead:

1. **Context-based dismissable banners** — a small controller component
   holds `{ notifications, addNotification, removeNotification }` in a
   React Context, rendered into a fixed container; consumers call
   `context.addNotification({ title, content, type: "success" })` on
   success, or keep an inline `error` in local state and render a Semantic
   UI `<Message floating role="alert">`-based `ErrorMessage`/`SuccessMessage`
   component on failure. Full API, auto-dismiss timing, and which packages
   actually reuse it vs. roll their own local error state:
   [reusable-widgets.md §1](reusable-widgets.md#1-notificationcontext--the-global-toast-system).
2. **Confirmation modals** are hand-built from plain `semantic-ui-react`
   `Modal`/`Button` (not the `Confirm` shorthand component — it's unused
   throughout this ecosystem). Every domain defines its own
   `*ConfirmationModal.js`/`*DeleteModal.js` following: local `loading`/
   `error` state → async handler in try/catch → `ErrorMessage` on failure,
   `onClose()`/notification on success. These modals are themselves usually
   `Overridable.component`-wrapped so a deployment can restyle or replace
   them without forking. This is one instance of a broader, recurring
   architecture (Controller-context + Button + Modal + Trigger) used for any
   async action, not just deletes — see
   [action-modals.md](action-modals.md) for the full pattern and three real
   side-by-side implementations, and
   [bulk-actions.md](bulk-actions.md) for triggering an action against
   multiple selected search results at once. For smaller, non-modal reusable
   pieces (loaders, error boundaries, inline async-save feedback, a
   danger-zone delete pattern, and more), see
   [reusable-widgets.md](reusable-widgets.md).
