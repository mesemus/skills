# JinjaX components — authoring, the reusable catalog, and static pages

Read this when writing or overriding a JinjaX component, a record-detail
Jinja partial, or a static/informational page — anything that is **not**
a React island. [conventions.md](conventions.md#the-jinjax-to-react-bridge)
and [wiring.md](wiring.md#6-the-templates-are-mostly-empty-extends-stubs)
cover the page-level `{% extends %}`/named-block chain and the bridge to
React; this file covers the component layer underneath that — how a
`.jinja` file itself is written, and what's already available to compose
one from.

## 1. Authoring a `.jinja` component

A component is a file (default extension `.jinja`) with an optional
`{# def ... #}` signature comment as its first line(s):

```jinja
{# def copyText #}                                    {# required param #}
{#def identifier=None, fallbackImage="/static/..." #} {# defaulted params, no space after {# also works #}
{#def record, permissions, groups_enabled #}          {# multiple required params #}
{# def                                                 {# multi-line, trailing commas OK #}
  record, record_ui, files, community = None, embedded = False,
#}
```

**No `{# def #}` at all is valid** if the component doesn't need typed/
defaulted params — a plain Jinja page template using `{% extends %}`/
`{% block %}` is a perfectly good "component" as far as the catalog is
concerned (real example: `GetAccess.jinja`, see §5). Any kwargs passed to
a component with no signature are simply not bound to local names — no
error is raised, which is exactly how `EmptyComponent.jinja` (literally
`<span></span>`) works as a **no-op placeholder** for a pluggable slot: it
accepts and ignores whatever kwargs the caller passes.

### The implicit `content` slot

A component doesn't need to declare `content` in its `{# def #}` to use
it — whatever was placed between the opening/closing tags at the call site
is available as `{{ content }}`. A common pattern combines this with a
fallback to a computed default when no children were supplied:

```jinja
{%- if content and ((content is not string) or content.strip()) -%}
  {{ content }}
{%- else -%}
  <IValue value={{ ui_value(d) }} />
{%- endif -%}
```

### Invocation

- **As a tag**, resolved by the catalog from a PascalCase name:
  ```jinja
  <IURL rel="noopener noreferrer" href={{pidObject.url | e}} title={{pidObject.identifier}} />
  <ClipboardCopyButton copyText={{pidObject.url | e}} />
  <DefinitionLink href={definitionLink}>{{ term }}</DefinitionLink>
  ```
  Attribute values can be `{{ jinja_expression }}` (standard Jinja) **or**
  bare `{expr}` (JinjaX's own curly-brace attribute syntax) — both are used
  throughout this codebase; don't be surprised by either.
- **From Python**, via `catalog.render(name, **kwargs)` — the entry point
  every UI resource uses to turn a request into HTML (already shown for
  `record_detail`/`search`/`deposit_edit` in
  [conventions.md](conventions.md) and [wiring.md](wiring.md)).
- **From inside a template**, via `catalog.irender(name, **kwargs)` — for
  dynamically-named components, e.g. resolving a custom field's configured
  renderer:
  ```jinja
  {% if cf.props.landing_page_component %}
    {{ catalog.irender(cf.props.landing_page_component, d=val) }}
  {% else %}
    <IValue value={{ ui_value(val) }} />
  {% endif %}
  ```
- **`catalog.render_first_existing([names...], **kwargs)`** — an
  OARepo-specific addition (not stock JinjaX), tries each name in order and
  only raises if none exist. **This is the per-type override mechanism for
  the Jinja layer** — the direct analog of the React-side `$schema`-keyed
  `DynamicResultsListItem` dispatch (see
  [search.md](search.md#2-multi-model-result-dispatch-dynamicresultslistitem)),
  but for server-rendered components:
  ```jinja
  {{ catalog.render_first_existing(
      ["VocabularyExtraInfo." + extra_context.vocabularyType, "EmptyComponent"],
      record=record, extra_context=extra_context, d=d
  ) }}
  ```
  To add type-specific rendering, create a component named
  `<Base>.<yourType>` (e.g. `VocabularyExtraInfo.awards`) rather than
  editing the generic one — the dotted suffix is looked up automatically.

### How component names resolve to files

`OarepoCatalog` (a subclass of JinjaX's own `Catalog`, wired up in
`oarepo_ui/ext.py`) scans the **entire** Jinja search path — across every
installed package, not just one — for `*.jinja` files. A file's dotted
component name is derived from its path relative to the templates root,
e.g. `oarepo_ui/templates/oarepo_ui/pages/RecordDetail.jinja` →
`oarepo_ui.pages.RecordDetail`, and a top-level
`oarepo_ui/templates/components/IdentifierBadge.jinja` → `IdentifierBadge`.
Two path-prefix conventions to know about:

- **A leading numeric `NNN-` prefix on a filename controls override
  priority** — used when a downstream package needs its version of a
  same-named component to win over an upstream one placed earlier in the
  search path. If two packages ship a same-named component and you can't
  tell which one is winning, check for this prefix before assuming it's a
  simple search-path-order question.
- **An `APP_THEME` prefix** is stripped the same way, for theme-specific
  variants.

## 2. The `I*` data-field family — the toolkit for record-detail content

`oarepo_ui/templates/components/datafields/` is the backbone of how a
record's metadata actually gets rendered into a detail page — this is what
you reach for inside a `record_detail/main.html` override (see
[wiring.md §6](wiring.md#6-the-templates-are-mostly-empty-extends-stubs))
rather than hand-writing `<dt>`/`<dd>` markup yourself:

| Component | Props | Purpose |
|---|---|---|
| `IField` | `d, label=None, placeholder=None` | `<dt>`/`<dd>` field row; renders supplied `content` children, or falls back to `<IValue>` |
| `IBaseField` | `label, label_class, data_class` | Low-level `<dt>`/`<dd>` pair `IField` is built on |
| `IValue` | `value, placeholder=""` | Prints a scalar, or a placeholder if null/empty |
| `IValueList` | — | `<dl>` wrapper around a group of `IField`s |
| `IArray` | `d` | Iterates an array field and renders each item as `IValue` inside `IValueList` |
| `INonEmpty` | `d` | Renders `content` only if the field actually has a value — a conditional-slot wrapper |
| `ISection` | `title=None, title_hidden=False` | Generic titled section wrapper around `content` |
| `ITable` / `ITableField` / `ITableSection` | — | Table-row equivalents of `IValueList`/`IField`/`ISection`, for nested tabular layouts |
| `ITableArrayValue` | `value, placeholder=""` | Comma-joins an array for a table cell |
| `SeparatedProperty` | `d, label=None, isLast=False` | Compact inline "Label: value•" row with a trailing separator |
| `ICustomFields` | `custom_fields_config, d` | Renders a record's custom-fields config, dispatching per field to a configured component (via `catalog.irender`) or `IValue` |
| `IURL` / `ISearchLink` | — | Lower-level link builders used by `SearchLink`/vocabulary display components |

All of these operate on a `FieldData`/`d`-wrapper object (the same
server-built field-metadata concept behind the React-side
`getFieldData()`/`ui_model` mechanism in
[forms.md §5](forms.md#5-schema-driven-behavior-labelshelprequired-only-not-widget-choice)
— labels/placeholders come from the same source on both the Jinja and
React sides). Compose a custom detail section from these primitives rather
than writing raw `<dt>`/`<dd>`/`<table>` markup by hand.

## 3. Other reusable components — by package

**Generic UI atoms** (`oarepo_ui/templates/components/`, beyond
`ClipboardCopyButton`/`IdentifierBadge`/`RecordSharing` already covered in
[reusable-widgets.md](reusable-widgets.md)):

| Component | Props | Purpose |
|---|---|---|
| `DefinitionLink` | `href, hasTooltip=True` | Book-icon link to a vocabulary term's definition page |
| `IdentifiersAndLinks` | `originalRecordUrl=None, objectIdentifiers=None, pids=None` | Sidebar "Identifiers and links" box; composes `IURL` + `ClipboardCopyButton` |
| `Multilingual` | `d` | Tabbed multi-language value display |
| `SearchLink` | `d, search_link, searchFacet, class_name=None` | Builds a faceted-search link from a vocabulary/value dict |
| `RecordExport` | `record, extra_context` | Sidebar "Export" box (mounts the React export-format widget) |
| `RecordVersions` | `record, is_preview=False` | Sidebar "Versions" box (mounts a React widget) |
| `files/FilesViewer` | `files` | The deposit-form files table (name/size/preview/download) — deposit context, distinct from the record-detail file list |

**Page-shell components** (`oarepo_ui/templates/oarepo_ui/pages/`) — the
full set behind the `templates` dict already shown in
[conventions.md](conventions.md): `RecordDetail`, `RecordSearch`,
`DepositCreate`, `DepositEdit`, `Tombstone`, `NotFound`. Each declares a
large typed `{# def #}` contract and immediately `{% extends model_name ~
"/<page>.html" %}` — this signature is effectively the documented contract
for what a `catalog.render()` call for that page must supply.

**Vocabulary display components** (`oarepo_vocabularies_ui`) — two
parallel families: plain-dict ones (`TaxonomyItem`/`TaxonomyArray`,
`VocabularyItem`/`VocabularyArray`) and `FieldData`-flavored equivalents
meant to be dropped into an `IField` `content=` slot from record-detail
context (`ITaxonomyItem`/`ITaxonomyArray`, `IVocabularyItem`/
`IVocabularyArray`). Plus the vocabulary admin/detail page's own component
set (`VocabulariesDetail`, `VocabulariesMain`, `VocabulariesList`,
`VocabulariesForm`, `VocabulariesSearch`, `VocabulariesSidebar`,
`VocabulariesBreadcrumb`, `SidebarLink`) — and `VocabularyExtraInfo.awards`
as the concrete real example of the `render_first_existing` per-type
override pattern from §1.

**What's thin or absent — don't go looking for these:**

- **`oarepo_rdm`** ships only plain (non-JinjaX) `.html` files:
  `new_upload_page.html` (model-picker cards) and
  `record_detail_iframe.html` (wraps a record in an `<iframe>`). No
  component library here.
- **`oarepo_requests` has no JinjaX components and no page-content
  templates at all.** Its `templates/` tree is entirely email/notification
  fragments plus thin page shells that `{% extends %}` base
  `invenio_requests` templates. **Request-type labels/badges are rendered
  entirely client-side in React** (see
  [requests.md](requests.md#what-does-exist-a-python-side-labeliconregistry)) —
  there is no Jinja-side equivalent to look for.
- **`ccmm_invenio` has zero `.jinja`/`.html` files anywhere in the
  package.** Its config/resource classes are empty subclasses of
  `oarepo_rdm`'s own; **all CCMM-specific UI is React**, and CCMM's record
  pages render entirely through the generic `oarepo_ui` components above.
  If you're customizing a CCMM-based model's detail page at the Jinja
  layer, you're writing that customization in your own repository (as
  `datarepo/ui/datasets/templates/.../record_detail/main.html` does — see
  [wiring.md §6](wiring.md#6-the-templates-are-mostly-empty-extends-stubs)),
  not looking for it inside `ccmm_invenio` itself.

## 4. Plain Jinja2 macros — the minority pattern

Most reusable Jinja content in this ecosystem is a JinjaX component, not a
`{% macro %}`. Macros are used specifically for two narrower cases: a
simple, mostly-static content block, and grouping several small formatting
helpers that get called repeatedly within one larger partial.

```jinja
{# oarepo_ui/templates/oarepo_ui/macros/records_list.html #}
{% macro records_list(title=_("Recent uploads"), fetch_url="/api/records?sort=newest&size=10") %}
  <div id="records-list" data-fetch-url="{{ fetch_url }}" data-title="{{ title }}"></div>
{% endmacro %}
```
```jinja
{# calling template #}
{% from "oarepo_ui/macros/records_list.html" import records_list %}
{{ records_list() }}
```

Base `invenio_app_rdm` ships the richer, real-world examples this
ecosystem's record-detail rendering actually depends on:
`invenio_app_rdm/records/macros/creatibutors.html` (`show_creatibutors`,
`affiliations_accordion` — the creators/contributors renderer) and
`invenio_app_rdm/records/macros/detail.html` (`show_dates`,
`list_languages`, `show_alternate_identifiers`, `show_funding`,
`show_references`, `show_section_custom_fields`, ...) — both `{% include %}`d
from `oarepo_ui/templates/oarepo_ui/record_detail/main.html`'s default
blocks.

**`with context` is required** when a macro needs to see globals injected
into the Jinja context (`_`, `ui_value`, `value`, `as_array`) rather than
receiving them as explicit arguments — every real macro import in this
codebase uses it:

```jinja
{%- from "datasets/macros/related_resources.html" import related_resources_section with context -%}
```

If you write your own macro file (the real, repository-local example is
`datarepo/ui/datasets/templates/.../macros/related_resources.html`, which
defines `format_creators`/`related_resource_citation`/
`related_resources_section` and is imported exactly this way from that
model's `record_detail/main.html` override), always import it `with
context` unless every value the macro needs is passed as an explicit
argument.

## 5. Static/informational pages

A static page (not tied to a record model) is wired via
`TemplatePageUIResource`/`TemplatePageUIResourceConfig` (Python side, fully
covered in
[wiring.md §7](wiring.md#7-wiring-a-plain-non-model-page-a-smaller-worked-example)).
The `.jinja` template itself needs **no `{# def #}` block at all** — it's
rendered via `catalog.render("PageName", ...)` exactly like any other
component, but since it's really just an ordinary block-overriding Jinja
page template, no typed signature is required (unused/extra kwargs are
silently ignored the same way `EmptyComponent` ignores them, §1).

**Base template chain**: `"page.html"` (or the indirected
`config.BASE_TEMPLATE`, used by e.g. `oarepo_vocabularies_ui`'s own pages)
→ `invenio_app_rdm/page.html` (adds the RDM footer) →
`invenio_theme/page.html` (the actual `<html>`/`<head>`/`<body>` skeleton,
defining blocks `head`, `head_meta`, `head_title`, `head_links`, `header`,
`css`, `body`, `page_header`, `page_body`, `page_footer`, `javascript`,
`trackingcode`). `oarepo_ui/templates/oarepo_ui/base_page.html` is an
alternative base adding Matomo analytics inclusion and embedded-mode
header/footer suppression.

**The standard recipe** (real example, `GetAccess.jinja`):

```jinja
{% extends "page.html" %}
{% block css %}{{ super() }}<style>...</style>{% endblock css %}
{% block javascript %}{{ super() }}{{ webpack['get_access.js'] }}{% endblock javascript %}
{% block page_body %}
  <div class="ui main container rel-mt-3">
    <h1>{{ _("Page title") }}</h1>
    ...
    <div id="my_react_island"></div>
  </div>
{% endblock %}
```

Override `css`/`javascript` with `{{ super() }}` plus your webpack entry,
override `page_body` with hand-written Semantic UI markup, and optionally
mount a React island the same way record pages do (a plain `<div id="...">`,
hydrated by the webpack entry — the JinjaX-to-React bridge is identical
here, just written directly instead of via a `data-invenio-search-config`
attribute). **In practice, static pages are usually hand-written HTML in
`page_body`, not composed from the `IField`/`IValue` component family** —
that family is built around record/`FieldData` objects a static page
doesn't have. Reuse the generic atoms from §3 (`ClipboardCopyButton`,
`IURL`, macros like `records_list`) where relevant instead.
