---
name: oarepo-semantic-ui
description: Write and wire React components, JinjaX/Jinja2 templates, AND LESS/branding assets for the CESNET Invenio repository framework's Semantic UI theme — the "Model UI" layer built on top of vanilla Invenio (oarepo_ui, oarepo_rdm_ui, oarepo_vocabularies_ui, oarepo_requests, oarepo_dashboard, and CCMM as a worked example). Use when writing or editing deposit-form "sections", a model's forms/index.js or search/index.js entry, a UIResourceConfig/webpack.py/pyproject.toml entry-point for a new or existing model, a JinjaX `.jinja` component (record-detail partial, static page, `{# def #}` signature), a vocabulary field/form, a custom oarepo_requests widget, repository branding/theming (colors, fonts, logo, header/footer/frontpage under assets/less or templates/), or any *.js/*.jsx/*.jinja/*.html/*.less under an oarepo_* or ccmm_* package or repository root. Assumes the sibling invenio-semantic-ui skill's knowledge of react-invenio-forms/react-searchkit/Formik/react-overridable as a prerequisite and documents only what OARepo adds or does differently on top of it.
metadata:
  author: miroslav.simek@cesnet.cz
---

# OARepo/CCMM React + Semantic UI components

This skill covers the OARepo/NRP repository framework's frontend layer,
built on top of vanilla Invenio. **It assumes you already know the base
stack documented in the sibling `invenio-semantic-ui` skill** (react-invenio-forms
field components, react-searchkit, Formik, the generic `react-overridable`
mechanism, i18next conventions) — load that skill first if you haven't.
This skill documents only what OARepo's own packages add or do differently.

## The one big idea: one generic app shell, many models

Base Invenio has exactly one hardcoded `RDMDepositForm` and one search page
per resource. OARepo instead has **one generic, prop-injectable app shell**
(`DepositFormApp` for forms; `SearchApp` via `createSearchAppsInit` for
search) that every record model instantiates fresh, supplying its own
*sections* (a list of `{key, label, component, includesPaths}` objects),
its own serializer, and its own API client as plain props/injected classes.
A tiny, effectively-generated per-model webpack entry file does the wiring.
Nearly everything else in this skill is a consequence of this one design:
override-namespace isolation between models, the tab/wizard layer, the
`$schema`-based multi-model search dispatch, and the "sections" authoring
convention.

## Reference files

- **[references/wiring.md](references/wiring.md)** — the complete,
  real-repository wiring recipe: directory layout, the Python
  `UIResourceConfig`/`webpack.py`/`pyproject.toml` entry-points that make a
  model's pages actually run, and real JS entry points with real
  `parametrize`/override examples. Read this first when adding a new model
  or figuring out why something isn't loading.
- **[references/forms.md](references/forms.md)** — the `DepositFormApp`
  shell, the "section" object convention, the tab/wizard form layer,
  schema-driven label/help injection (not widget selection), and the
  OARepo-specific field components (multilingual strings, `StringArrayField`,
  ...). Read before writing or editing any deposit form section.
- **[references/search.md](references/search.md)** — `createSearchAppsInit`,
  `DynamicResultsListItem`'s `$schema`-based multi-model dispatch, and
  OARepo's reworked layout/facets/results (error-boundary wrapping,
  collapsible facets, a date-histogram facet, richer `ActiveFilters`). Read
  before writing or editing a search page or facet.
- **[references/conventions.md](references/conventions.md)** — the JinjaX
  templating bridge (a real additional layer, though the final React mount
  contract is unchanged from base Invenio), the `overridableIdPrefix`/
  `application_id` namespacing convention, and the `util.js` helper
  grab-bag (multilingual string resolution, Formik-error scrolling, ...).
  Read before wiring a new model's entry point or debugging why an override
  isn't matching.
- **[references/vocabularies.md](references/vocabularies.md)** —
  `oarepo_vocabularies_ui`'s form/search/detail apps and the generic,
  reusable `VocabularyField` picker. **Corrects a natural assumption**:
  there is no hierarchical tree-editor widget; hierarchy is handled via
  flat lists, breadcrumbs, and a `?h-parent=` query param.
- **[references/requests.md](references/requests.md)** —
  `oarepo_requests`. **Corrects another natural assumption**: there is no
  request-type-to-custom-form registry for the accept/decline modal; custom
  request UX is either a Python-registered per-type Label/Icon override or
  a fully standalone widget that bypasses the action-controller system.
- **[references/reusable-widgets.md](references/reusable-widgets.md)** — a
  catalog of smaller reusable pieces (`ClipboardCopyButton`, `IdentifierBadge`,
  the `Disabled` no-op override placeholder, `FacetsButtonGroupNameToggler`,
  and the standalone `@oarepo/file-manager` single-file upload/edit dialog).
- **[references/worked-example-ccmm.md](references/worked-example-ccmm.md)** —
  CCMM as a concrete, non-trivial example of building a model's sections on
  top of `oarepo_ui`/`oarepo_rdm_ui`, including a fully worked "reorderable
  list + bulk DOI import" custom field. Read this when building a new
  model's sections from scratch — it's the best template to imitate.
- **[references/jinjax-components.md](references/jinjax-components.md)** —
  writing/overriding a JinjaX `.jinja` component (the `{# def #}` syntax,
  the per-type `render_first_existing` override mechanism), the `IField`/
  `IValue` component family that's the actual toolkit for record-detail
  content, the reusable component catalog across packages, and how static
  pages are built. Read before touching a record-detail partial or a
  static/informational page — most of that work is **not** React.
- **[references/branding.md](references/branding.md)** — repository-wide
  look and feel: LESS color/font variables (the real numbered-color-scale
  pattern, not just one `@brandColor`), logos, and the header/footer/
  frontpage template overrides — grounded in a real repository's actual
  files. Read before touching anything under `assets/less/`,
  `static/images/`, or a repository-root `templates/` file.

## Core principles

1. **"Sections" are the atomic customization unit for a deposit form**, not
   individual fields. A section is `{key, label, component, includesPaths}`;
   models assemble an array of these and hand it to `DepositFormApp` as a
   prop. `oarepo_rdm_ui` ships three ready-made section-array presets
   (Minimal/Basic/Complete) as a starter kit — import one wholesale, or
   write your own array reusing only the section objects you need. See
   [forms.md](references/forms.md).

2. **Schema-driven generation only injects labels/help/required text — not
   widget choice.** `getFieldData()`/`mergeFieldData()` pull `label`/`help`/
   `hint`/`required` for a `fieldPath` out of the model's server-built
   `ui_model` tree, and every OARepo field wrapper calls it. Which
   component renders a field is still a manual choice in the section's
   source — don't expect a JSON-Schema type to auto-select a widget.

3. **Every model gets its own isolated override namespace, automatically.**
   `overridableIdPrefix` (forms) / `appName` (search) is derived server-side
   from the model's `application_id` (e.g. `"Datasets.Form"`,
   `"Datarepo.Search"`), not set manually per component. When adding a new
   overridable slot, build its ID from the `overridableIdPrefix` you're
   handed (usually via `tabConfig.formConfig.overridableIdPrefix` or
   react-searchkit's `buildUID`) — never hardcode a prefix.

4. **The override chain can span three+ layers.** A CCMM field can override
   a base `invenio_rdm_records` component (e.g.
   `InvenioRdmRecords.DepositForm.DatesField.DateField`), an `oarepo_rdm_ui`
   preset section can be reused or overridden by CCMM, and CCMM's own
   sections are themselves wrapped in `Overridable` so an *instance* of a
   CCMM-based repository can override them again. Check which layer you're
   actually overriding before assuming you need to touch the source
   package.

5. **The JinjaX layer is real, but the React mount contract is not new.**
   OARepo pages render through an actual JinjaX `Catalog` with typed
   `{# def #}` component signatures — but by the time data reaches React,
   it's the same `data-invenio-search-config` attribute / hidden-input
   `getInputFromDOM` convention as base Invenio. Don't assume a different
   bridge mechanism exists just because the template layer looks different.
   See [conventions.md](references/conventions.md).

6. **Multi-model search results dispatch by `$schema`, not by a fixed
   shape.** `DynamicResultsListItem` looks up a field (default `$schema`)
   on each hit and renders `Overridable id={buildUID("ResultsList.item", value)}`
   — register one result-item component per model's schema value, not one
   fixed component for the whole search app. See
   [search.md](references/search.md).

7. **Before building something new, check whether it's already thin/absent
   elsewhere, or already generic.** Several plausible-sounding subsystems
   turned out not to exist as hypothesized (a request-type form registry,
   a vocabulary tree editor, runtime multi-model dispatch in `oarepo_rdm_ui`)
   — read the relevant reference file's corrections before assuming the
   pattern you expect is there. Conversely, `VocabularyField`,
   `ClipboardCopyButton`, `IdentifierBadge`, and `@oarepo/file-manager` are
   genuinely generic and ready to reuse — see
   [vocabularies.md](references/vocabularies.md) and
   [reusable-widgets.md](references/reusable-widgets.md).

8. **A correctly-written Python resource class does nothing until it's
   registered as an entry point.** `webpack.py`'s `WebpackThemeBundle`,
   `create_blueprint`, and `finalize_app` only run if `pyproject.toml` has
   matching `invenio_assets.webpack`/`invenio_base.blueprints`/
   `invenio_base.finalize_app` entries pointing at them. If a new model's
   page 404s or its search/menu never appears, check `pyproject.toml`
   before suspecting the Python or React code. See
   [wiring.md §4](references/wiring.md#4-webpackpy-and-pyprojecttoml--the-actual-last-mile).

9. **To extend (not replace) an existing override, unwrap it with
   `.originalComponent` before rendering it inside your own override for
   the same ID** — rendering the plain exported (already-`Overridable`-
   wrapped) component from inside your override for that same ID creates an
   infinite override-lookup loop. See
   [wiring.md §5](references/wiring.md#5-the-js-entry-points-with-real-overrides).

10. **The record *detail* page is mostly server-rendered Jinja/JinjaX, not
    React.** Unlike deposit and search, there is no `RecordDetailApp` React
    shell — customizing how a detail page looks is a named-Jinja-block
    override task; React only mounts small islands within it (sharing
    button, clipboard/identifier widgets). The actual toolkit for that
    Jinja-side content is the `IField`/`IValue`/`IArray` component family,
    not hand-written `<dt>`/`<dd>` markup — see
    [jinjax-components.md §2](references/jinjax-components.md#2-the-i-data-field-family--the-toolkit-for-record-detail-content).

11. **JinjaX components have their own per-type override mechanism,
    parallel to but distinct from the React-side `$schema` dispatch (§6).**
    `catalog.render_first_existing(["Base.type", "EmptyComponent"], ...)`
    tries a dotted `<Base>.<type>` name before falling back — add a
    type-specific Jinja component this way instead of editing the generic
    one. See
    [jinjax-components.md §1](references/jinjax-components.md#invocation).

12. **`ccmm_invenio` has zero Jinja/JinjaX template files — all of its UI
    is React**, and `oarepo_requests` has no Jinja/JinjaX UI components
    either (request labels/badges are React-only). Don't go looking for a
    server-rendered equivalent of something that's genuinely React-only in
    those two packages. See
    [jinjax-components.md §3](references/jinjax-components.md#3-other-reusable-components--by-package).

13. **Repository-wide branding LESS lives at `assets/less/...`, not
    `assets/semantic-ui/less/...`** — unlike a Python package's own theme
    assets, the top-level instance override root has no `semantic-ui/`
    segment, and is picked up by convention with no `webpack.py`
    entry-point needed (unlike a model's own webpack bundle). See
    [branding.md](references/branding.md#file-locations).

14. **Some `invenio-theme` styles can't be reached by any LESS variable and
    must be force-overridden by CSS selector with `!important`** — if a
    variable you defined isn't taking effect, that may be why; check for
    this case before assuming a naming mistake. See
    [branding.md §2](references/branding.md#2-style-overrides-beyond-variables).

## Wiring a new model (or a new page within one) — quick procedure

Usually skippable: a model's `ui/<model>/` tree is scaffolded automatically
when the model is created (per that directory's own `README.md`), so you're
typically editing an existing tree, not building it from scratch. Full
detail and a real worked example: [wiring.md](references/wiring.md).

1. Confirm the model has a `UIResourceConfig` (extending `RecordsUIResourceConfig`,
   or a metadata-profile-specific subclass of it like CCMM's) with
   `model_name`, `application_id`, `blueprint_name`, `url_prefix` set.
2. Confirm `pyproject.toml` has matching `invenio_assets.webpack` and
   `invenio_base.blueprints` entries (plus `invenio_base.finalize_app` if
   the model registers menu entries or UI overrides) pointing at that
   model's `webpack.py`/`create_blueprint`/`finalize_app` — nothing runs
   without this, regardless of how correct the Python/React code is.
3. Only create/edit a template partial (e.g. `record_detail/main.html`)
   when you need to override one specific named block — the top-level page
   templates are almost always inherited unchanged.

## Writing a new model's deposit form — quick procedure

1. Decide whether an `oarepo_rdm_ui` preset (`RDMMinimalSections`/
   `RDMBasicSections`/`RDMCompleteSections`) already covers your needs —
   import it wholesale if so.
2. Otherwise, write your own `sections` array of `{key, label, component,
   includesPaths}` objects, reusing base `invenio_rdm_records` field
   components, `oarepo_ui` field wrappers, and/or `oarepo_rdm_ui`'s
   individual section/component exports as building blocks — see the
   worked example in [worked-example-ccmm.md](references/worked-example-ccmm.md).
3. Wrap each section's rendered content in
   `<Overridable id={buildUID(overridableIdPrefix, "YourSectionKey")}>` so
   downstream repositories can override it.
4. Write the model's `forms/index.js` webpack entry that calls
   `parseFormAppConfig()`, constructs your serializer (or reuses
   `RDMDepositRecordSerializer`/`OARepoDepositSerializer`), and renders
   `<DepositFormApp config={config} sections={YourSections}
   recordSerializer={recordSerializer} ... />` — see
   [wiring.md](references/wiring.md) for a complete real example including
   `parametrize`-based field overrides.
5. If the form needs more than one tab/step, pass `useWizardForm` and rely
   on the built-in tab/wizard layer rather than building your own
   navigation — see [forms.md](references/forms.md).

## Writing a new model's search page — quick procedure

1. Confirm the page needs OARepo's search bootstrap at all — most of the
   Python-side config (`SearchAppConfig`/`FacetsConfig`) is reused unchanged
   from base `invenio_search_ui`.
2. Use `createSearchAppsInit` (plural — OARepo's own, not base Invenio's
   `createSearchAppInit`) and read `overridableIdPrefix` from
   `parseSearchAppConfigs()` rather than hardcoding an app name.
3. If a *different*, aggregated page needs to render this model's results
   generically alongside other models', register a `search_component`
   (`UIComponent(...)`) on the model's `UIResourceConfig` and call
   `current_oarepo_ui.register_result_list_item(...)` — that's what feeds
   `DynamicResultsListItem`'s `$schema`-keyed dispatch. The model's *own*
   dedicated search page doesn't need this; it just registers
   `ResultsList.item` under its own `overridableIdPrefix` directly.
4. Reuse OARepo's layout pieces (`SearchAppLayout`, `SearchAppFacets`,
   `FoldableBucketAggregationElement`, the date-histogram facet) rather
   than react-searchkit's bare defaults — they add responsive layout and
   per-facet/per-item error boundaries the base components don't have.

## Customizing a record-detail page, or building a static page — quick procedure

This is Jinja/JinjaX work, not React — see
[jinjax-components.md](references/jinjax-components.md) for everything
below.

1. Don't edit the top-level page template (`record_detail.html`, etc.) —
   create/edit the specific named partial it documents in its comment
   (e.g. `record_detail/main.html`) and override just the one named block
   you need, calling `super()` to keep the rest.
2. Compose the block's content from the `IField`/`IValue`/`IArray`/
   `ISection` family, not hand-written `<dt>`/`<dd>`/`<table>` markup —
   this family already handles the label/placeholder lookup the same way
   the React field wrappers do.
3. If you need type-specific rendering (e.g. one behavior for vocabulary
   type A, another for type B), write `catalog.render_first_existing(
   ["Base.yourType", "EmptyComponent"], ...)` and add a `Base.yourType`
   component, rather than branching inside the generic component.
4. For a small piece of reusable formatting logic called several times
   within one partial (not a full component), write a plain Jinja2
   `{% macro %}` in your own template tree and import it `with context` —
   don't reach for a full JinjaX component just for a formatting helper.
5. For a standalone page unrelated to any record model, use
   `TemplatePageUIResource`/`TemplatePageUIResourceConfig`
   ([wiring.md §7](references/wiring.md#7-wiring-a-plain-non-model-page-a-smaller-worked-example)),
   extend `"page.html"`, and hand-write the body — don't reach for the
   record-detail component family, it's built around record data a static
   page doesn't have.

## Rebranding a repository (colors, fonts, logo, header/footer) — quick procedure

See [branding.md](references/branding.md) for everything below.

1. Colors: define a numbered color scale per family in
   `assets/less/site/globals/site.variables`, derive semantic variables
   (`@primaryColor`, `@secondaryColor`, ...) from it, and set
   `@navbarBackgroundColor`/`@footerLightColor`/`@footerDarkColor` from
   those — don't just set one flat `@brandColor` and stop.
2. Logo: put the file(s) in `static/images/`, set `THEME_LOGO` in
   `invenio.cfg`, and if you need per-locale variants, switch on
   `current_i18n.language` inside a `header.html` `brand` block override.
3. Fonts: put font files in `assets/less/site/fonts/`, declare `@font-face`
   in `site.overrides`, and reference the family by a variable
   (`@fontName`) rather than a hardcoded string.
4. Header/footer/frontpage: create the same-relative-path template under
   your repository's `templates/`, `{% extends %}` the default, and
   override only the specific named block you need — the same convention
   used for model page templates.
5. If a LESS variable you defined isn't visibly taking effect, check
   whether the target style is one of the few that must be
   selector-overridden directly (principle 14) before assuming a naming
   mistake.

See the reference files for the full, sourced detail behind every point
above.

## Further reading

The official NRP/OARepo "Model UI" customization docs (Python-side
`RecordsUIResourceConfig`/JinjaX template customization, complementary to
this skill's React focus): https://nrp-cz.github.io/docs/customize/model_ui
