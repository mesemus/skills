# react-searchkit — consolidated gotchas

A quick-scan list of non-obvious behaviors, each cross-referenced to the
file with full detail. Read the relevant section before writing a custom
component in that area — these are the mistakes that are easy to make once
and then hard to diagnose.

- **Filters must be arrays, never strings.** `["aggName", "value"]`, or a
  3-element nested form. The default request serializer throws if you push a
  plain string. → [aggregations.md](aggregations.md#filter-data-model)

- **Checkbox/facet clicks are idempotent, not add/remove.** The same
  `updateQueryFilters(filter)` call fires regardless of current selection
  state; the reducer/selector decides add-vs-remove by diffing. Custom facet
  components must dispatch unconditionally on click, matching this. →
  [aggregations.md](aggregations.md#filter-toggle-semantics)

- **Aggregation buckets can be an array OR an object keyed by bucket key.**
  `BucketAggregation` normalizes both; a hand-rolled facet renderer that
  bypasses it must do the same. → [aggregations.md](aggregations.md)

- **`SearchBar` has no debounce and only searches on Enter/click.** Only
  `AutocompleteSearchBar`'s *suggestions* request is debounced (via a
  `debounce` prop); the main query is never live-searched without you adding
  your own debounce on top of `updateQueryString`. →
  [components.md](components.md#searchbar)

- **Request de-duplication happens via cancellation, not time-based
  debouncing.** Every `InvenioSearchApi.search()` call cancels the previous
  in-flight request; rapid dispatches are safe from race conditions but each
  one still hits the network. `RequestCancelledError` is deliberately
  swallowed by `executeQuery` rather than shown as an error — a custom
  `searchApi` must reject with the same error class for cancellations or
  you'll get spurious error flashes. → [api.md](api.md)

- **`Sort`/`SortBy`/`SortOrder`/`ResultsPerPage`/`Pagination`/`Count` all
  render `null`** while `loading` or when `totalResults === 0`. This is
  intentional hiding, not a sign the component failed to wire up. →
  [components.md](components.md)

- **`SearchBar`'s `elementId` prop portals it elsewhere in the DOM** via
  `ReactDOM.createPortal` — if a search bar seems to have vanished from
  where you rendered it, check for this prop first. →
  [components.md](components.md#searchbar)

- **`Pagination`'s `maxTotalResults` (default 10000) caps shown pages** to
  guard against Elasticsearch's default 10k deep-pagination window — raising
  it without also raising the backend's window just produces pages that
  error. → [components.md](components.md#pagination)

- **Two bundled copies of `react-searchkit` break `AppContext` identity**,
  silently making `buildUID`/`nextComponentIndex` return `undefined`. A
  classic monorepo/vendoring footgun. →
  [architecture.md](architecture.md#7-known-footgun-duplicate-library-copies-break-appcontext)

- **Under multi-instance pages, every `appName` must be distinct**, and
  every Overridable id registered for that instance must be manually
  prefixed with it — see the Invenio-specific `multi` flag wiring in
  [../search.md §3](../search.md#3-appname--multi--required-for-more-than-one-search-app-per-page).

- **There is no `createSearchAppInit`, and no `ResultsList`/`ResultsGrid`
  render-prop for per-hit rendering** exported by `react-searchkit` itself —
  both of those are Invenio conventions layered on top
  (`invenio_search_ui`'s bootstrap helper, and the `"ResultsList.item"`
  Overridable id respectively). Don't look for them inside the library's own
  exports. → [components.md](components.md#resultslist--resultsgrid--resultsmultilayout)
