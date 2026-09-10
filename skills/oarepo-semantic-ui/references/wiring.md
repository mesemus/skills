# Wiring a model end-to-end (real example: `datasets`/CCMM)

The other reference files describe the *pieces* (`DepositFormApp`,
`createSearchAppsInit`, sections, JinjaX...). This file walks the complete,
real wiring recipe that connects them into a running page, using an actual
production model (`datasets`, built on CCMM, from a real OARepo repository)
as the worked example throughout. Read this when adding a new model to a
repository, or when something isn't loading and you need to trace the whole
chain from Python entry point to mounted React component.

## 1. Directory layout

```
ui/datasets/
  __init__.py                              # UIResourceConfig, UIResource, app hooks
  webpack.py                               # WebpackThemeBundle: JS entries + @js alias
  semantic-ui/js/datasets/
    forms/index.js                         # deposit form entry point
    search/index.js                        # search page entry point
    search/ResultsListItem.jsx             # this model's result item
  templates/semantic-ui/datasets/
    deposit_create.html, deposit_edit.html
    record_search.html, record_detail.html, record_detail/main.html
    tombstone.html, not_found.html
    macros/related_resources.html          # a model-specific Jinja macro
```

This whole tree (except the files you actually customize) is scaffolded
automatically when a model is created — the `ui/README.md` convention in a
real repository says exactly this: *"When you create a new model, the UI
for the model will be created automatically... Please modify the sources,
they will not be touched later."* Expect to find this shape already
present and mostly working; your job is usually to edit a handful of files
in it, not to create the tree from scratch.

## 2. The Python resource config

```python
# ui/datasets/__init__.py
from ccmm_invenio.ui.config import CCMMRecordsUIResourceConfig
from ccmm_invenio.ui.resource import CCMMRecordsUIResource
from oarepo_ui.overrides import UIComponent
from oarepo_ui.overrides.components import UIComponentImportMode
from oarepo_ui.proxies import current_oarepo_ui
from oarepo_ui.utils import can_view_deposit_page

class DatasetsUIResourceConfig(CCMMRecordsUIResourceConfig):
    template_folder = "templates"
    url_prefix = "/datasets"
    blueprint_name = "datasets_ui"
    model_name = "datasets"
    application_id = "datasets"

    search_component = UIComponent(
        "DatasetsResultsListItem",
        "@js/datasets/search/ResultsListItem",
        UIComponentImportMode.DEFAULT,
    )
    components = (*CCMMRecordsUIResourceConfig.components,)

class DatasetsUIResource(CCMMRecordsUIResource):
    """A resource for datasets records."""
```

Note the **Python-side inheritance chain mirrors the JS override chain**
from [worked-example-ccmm.md](worked-example-ccmm.md): base `oarepo_ui`
`RecordsUIResourceConfig` → `CCMMRecordsUIResourceConfig` (the metadata
profile's own config, from `ccmm_invenio.ui.config`) →
`DatasetsUIResourceConfig` (this specific model). The `UIResource` subclass
itself is typically empty — nearly all customization happens in the config
class's attributes, not by overriding resource methods.

### `UIComponent` — the Python-side component registration primitive

`UIComponent(name, import_path, mode)` describes a React component to be
dynamically imported and registered as an override, without the requesting
JS bundle needing to import it directly:

```python
UIComponent("DatasetsResultsListItem", "@js/datasets/search/ResultsListItem", UIComponentImportMode.DEFAULT)
```

- `name` — an identifier for the registration (used as the generated
  variable name / registry key).
- `import_path` — a module path using the same `@js/<pkg>` alias convention
  as everywhere else.
- `mode` (`UIComponentImportMode.DEFAULT` here) — how the module is
  imported (default vs. named export).

Setting `search_component` on the config and calling
`current_oarepo_ui.register_result_list_item(json_schema, search_component)`
(done in `ui_overrides()`, §3) is what feeds
[search.md](search.md#2-multi-model-result-dispatch-dynamicresultslistitem)'s
`DynamicResultsListItem` `$schema`-based dispatch table — this is the
missing half of that mechanism: `DynamicResultsListItem` is the *consumer*
of a dispatch table, and `register_result_list_item` is the *producer*,
called once per model at app-init time. `oarepo_requests`' per-request-type
Label/Icon registration (see [requests.md](requests.md)) uses the sibling
`UIComponentOverride` class for the analogous "register under a Flask
endpoint" case — same underlying idea, different registration function.

**Note the model's own dedicated search page does NOT go through this
mechanism** — its `search/index.js` (§5) registers `ResultsListItem`
directly under `${overridableIdPrefix}.ResultsList.item` via
`componentOverrides`, a completely ordinary override. `register_result_list_item`
only matters for a *different*, aggregated/multi-model search page that
needs to render results from several models generically (e.g. a sitewide
search) — don't assume you need it for a model's own single-model search
page.

## 3. App-lifecycle hooks

```python
def ui_overrides(_app: Flask) -> None:
    """Register UI overrides."""
    ui_resource_config = DatasetsUIResourceConfig()
    if (current_oarepo_ui is not None and ui_resource_config.model
            and ui_resource_config.model.record_json_schema
            and ui_resource_config.search_component):
        current_oarepo_ui.register_result_list_item(
            ui_resource_config.model.record_json_schema,
            ui_resource_config.search_component,
        )

def init_menu(app: Flask) -> None:
    """Initialize menu before first request."""
    with app.app_context():
        current_menu.submenu("plus.create_datasets").register(
            f"{DatasetsUIResourceConfig().blueprint_name}.deposit_create",
            _("New Dataset"), order=1, visible_when=can_view_deposit_page,
        )
        # ... more submenu registrations (docs links, "About" links, etc.)

def finalize_app(app: Flask) -> None:
    """Finalize app."""
    init_menu(app)
    ui_overrides(app)

def create_blueprint(_app: Flask) -> Blueprint:
    """Register blueprint for this resource."""
    return DatasetsUIResource(DatasetsUIResourceConfig()).as_blueprint()
```

- `init_menu` registers the model into the site's navigation (e.g. a
  "Create new..." menu) via Flask-Menu, gated by a visibility predicate —
  `can_view_deposit_page` from `oarepo_ui.utils` is the standard permission
  check for "should this menu entry be shown."
- `finalize_app` is the conventional hook name for "run this once the Flask
  app is fully assembled" — it's where menu registration and UI-override
  registration both happen for this model.
- `create_blueprint` is the factory the entry-points system (§4) calls to
  get this model's Flask blueprint.

## 4. `webpack.py` and `pyproject.toml` — the actual "last mile"

None of the above runs unless it's registered as a Python entry point.
This is the wiring step that's easy to forget when adding something new,
because everything upstream of it looks complete on its own.

```python
# ui/datasets/webpack.py
from invenio_assets.webpack import WebpackThemeBundle

theme = WebpackThemeBundle(
    __name__, ".", default="semantic-ui",
    themes={"semantic-ui": dict(
        entry={
            "datasets_search": "./js/datasets/search/index.js",
            "datasets_deposit_form": "./js/datasets/forms/index.js",
        },
        aliases={"@js/datasets": "./js/datasets"},
    )},
)
```

```toml
# pyproject.toml, at the repository root
[project.entry-points."invenio_assets.webpack"]
components  = "ui.components.webpack:theme"
ui_datasets = "ui.datasets.webpack:theme"
ui_pages    = "ui.pages.webpack:theme"

[project.entry-points."invenio_base.blueprints"]
ui_datasets = "ui.datasets:create_blueprint"
ui_pages    = "ui.pages:create_blueprint"

[project.entry-points."invenio_base.finalize_app"]
ui_datasets = "ui.datasets:finalize_app"
```

Every model needs **three** entry-point registrations (webpack theme,
blueprint, and — only if it needs menu/UI-override registration like
`init_menu`/`ui_overrides` — `finalize_app`); a plain, non-model page
package (like `ui.pages` below) typically only needs the first two. The
entry-point *name* (left-hand side, e.g. `ui_datasets`) just needs to be
unique across the whole app — it's not otherwise meaningful — but the
right-hand side must point at the exact importable callable/object.
**If a new model's page 404s, its search doesn't appear, or its
`finalize_app` hook never runs, check `pyproject.toml` first** — a correct
`ui/<model>/__init__.py` with no matching entry-point line simply never
executes.

The `components` entry above is a different kind of registration — it's
not a record model, it's the repository-wide custom CSS/JS bundle (no
matching `invenio_base.blueprints` entry, since it isn't its own page).
See [branding.md](branding.md#5-page-chrome-templates-header-footer-frontpage)
for where that bundle actually gets included on every page.

## 5. The JS entry points, with real overrides

The base shape of `forms/index.js` — `parseFormAppConfig()`, constructing a
`recordSerializer`, rendering `<DepositFormApp>` — is exactly the pattern
shown in [forms.md §1](forms.md#1-one-generic-shell-model-supplied-everything-else);
the real file adds two override-related lines worth studying in full:

```js
// ui/datasets/semantic-ui/js/datasets/forms/index.js (additions over the base shape)
import { EDTFSingleDatePicker } from "@js/oarepo_ui/forms";
import { parametrize } from "react-overridable";
import { SubmitReviewModal } from "@js/invenio_rdm_records";

const parametrizeEDTFSingleDatePicker = parametrize(EDTFSingleDatePicker, {
  customInputProps: { width: 16 },
});

// Unwrap the Overridable to avoid the override-lookup loop when we render it below.
const RawSubmitReviewModal = SubmitReviewModal.originalComponent;
const CurationPolicySubmitReviewModal = (props) => (
  <RawSubmitReviewModal {...props} afterContent={() => <p>...curation policy link...</p>} />
);

export const componentOverrides = {
  "InvenioRdmRecords.DepositForm.DatesField.DateField": parametrizeEDTFSingleDatePicker,
  "InvenioRdmRecords.SubmitReviewModal.container": CurationPolicySubmitReviewModal,
};

ReactDOM.render(
  <DepositFormApp config={config} {...rest} sections={CCMMSections}
    recordSerializer={recordSerializer} componentOverrides={componentOverrides} useWizardForm />,
  rootEl
);
```

Two genuinely important, non-obvious things this real example demonstrates:

- **`componentOverrides` is passed as a prop directly to `DepositFormApp`**,
  not registered globally via `overrideStore.add()` beforehand. `parametrize`
  (from `react-overridable`, the sibling skill's `conventions.md`) is the
  standard way to inject extra fixed props (here, `customInputProps`,
  `afterContent`) into an existing component before registering it as an
  override — you rarely write a brand-new component from scratch just to
  add one prop.
- **`.originalComponent` — the way to wrap, not replace, an existing
  override target.** `SubmitReviewModal` (as exported from
  `@js/invenio_rdm_records`) is already `Overridable.component`-wrapped
  under its own ID. If you render that exported component *inside your own
  override for that same ID*, you get an infinite override-lookup loop —
  your override renders `SubmitReviewModal`, which looks itself up in the
  override map, finds your override again, and renders it again. The fix,
  shown verbatim in this real code, is to reach for the **unwrapped
  original** via the `.originalComponent` property that `Overridable.component()`
  stashes on the wrapper (documented generically in the sibling skill's
  `conventions.md`), render *that*, and add your own content around it.
  **Any time you want to extend rather than fully replace an existing
  overridable component from inside its own override, use
  `Component.originalComponent`, not the plain exported name.**

```js
// ui/datasets/semantic-ui/js/datasets/search/index.js
import { parseSearchAppConfigs, createSearchAppsInit, SearchAppFacets, SearchAppLayout } from "@js/oarepo_ui/search";
import ResultsListItem from "./ResultsListItem";
import { parametrize } from "react-overridable";

const [{ overridableIdPrefix }] = parseSearchAppConfigs();

const SearchAppFacetsWithTitle = parametrize(SearchAppFacets, { title: i18next.t("Data Catch-all Repository") });
const SearchAppLayoutWithTip = parametrize(SearchAppLayout, {
  searchBarTip: i18next.t("TIP: Most of the content is in English..."),
});

export const componentOverrides = {
  [`${overridableIdPrefix}.ResultsList.item`]: ResultsListItem,
  [`${overridableIdPrefix}.SearchApp.facets`]: SearchAppFacetsWithTitle,
  [`${overridableIdPrefix}.SearchApp.layout`]: SearchAppLayoutWithTip,
};

createSearchAppsInit({ componentOverrides });
```

Same `parametrize` idiom again, this time to inject a `title` into
`SearchAppFacets` and a `searchBarTip` string into `SearchAppLayout` — both
documented as bare "slots" in [search.md](search.md), now shown wired up
with real deployment-specific copy.

## 6. The templates are mostly empty extends-stubs

Every one of this real model's page templates (`deposit_create.html`,
`deposit_edit.html`, `record_search.html`, `record_detail.html`,
`tombstone.html`, `not_found.html`) is a ~15-line file containing nothing
but `{% extends "oarepo_ui/<name>.html" %}` and a documentation comment
listing the inheritance chain and the named partials/blocks available to
override. Verbatim example (`record_detail.html`):

```jinja
{% extends "oarepo_ui/record_detail.html" %}
{# Template inheritance chain:
   oarepo_ui.pages.RecordDetail (JinjaX) → this template → oarepo_ui/record_detail.html → invenio_app_rdm/records/detail.html

   To customize, create/edit these partials (they extend oarepo_ui defaults):
   - record_detail/banners.html - Banner blocks (community, preview, version)
   - record_detail/main.html - Record body blocks (header, title, content, files, details)
   - record_detail/css.html / head_meta.html / javascript.html - optional extras
   - record_detail/record_sidebar.html - Sidebar content
#}
```

**In practice, you almost never edit these top-level stub files** — you
create the *named partial* they mention (e.g. `record_detail/main.html`)
only when you actually need to customize that specific piece, and even then
you extend the default partial and override one named block, calling
`super()` to keep the rest:

```jinja
{# ui/datasets/templates/semantic-ui/datasets/record_detail/main.html #}
{% extends "oarepo_ui/record_detail/main.html" %}
{%- from "datasets/macros/related_resources.html" import related_resources_section with context -%}
{%- set m = d.metadata -%}

{%- block record_footer -%}
  {{ super() }}
  {%- if m.related_resources -%}
    {{ related_resources_section(as_array(m.related_resources)) }}
  {%- endif -%}
{%- endblock record_footer -%}
```

Note the `{% from ... import ... with context %}` — required whenever the
imported macro needs Jinja globals (`_`, `ui_value`, `value`, `as_array`)
rather than receiving everything as an explicit argument; every real macro
import in this codebase uses it. `related_resources_section` itself is a
repository-local macro file
(`datasets/templates/.../macros/related_resources.html`), not something
`ccmm_invenio` or `oarepo_ui` ships — writing your own small macro file
like this one, for logic reused several times within one partial, is a
normal and expected pattern (see
[jinjax-components.md §4](jinjax-components.md#4-plain-jinja2-macros--the-minority-pattern)).

**Important scoping note**: the record **detail** page is primarily
server-rendered Jinja2/JinjaX with named blocks (`record_header`,
`record_title`, `record_content`, `record_files`, `additional_record_details`,
`record_footer`, ...) — it is **not** a React app the way deposit and
search pages are. Customizing "how a record's detail page looks" is mostly
a Jinja block-override task, not a React component task. React only mounts
small islands *within* this server-rendered page (the sharing button,
clipboard-copy buttons, identifier badges — see
[reusable-widgets.md](reusable-widgets.md)) — don't go looking for a
`RecordDetailApp`-equivalent React shell; there isn't one. For the actual
component toolkit you compose a partial like the one above from (the
`IField`/`IValue` data-field family, vocabulary display components, and
how to write a brand-new JinjaX component), see
[jinjax-components.md](jinjax-components.md).

## 7. Wiring a plain, non-model page (a smaller worked example)

Not everything is a record model. `ui.pages` in the same repository shows
the minimal pattern for a standalone informational page with one small
React island — the "Get access" onboarding page:

```python
# ui/pages/__init__.py
from oarepo_ui.resources import TemplatePageUIResource, TemplatePageUIResourceConfig

class PagesUIResourceConfig(TemplatePageUIResourceConfig):
    template_folder = "templates"
    url_prefix = "/"
    blueprint_name = "pages_ui"
    application_id = "pages"
    pages: Mapping[str, str] = {"get-access": "GetAccess"}

class PagesUIResource(TemplatePageUIResource):
    def create_url_rules(self) -> list:
        from flask_resources import route
        return [route("GET", "get-access", self.get_access)]

    def get_access(self) -> str:
        return self.render(page="GetAccess")
```

```jinja
{# ui/pages/templates/GetAccess.jinja #}
{% extends "page.html" %}
{% block javascript %}
  {{ super() }}
  {{ webpack['get_access.js'] }}
{% endblock javascript %}
{% block page_body %}
  ...
  <div id="standalone_submitter_application"></div>
{% endblock %}
```

```js
// ui/pages/semantic-ui/js/get_access/index.js
import { GetAccessButton } from "@js/oarepo_requests/get_access";

const domContainer = document.getElementById("standalone_submitter_application");
if (domContainer) {
  ReactDOM.render(
    <GetAccessButton groupId="submitters" groupName={i18next.t("Submitters")} />,
    domContainer
  );
}
```

Note **`GetAccessButton` is imported from `@js/oarepo_requests/get_access`**
— it's the generic, reusable widget documented in
[requests.md](requests.md#custom-request-creation-ux-bypass-the-action-system-entirely),
parameterized per-deployment via plain props (`groupId`, `groupName`) at
the mount site. `TemplatePageUIResource`/`TemplatePageUIResourceConfig` (for
plain pages, as opposed to `RecordsUIResourceConfig` for record models) is
the base class to use for any standalone page that isn't tied to a record
model — a `pages` dict maps URL-path-safe keys to JinjaX macro names, and
`create_url_rules`/one method per page renders each. `ui.pages` has **no**
`invenio_base.finalize_app` entry point in `pyproject.toml` — plain pages
that don't need menu/UI-override registration can skip that hook entirely.
