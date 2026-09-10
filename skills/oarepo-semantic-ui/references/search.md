# OARepo search pages — `createSearchAppsInit` and multi-model dispatch

Read this file when writing or editing an OARepo search/listing page or a
custom facet. Assumes you already know react-searchkit's core components
and the `data-invenio-search-config` bootstrap convention (sibling
`invenio-semantic-ui` skill) — this file covers only the OARepo layer on
top. For the Python entry-point registration and a real search entry point
with real `parametrize`-based overrides, see [wiring.md](wiring.md).

## 1. Same Python config, different JS bootstrap

The Python side is largely **unchanged**: `RecordsUIResourceConfig.search_app_config()`
delegates straight to base `invenio_search_ui`'s own
`SearchAppConfig`/`FacetsConfig`/`SortConfig.generate(...)`, and the
rendered page still emits the identical
`<div data-invenio-search-config='...'>` DOM contract. What's OARepo-specific
is the **JS bootstrap function**:

```js
import { createSearchAppsInit, parseSearchAppConfigs, SearchAppLayout } from "@js/oarepo_ui/search";

const [{ overridableIdPrefix }] = parseSearchAppConfigs();
export const componentOverrides = {
  [`${overridableIdPrefix}.ResultsList.item`]: ResultsListItemWithConfig,
  [`${overridableIdPrefix}.SearchApp.layout`]: SearchAppLayoutWithConfig,
};
createSearchAppsInit({ componentOverrides });
```

`createSearchAppsInit` (**plural** — not base Invenio's singular
`createSearchAppInit`):

- Parses **all** `[data-invenio-search-config]` elements on the page (via
  its own `parseSearchAppConfigs()`), supporting multiple independent
  search apps per page out of the box.
- Reads `overridableIdPrefix` out of each config (set server-side from the
  model's `application_id`, e.g. `"Datarepo.Search"`) instead of relying on
  a manually-passed `appId`/`appName` — see
  [conventions.md](conventions.md#the-overridableidprefixapplication_id-namespacing-convention).
- Pre-registers a **larger default component-slot table** than base
  Invenio: `ActiveFilters.element`, `BucketAggregation.element`,
  `BucketAggregationValues.element`, `Count.element`, `EmptyResults.element`,
  `Error.element`, `SearchApp.facets`, `SearchApp.layout`,
  `SearchApp.resultOptions`, `ResultsPerPage.element`,
  `SearchApp.searchbarContainer`, `SearchFilters.Toggle.element`,
  `Sort.element`, `SearchApp.results`, `SearchBar.element`,
  `ResultsList.container` — all pointing at OARepo's own reworked
  components (§3) by default, merged with your `componentOverrides` and
  anything already in `overrideStore`.
- Still ultimately renders base Invenio's own `<SearchApp>` from
  `@js/invenio_search_ui/components` underneath — the customization is in
  *which components fill the slots*, not a replacement search engine.

## 2. Multi-model result dispatch: `DynamicResultsListItem`

Base RDM only ever has one record shape, so it only needs one
`"ResultsList.item"` override. OARepo repositories can host many different
record models on one aggregated search page, so it needs a **dispatch
layer**:

```jsx
export const DynamicResultsListItem = ({ result, selector = "$schema", FallbackComponent, appName }) => {
  const selectorValue = _get(result, selector);
  if (!selectorValue) return <FallbackComponent result={result} />;
  return (
    <Overridable id={buildUID("ResultsList.item", selectorValue)} result={result}>
      <FallbackComponent result={result} />
    </Overridable>
  );
};
```

It reads a field off each hit (default `$schema`) and builds an
`Overridable` id from *that value*, not from a fixed string. **To add a
result-item renderer for a new model**, register an override keyed by that
model's `$schema` value — `buildUID("ResultsList.item", selectorValue)` —
rather than the plain `"ResultsList.item"` id base Invenio search pages use.
If the field is missing on a hit, it falls back to a generic
`FallbackItemComponent` and logs a console warning — don't be surprised by
that warning on malformed/legacy records.

**Note the ID-scheme overload**: `"ResultsList.item"` is used both as a
plain per-app default (`buildUID("ResultsList.item", "", appName)`, in the
standalone `RecordsList` widget, §3) and as this multi-model dispatch key
(`buildUID("ResultsList.item", selectorValue)`) — same convention, second
segment means something different depending on which component built the
ID. Read the call site before assuming which one you're overriding.

## 3. Reworked layout, facets, and results

`oarepo_ui/search/` substantially reworks react-searchkit's default layout
rather than using it as-is:

- **`SearchAppLayout`** — full layout replacement: responsive
  `GridResponsiveSidebarColumn` facet sidebar, a mobile facet-toggle button
  with an `ActiveFiltersCountFloatingLabel`, an optional third grid column
  (`SearchApp.buttonSidebarContainer` slot — used e.g. by
  `oarepo_vocabularies_ui` for a "New item" button), a `searchBarTip` slot,
  and a floating scroll-to-top button.
- **`SearchAppFacets`** — wraps each `BucketAggregation` in its own
  `react-error-boundary` (`SearchAppFacetsFallback`), so one malformed
  facet config can't take down the whole facet panel. Also adds a built-in
  "all versions" `Toggle`.
- **`FoldableBucketAggregationElement`/`BucketAggregationElement`** —
  collapsible (accordion) facet wrapper, not present in base react-searchkit.
  Supports per-aggregation dynamic override ids
  (`` buildUID("BucketAggregation.element", agg.aggName) ``) so a single
  named facet can be overridden without replacing the whole facets panel.
- **`histogram/` (`Histogram`, `HistogramWSlider`, `Slider`)** — a
  D3-based date-histogram facet with a range slider and click-to-filter
  bars. Use this for date-range faceting instead of react-searchkit's
  `RangeFacet` if you want the histogram visualization.
- **`ActiveFilters`** — richer than base: groups filters by aggregation,
  supports nested/child filters, and resolves labels via
  `additionalFilterLabels` (server-supplied through `search_app_config`'s
  `overrides`) so a filter can show a human label even if it's not present
  in the current response's aggregations.
- **`ClearableSearchbarElement`/`MultilineSearchbarElement`** — search bar
  variants (clear button; multi-line/textarea query input) replacing
  react-searchkit's plain `SearchBar`.
- **`ListItemContainer`** (in `ResultsList.jsx`) — wraps every result item
  in its own error boundary (`DefaultListItemErrorFallback`), registered by
  default as `${overridableIdPrefix}.ResultsList.container`, so one broken
  hit can't crash the whole results list.
- **`RecordsList`** — a standalone class component (e.g. for a "recent
  uploads" widget) that does its own plain `fetch`-based listing,
  independent of react-searchkit's Redux state machine entirely, reusing
  `DynamicResultsListItem` for per-hit rendering. Reach for this instead of
  a full `<ReactSearchKit>` tree when you just need a simple, non-faceted,
  non-paginated (or self-paginated) list.

Reuse these instead of react-searchkit's bare defaults when building a new
OARepo search page — they already add the responsive layout and error
isolation base react-searchkit doesn't have.
