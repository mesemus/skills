# react-searchkit — aggregations / facets in depth

Read this before building any facet beyond a plain checkbox list, before
building a nested/hierarchical facet, or when a facet's selected state isn't
toggling the way you expect.

## `BucketAggregation`

```js
BucketAggregation.propTypes = {
  title: PropTypes.string.isRequired,
  agg: PropTypes.shape({
    field: PropTypes.string.isRequired,     // ES field the agg is on (mostly informational to the UI)
    aggName: PropTypes.string.isRequired,   // key used to look up resultsAggregations[aggName]
    childAgg: PropTypes.object,             // nested facet config, same shape, for hierarchical filters
  }).isRequired,
};
```

Connected: `userSelectionFilters` (= `state.query.filters`),
`resultsAggregations` (= `state.results.data.aggregations`),
`updateQueryFilters`. It reads `resultsAggregations[agg.aggName].buckets`,
**normalizing both array and object-keyed bucket shapes** (Invenio's
Elasticsearch/OpenSearch responses aren't always consistent about which one
comes back — `Object.entries(...)` handles the object case), filters
`userSelectionFilters` down to entries whose `filter[0] === agg.aggName`, and
passes both to `BucketAggregationValues`.

## Nested / hierarchical facets (`childAgg`)

`BucketAggregationValues` recurses for `agg.childAgg`: for each bucket it
checks `bucket[childAgg.aggName].buckets` and, if present, renders another
`BucketAggregationValues` for the nested level, with a scoped
`onFilterClicked` that appends a third element:

```
["type", "publication"]                          // simple selection
["type", "publication", ["subtype", "report"]]   // parent selected, specific child selected
```

## Filter data model

A filter is always an **array**, never a bare string:

```js
["file_type", "pdf"]                        // simple
["type", "publication", ["subtype", "report"]]  // nested parent+child
```

`InvenioSearchApi`'s request serializer explicitly throws
(`Filter value "..." in query state must be an array`) if a plain string is
pushed instead — see [api.md](api.md).

## Filter toggle semantics

```js
updateQueryFilters(filter) // dispatches SET_QUERY_FILTERS
```

The reducer delegates to `updateFilter()` in `state/selectors/query.js`,
which **toggles** the filter based on string-serialization matching: if the
exact filter (or its parent/child relationship) is already present, it's
removed; otherwise it's added. This includes special handling to drop child
filters when their parent is toggled off, and to re-add "parent without
child" when only the specific child selection is removed.

**Consequence for custom facet UI**: `BucketAggregationValues`'s checkbox
`onClick` always calls the *same* `onFilterClicked(bucket.key)` regardless
of the checkbox's current checked state — add-vs-remove is entirely decided
by the reducer/selector, not by the click handler. Any custom facet
component should do the same: dispatch the filter unconditionally on click
and let `updateQueryFilters` figure out whether that's an add or a remove.

## Extension points beyond checkboxes

Override one of these Overridable ids (via the mechanism in
[conventions.md](../conventions.md#1-overridable-components--the-central-customization-mechanism)),
matching the same `agg`/`aggName`/`childAgg` config contract:

- `"BucketAggregationValues.element"` — one checkbox row. Receives `bucket`,
  `label`, `onFilterClicked`, `isSelected`, `childAggCmps`.
- `"BucketAggregationContainer.element"` — the `<List>` wrapper. Receives
  `valuesCmp`.
- `"BucketAggregation.element"` — the whole facet block. Receives `agg`,
  `title`, `containerCmp`, `updateQueryFilters`.

Invenio's own packages layer polished, reusable versions of these on top
(card wrapper with a "Clear" button, an accordion-based nested-facet
renderer, etc.) — see
[../search.md §5](../search.md#5-aggregations--facets) for the concrete
`Contrib*` components used across `invenio_search_ui`/`invenio_communities`/
`invenio_requests`/`invenio_administration`, which are the components you'll
actually compose day-to-day rather than the bare `BucketAggregation`.

## `RangeFacet`

A date/numeric-range facet (`@visx/*`-based histogram + slider), built on
`withState` rather than `connect()` — it reads
`currentQueryState.filters`/`currentResultsState.data.aggregations`
directly. Unlike the bucket-array model, it encodes its selected range as a
**single string value**, `` `${from}${rangeSeparator}${to}` `` (e.g. with
`rangeSeparator="--"`), passed as a required `rangeSeparator` prop, plus an
optional `defaultRanges` list and a custom range input UI. Use this (or
route `agg.type === "date"` to it, matching `invenio_search_ui`'s
convention) for date/numeric facets instead of `BucketAggregation`.

## `Toggle`

A simple boolean facet — clicking it toggles one fixed filter value
(e.g. `filterValue={["allversions", "true"]}`) rather than offering multiple
bucket choices. Use for "show all / show only X" switches.

## Building a fully custom, non-bucket facet

When the desired UX is genuinely different from a checkbox/value list (tabs,
segmented buttons, exclusive single-select), skip `BucketAggregation`
entirely and build a `withState`-wrapped component that reads/writes
`currentQueryState.filters` directly:

```jsx
const FilterButtons = withState(({ currentQueryState, updateQueryState }) => {
  const setStatus = (value) => {
    const filters = currentQueryState.filters.filter((f) => f[0] !== "status");
    filters.push(["status", value]);
    updateQueryState({ ...currentQueryState, filters });
  };
  return (
    <Button.Group>
      <Button onClick={() => setStatus("open")}>Open</Button>
      <Button onClick={() => setStatus("closed")}>Closed</Button>
    </Button.Group>
  );
});
```

This is the real pattern behind e.g. an admin "deletion status" filter and a
requests "open/closed" filter (both hand-rolled `Button.Group`s, not
`BucketAggregation`s) — reach for it whenever a facet needs exclusive/tab-like
selection instead of independent checkboxes.
