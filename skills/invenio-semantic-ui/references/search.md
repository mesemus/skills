# Invenio search/listing pages — deep reference (react-searchkit)

Read this file when building or editing a search results page, a listing
page with facets/aggregations, or any component that consumes search state.
This file covers how Invenio *uses* `react-searchkit` (entrypoints,
multi-app pages, result-item conventions). For the library's own internals
in full detail — Redux store shape, exact component props, the aggregation
filter model, URL sync, and a consolidated gotchas list — see
`references/react-searchkit/`:

- [react-searchkit/architecture.md](react-searchkit/architecture.md) — the
  Redux store, `AppContext`/`buildUID`, render tree, multi-instance mechanics.
- [react-searchkit/api.md](react-searchkit/api.md) — the `searchApi`
  contract, `InvenioSearchApi`, request/response shape, cancellation.
- [react-searchkit/components.md](react-searchkit/components.md) — full
  prop reference for every built-in component (`SearchBar`, `Sort`,
  `Pagination`, `ResultsList`, ...).
- [react-searchkit/aggregations.md](react-searchkit/aggregations.md) — the
  facet/filter data model, nested facets, `RangeFacet`/`Toggle`, building a
  fully custom facet.
- [react-searchkit/url-state.md](react-searchkit/url-state.md) — URL
  parameter mapping and browser history sync.
- [react-searchkit/gotchas.md](react-searchkit/gotchas.md) — consolidated,
  cross-referenced gotchas for the library itself.

Checkbox row-selection plus a "do X to all selected results" toolbar is a
separate, substantial extension on top of this — see
[bulk-actions.md](bulk-actions.md) rather than building it from scratch.
The responsive facets sidebar (`GridResponsiveSidebarColumn`) used by every
search layout below is documented in
[react-invenio-forms/ui-primitives.md](react-invenio-forms/ui-primitives.md).

## 1. Architecture in one paragraph

`react-searchkit` is a self-contained Redux app. `<ReactSearchKit>` builds a
store (`{app, query, results}` reducers, `redux-thunk` with the `searchApi`
config injected as the thunk extra argument) and provides it via both a
`redux` `Provider` and a plain React `AppContext` (used only for
`buildUID`/`appName`, not for query state). Every built-in UI component
(`SearchBar`, `Sort`, `ResultsList`, `BucketAggregation`, ...) is
`connect()`-ed to this store — you compose them declaratively; you never
dispatch actions directly except when building a fully custom facet (§5).

```jsx
<ReactSearchKit searchApi={{ axios: { url: "/api/records" }, invenio: {} }}>
  {/* SearchBar, Sort, ResultsList, BucketAggregation, Pagination, ... */}
</ReactSearchKit>
```

`InvenioSearchApi` (the default `searchApi` implementation) expects an
Elasticsearch/OpenSearch-shaped JSON response: `{ hits: { hits: [...], total
}, aggregations: {...} }`. If you point this at a custom backend, match that
response shape or provide your own `SearchApi`/response serializer — full
contract in [react-searchkit/api.md](react-searchkit/api.md), and the
Redux/render-tree mechanics behind `<ReactSearchKit>` in
[react-searchkit/architecture.md](react-searchkit/architecture.md).

## 2. Entrypoint pattern used across every Invenio package

Search pages are bootstrapped by `createSearchAppInit` (from
`invenio_search_ui`), which finds a DOM element carrying a
`data-invenio-search-config` JSON attribute (rendered by Jinja from a Python
`SearchAppConfig`), merges a `defaultComponents` override map with any
globally-registered overrides, and mounts `<SearchApp>`:

```jsx
createSearchAppInit(
  defaultComponents,           // { "SearchApp.layout": MyLayout, "ResultsList.item": MyItem, ... }
  true,                        // autoInit
  "invenio-search-config",     // data-attribute name
  multi,                       // see §3
  ContainerComponent           // optional extra context wrapper, defaults to React.Fragment
);
```

`SearchApp` itself composes: `OverridableContext.Provider` →
`SearchConfigurationContext.Provider` → `<ReactSearchKit>` → an
`Overridable id="SearchApp.layout"` wrapping a responsive Grid: search bar
row, a `Sort`/mobile-filter-toggle row, then a two-column row
(`GridResponsiveSidebarColumn` for facets + a results column). If
`config.aggs` is empty, the facets sidebar is omitted entirely and results
take the full width.

## 3. `appName` / `multi` — required for more than one search app per page

`buildUID(elementName, overridableId, appName)` produces the Overridable id
every internal component looks itself up by:
`${appName ? appName + "." : ""}${elementName}${overridableId ? "." + overridableId : ""}`.

- **`multi=false`** (the common case: one search app per page — the generic
  `invenio_search_ui` page, an admin resource list): override keys are bare,
  e.g. `"SearchApp.layout"`, `"ResultsList.item"`.
- **`multi=true`** (two or more `ReactSearchKit` trees can exist on the same
  page — e.g. a community's own search vs. a sub-community's — the `appId`
  from the JSON config is used as `appName`): **every override key must be
  manually prefixed** with `${appId}.`, e.g.
  `"InvenioCommunities.Search.SearchApp.layout"`. Getting this prefix wrong
  is the most common real-world mistake — an unprefixed key under `multi=true`
  silently never matches.

Each `ReactSearchKit` instance owns its own isolated store — state is never
shared between instances; `appName` only affects override-id namespacing and
DOM-id uniqueness for facet checkboxes, and lets a custom
`window`-level `queryChanged` event target one specific instance.

## 4. Core components

`SearchBar`, `Sort`/`SortBy`/`SortOrder`, `Pagination`, `ResultsPerPage`,
`Count`, `ResultsList`/`ResultsGrid`/`ResultsMultiLayout`, `ResultsLoader`,
`Error`, `EmptyResults`, `ActiveFilters` are composed declaratively inside
`<ReactSearchKit>` — full prop tables and behavior notes (hidden-until-loaded
semantics, the `SearchBar` portal prop, `Pagination`'s result-window cap,
etc.) are in [react-searchkit/components.md](react-searchkit/components.md).

## 5. Aggregations / facets

**Config shape** (Python side, flows into `config.aggs` in the JSON blob):

```python
type = TermsFacet(
    field="metadata.type.id",
    label=_("Type"),                                   # lazy_gettext — i18n happens server-side
    value_labels={"organization": _("Organization"), "event": _("Event")},
)
MY_FACETS = {"type": {"facet": type, "ui": {"field": "type"}}}
```

This becomes a JS `agg` object: `{aggName: "type", field: "type", title: "Type", childAgg?: {...}}`.
**Bucket/value labels are translated entirely server-side** — the React
layer just renders whatever localized string is already in `agg.title`/
`bucket.label`; don't re-translate facet labels client-side.

`<BucketAggregation title={agg.title} agg={agg} />` renders the standard
checkbox list, including nested `childAgg` facets. The full filter data
model (always an array, toggle-not-add/remove semantics), the extension
points for a different-looking facet, `RangeFacet`, `Toggle`, and the
hand-rolled `withState`-based pattern for exclusive/tab-like filters (used
for e.g. an admin "deletion status" filter or a requests "open/closed"
filter) are all in
[react-searchkit/aggregations.md](react-searchkit/aggregations.md) — read
that before building anything beyond a plain checkbox facet.

## 6. Customizing the result item

Register a component for `"ResultsList.item"` (and `"ResultsGrid.item"` if a
grid view exists) in the entrypoint's `defaultComponents` map (remember the
`appName.` prefix under `multi=true`, §3). Pull fields off `result`, being
aware of the recurring three-tier shape:

- `result.metadata.*` — raw record data.
- `result.ui.*` — server-computed, permission/label-friendly presentation
  fields (e.g. `result.ui.type.title_l10n`).
- `result.links.*` — hypermedia links (`self_html`, `logo`, ...).
- `result.expanded.*` — server-side "expand" of referenced entities (e.g. a
  request's creator) to avoid client-side N+1 lookups; check this before
  falling back to a raw reference id.

```jsx
export const MyResultItem = ({ result }) => (
  <Item href={result.links.self_html}>
    <Item.Content>
      <Item.Header>{result.metadata.title}</Item.Header>
      <Item.Extra>{result.ui?.type?.title_l10n}</Item.Extra>
    </Item.Content>
  </Item>
);
```

For a table-style listing (e.g. an admin resource list) instead of a
card/list, override `"ResultsList.item"` with a `<Table.Row>` per hit and
`"ResultsList.container"` with the `<Table>`/`<Table.Header>` wrapper, and
drive cell rendering from a column/schema config rather than hardcoded field
names — see `invenio_administration`'s `SearchResultItem`/`Formatter` for the
reference implementation of this variant.

## 7. URL state sync

Enabled by default (no router dependency) — bookmarkable search state is
automatic as long as you use the provided action creators
(`updateQueryFilters`, `updateQueryString`, `updateQueryPage`, ...) rather
than mutating state ad hoc. Parameter naming, filter serialization format,
and browser back/forward handling are detailed in
[react-searchkit/url-state.md](react-searchkit/url-state.md).

## 8. Gotchas checklist

The two mistakes specific to how *Invenio* composes `react-searchkit`:

- Under `multi=true`, every override map key needs the `${appName}.` prefix;
  under `multi=false`, it must NOT have one (§3).
- Not every package with `TermsFacet` definitions has its own search page —
  some (e.g. vocabularies) only contribute facet *config* that another
  package's generic search app (e.g. administration) renders. Check for an
  actual `createSearchAppInit`/`<ReactSearchKit>` call before assuming a
  dedicated frontend exists.

For the library's own gotchas (filter shape, toggle semantics, hidden-until-
loaded components, request cancellation vs. debouncing, duplicate-copy
`AppContext` breakage, and more), see the consolidated, cross-referenced
list in [react-searchkit/gotchas.md](react-searchkit/gotchas.md).
