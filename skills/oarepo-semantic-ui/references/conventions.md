# Cross-cutting OARepo conventions

Read this file for conventions that apply across both forms and search:
the JinjaX templating bridge, the `overridableIdPrefix`/`application_id`
namespacing convention, and the shared `util.js` helper grab-bag. The
generic `react-overridable` mechanism itself, i18next conventions, and the
`http`/`withCancel` HTTP client are **unchanged** from base Invenio — see
the sibling `invenio-semantic-ui` skill's `conventions.md` for those; this
file only covers what OARepo adds on top.

## The JinjaX-to-React bridge

OARepo pages render through a real [JinjaX](https://jinjax.scaletti.dev/)
`Catalog`, wired into Flask's existing Jinja environment
(`oarepo_ui/ext.py`). JinjaX component files use a `{# def ... #}` block to
declare typed parameters, e.g. (`templates/oarepo_ui/pages/RecordSearch.jinja`):

```jinja
{# def
  search_app_config, ui_config, ui_resource, ui_links,
  webpack_entry, model_name, extra_context, context,
#}
{% extends model_name ~ "/record_search.html" %}
```

The macro to render is resolved from a name→dotted-path mapping on
`RecordsUIResourceConfig`:

```python
templates: Mapping[str, str | None] = {
    "record_detail": "oarepo_ui.pages.RecordDetail",
    "search": "oarepo_ui.pages.RecordSearch",
    "deposit_edit": "oarepo_ui.pages.DepositEdit",
    "deposit_create": "oarepo_ui.pages.DepositCreate",
    "tombstone": "oarepo_ui.pages.Tombstone",
    "not_found": "oarepo_ui.pages.NotFound",
}
```

Each JinjaX page component `{% extends model_name ~ "/record_search.html" %}`
— a **plain, non-JinjaX Jinja2 template per model** — which itself typically
just `{% extends "oarepo_ui/record_search.html" %}` (or, for deposit pages,
ultimately `{% extends "invenio_app_rdm/records/deposit.html" %}`, i.e. base
Invenio's own template). **By the time data reaches the DOM, the contract is
identical to base Invenio**:

```jinja
{# oarepo_ui/record_search.html #}
<div data-invenio-search-config='{{ search_app_config | tojson }}'></div>
```

```jinja
{# oarepo_ui/deposit_edit.html, extends invenio_app_rdm's own deposit template #}
<input id="deposits-record" type="hidden" name="deposits-record" value='{{ record | tojson }}'>
<input type="hidden" name="deposits-config" value='{{ forms_config | tojson }}'>
<div id="deposit-form"></div>
```

**Take-away**: JinjaX adds a real, additional layer of typed component
composition and per-model template inheritance chains — but it does not
change how React gets its data. `getInputFromDOM`/`parseFormAppConfig`
(forms) and the `data-invenio-search-config` attribute (search) are exactly
the mechanisms documented in the sibling skill. What differs is *which JS
bundle* gets loaded for a given page: each model registers its own webpack
entries (`{model}_deposit_form.js`, `{model}_search.js`, ...), and the
template's `{{ webpack[webpack_entry] }}` block loads the model-specific
bundle rather than a single fixed one.

For record-detail-page React islands that aren't full search/form apps
(sharing button, clipboard-copy widgets), OARepo skips JinjaX config-passing
entirely and uses the plain `<div data-*>` + small dedicated
`ReactDOM.render` bootstrap pattern, one webpack entry per widget — see
[reusable-widgets.md](reusable-widgets.md) for concrete examples.

This section covers only the *bridge* to React. For how to actually write
or override a `.jinja` component (the `{# def #}` signature syntax, how
components resolve to files, the per-type `render_first_existing` override
mechanism) and the full catalog of reusable JinjaX components/macros
shipped across these packages, see
[jinjax-components.md](jinjax-components.md) — most record-detail and
static-page customization is this kind of work, not React.

## The `overridableIdPrefix`/`application_id` namespacing convention

Every model has an `application_id` (e.g. `"datasets"`), set on its Python
`RecordsUIResourceConfig` subclass. The server derives a per-model override
namespace from it and injects that into the config the React app reads:

```python
# forms
form_config["overridableIdPrefix"] = f"{self.application_id.capitalize()}.Form"
# search
search_app_config["overridableIdPrefix"] = f"{self.config.application_id.capitalize()}.Search"
```

So `datasets` gets `"Datasets.Form.*"`/`"Datasets.Search.*"` (or whatever
the deployment names the config's `appName`, e.g. `"Datarepo.Search"` seen
in one real config) automatically — **you never hardcode this prefix**; you
always read it from `tabConfig.formConfig.overridableIdPrefix` (forms) or
`parseSearchAppConfigs()`'s returned config (search) and build IDs with
`buildUID(overridableIdPrefix, "YourSlot")`. This is the same underlying
`buildUID`/`Overridable` mechanism as base react-searchkit — OARepo doesn't
invent a new ID scheme, it just automates generating the namespace segment
per model instead of requiring a manually-set `appName`.

Because the namespace is generated, not hand-assigned, two different models
never collide even though both mount the exact same generic
`DepositFormApp`/`SearchApp` component tree — this is what makes the
"one shell, many models" design in [forms.md](forms.md) safe.

## `util.js` grab-bag

`oarepo_ui/util.js` (plus a matching `util.test.js`) exports a set of
shared helpers with no other natural home. Worth knowing before
reimplementing any of them:

- **`getInputFromDOM(elementName)`** — the hidden-input DOM-config reader
  used by forms (distinct from search's `data-*` attribute mechanism).
- **`scrollTop()` / `scrollToElement(fieldPath)`** — smooth-scroll to a
  nested Formik field path's `label[for="..."]` or `#id`, trying
  progressively shorter path prefixes. Built for "jump to the first
  validation error" UX in deeply nested forms.
- **`object2array(obj)` / `array2object(arr)`** — bidirectional transforms
  between `{key: value}` maps and `[{keyName, valueName}]` arrays, used by
  multilingual-field editing UIs (see `MultilingualTextInput` in
  [forms.md](forms.md#6-oarepo-specific-field-components)).
- **`collectNestedErrors(obj, basePath)`** — recursively flattens a nested
  Formik/Yup error object (including arrays) into `[{errorPath,
  errorMessage}]` — used to build one combined error list for deeply
  nested records.
- **`unique(value, context, path, errorString)`** — a Yup custom-test
  helper enforcing array-item uniqueness by key.
- **`getLocalizedValue(multilingualData, defaultFallback)`** — resolves the
  best-fit localized string from either `{lang: value}` or `[{lang, value}]`
  shapes, with defined precedence: exact locale → base language → `"en"` →
  i18next `fallbackLng` → any non-`"und"` value → `"und"` → fallback. This
  is *the* canonical way to render a multilingual field read-only; don't
  hand-roll locale fallback logic elsewhere.
- **`getLocaleObject()` / `getDefaultLocale()` / `formatDate()`** —
  date-fns locale plumbing reading `window.__localeData__`/
  `window.__localeId__`, so `formatDate` renders dates in the current UI
  locale.
- **`goBack(fallBackURL = "/")`** — uses `document.referrer` to choose
  between `window.history.back()` and a hard redirect, avoiding "back"
  leaving the app entirely when a page was opened directly (bookmark/shared
  link) rather than navigated to from within the app.
- **`httpApplicationJson` / `httpVnd`** — two pre-configured `axios`
  instances (CSRF wiring; differing `Accept` headers —
  `application/json` vs `application/vnd.inveniordm.v1+json`). The
  in-source comment notes these are a stand-in "until we start using v4 of
  react-invenio-forms" — treat as the current OARepo-side HTTP convention
  for ad hoc calls outside the main deposit API client, not a permanent
  abstraction to build more on top of.
- **`encodeUnicodeBase64` / `decodeUnicodeBase64`** — Unicode-safe base64
  (`btoa(encodeURIComponent(...))` and its inverse).
- **`timestampToRelativeTime(timestamp)`** — Luxon-based, i18next-locale-aware
  relative time string ("4 days ago") — the OARepo-side equivalent of
  react-invenio-forms' `toRelativeTime`.
