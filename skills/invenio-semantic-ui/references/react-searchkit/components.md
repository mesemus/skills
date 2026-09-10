# react-searchkit — UI components reference

Read this when you need the exact props/behavior of a specific built-in
component, or when deciding how to wire a custom component to the store the
same way the built-ins do.

## Connection pattern

Every component below follows the same two-layer shape: an outer component
`connect()`-ed to Redux (`mapStateToProps`/`mapDispatchToProps`, reading only
the exact fields it needs) wraps an inner presentational component, often
gated by `<ShouldRender condition={...}>`, which renders an
`Overridable`-wrapped `semantic-ui-react` primitive. When writing a new
component that needs *broad* read access to state instead of a couple of
specific fields, prefer the `withState(Component)` HOC instead of hand-rolling
a wide `connect()` — it injects:

```js
currentQueryState   // = state.query
currentResultsState // = state.results
updateQueryState    // dispatches SET_QUERY_STATE, then executeQuery()
```

`RangeFacet` (see [aggregations.md](aggregations.md#rangefacet)) and the
custom exclusive-filter pattern in
[../search.md §5](../search.md#5-aggregations--facets) both use `withState`
for exactly this reason.

## `SearchBar`

Props: `queryString` (connected, from `state.query.queryString`),
`updateQueryString` (connected), `actionProps`, `autofocus`, `placeholder`,
`uiProps`, `overridableId`, plus override hooks `executeSearch`/
`onInputChange`/`onKeyPress`/`onBtnSearchClick`.

- **Uncontrolled with a `key={queryString}` remount trick** — rather than
  keeping the input value in sync via props on every keystroke, the
  component remounts itself whenever the *external* `queryString` changes
  (e.g. from a URL back/forward navigation or a "Clear query" action),
  avoiding a whole class of derived-state bugs.
- Renders a `semantic-ui-react` `Input` with a "Search" action button; Enter
  also submits. **No debounce** — it only searches on Enter/button click.
- `elementId` prop: if a DOM element with that id already exists on the
  page, the search bar **portals** into it (`ReactDOM.createPortal`) instead
  of rendering inline. Useful for putting the search box in a page header
  outside the React root; easy to misdiagnose as "the search bar
  disappeared" if you don't know this exists.

## `AutocompleteSearchBar`

The variant with real debouncing, used together with `InvenioSuggestionApi`
(see [api.md](api.md#5-suggestions--autocomplete)). Debouncing is
conditional on a `debounce` prop — when set, `updateSuggestions` is wrapped
in `_.debounce(fn, debounceTime, { leading: true })`. Note this only
debounces the *suggestions* request; the main search itself is still only
triggered by `executeSearch()` (Enter/click), matching plain `SearchBar`.

## `Sort` / `SortBy` / `SortOrder`

```js
Sort.propTypes = {
  values: PropTypes.arrayOf(
    PropTypes.shape({ text: PropTypes.string, sortBy: PropTypes.string, sortOrder: PropTypes.string })
  ).isRequired,
};
```

`Sort` computes `option.value = sortOrder ? `${sortBy}-${sortOrder}` : sortBy`
and renders a single `Dropdown`. `SortBy`/`SortOrder` are separate
components for independent by/order dropdowns (`values` shape there is
`{text, value}`). All three:

- Only render when `!loading && totalResults > 0` (via `ShouldRender`) — a
  component that appears to not be "wired up" before the first search
  completes is usually just correctly hidden.
- Accept a `label` render-prop (default identity) to wrap the dropdown:
  `label={(cmp) => <>Sort by: {cmp}</>}`.

## `Pagination`

```js
Pagination.defaultProps = {
  options: { boundaryRangeCount, siblingRangeCount, showEllipsis, showFirst, showLast, showPrev, showNext, size, maxTotalResults: 10000 },
  showWhenOnlyOnePage: false,
};
```

Connected: `currentPage`, `currentSize`, `loading`, `totalResults`,
`updateQueryPage`. Wraps `semantic-ui-react`'s own `Pagination`.
`maxTotalResults` caps how many pages are ever shown — a guard against
Elasticsearch's default 10k-result deep-pagination window; raising it
without also raising the backend's window just produces pages that 400/500.

## `ResultsPerPage`

`values: {text, value}[]`; connected `currentSize`, `updateQuerySize`;
renders a `Dropdown`.

## `Count`

No required own props besides `label`/`overridableId`; connected `loading`,
`totalResults`; renders `totalResults.toLocaleString("en-US")` in a `Label`.

## `ResultsList` / `ResultsGrid` / `ResultsMultiLayout`

Connected: `loading`, `totalResults`, `results` (= `state.results.data.hits`).

**There is no render-prop for per-hit rendering.** Each hit is rendered by
an internal `ListItem`/`GridItem` wrapped in
`<Overridable id={buildUID("ResultsList.item", overridableId)} result={result}>`.
To customize hit rendering, register an override for `"ResultsList.item"` /
`"ResultsGrid.item"` — see
[../search.md §6](../search.md#6-customizing-the-result-item) for the
Invenio-specific override-registration pattern and the conventional
`result.metadata`/`result.ui`/`result.links`/`result.expanded` shape.

`ResultsMultiLayout` composes both `ResultsList`/`ResultsGrid` behind a
`currentLayout` (`"list"`/`"grid"`) switch, driven by `LayoutSwitcher`.

## `ResultsLoader`

`children` (required node), shown when `!loading`; shows a `semantic-ui-react`
`Loader` (spinner) when `loading`. Used as a wrapper:
`<ResultsLoader><ResultsList/></ResultsLoader>`.

## `Error`

Connected `error` (`state.results.error`); shows only when
`!loading && !isEmpty(error)`.

## `EmptyResults`

Connected `loading`, `totalResults`, `error`, `queryString`, `resetQuery`,
`userSelectionFilters`; shows only when
`!loading && isEmpty(error) && totalResults === 0`. Ships a "Clear query"
button wired to `resetQuery()`, which dispatches `RESET_QUERY` (resets
`queryString`, `page`, `filters`).

## `ActiveFilters`

Connected `filters` (`state.query.filters`), `updateQueryFilters`. Renders
one removable `Label` per active filter — clicking a label removes it by
re-dispatching the same filter (filters are toggled, not added/removed
independently — see
[aggregations.md](aggregations.md#filter-toggle-semantics)). Supports a
nested/child filter's label text via `value.childLabel`.

## Full export surface

`ActiveFilters, AppContext, AutocompleteSearchBar, BucketAggregation, Count,
EmptyResults, Error, InvenioRecordsResourcesRequestSerializer,
InvenioRequestSerializer, InvenioResponseSerializer, InvenioSearchApi,
InvenioSuggestionApi, LayoutSwitcher, OSRequestSerializer, OSResponseSerializer,
OSSearchApi, Pagination, RangeFacet, ReactSearchKit, RequestCancelledError,
ResultsGrid, ResultsList, ResultsLoader, ResultsMultiLayout, ResultsPerPage,
SearchBar, Sort, SortBy, SortOrder, Toggle, UrlHandlerApi, UrlParamValidator,
buildUID, createStoreWithConfig, onQueryChanged, withState`.

Note there is **no exported `createSearchAppInit`** in this package — that
helper (and the whole Invenio entrypoint convention) lives in
`invenio_search_ui`; see
[../search.md §2](../search.md#2-entrypoint-pattern-used-across-every-invenio-package).
