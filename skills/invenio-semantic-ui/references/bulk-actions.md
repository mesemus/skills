# Bulk/multi-select actions on search results

Read this when a search/listing page needs checkbox row-selection plus a
"do X to all selected rows" action (bulk delete, bulk approve, bulk role
change, ...). Builds on [search.md](search.md) and `references/react-searchkit/`
— read those first for how the base search app is composed:

- [react-searchkit/architecture.md](react-searchkit/architecture.md) — the
  Redux store and render tree a bulk-actions context wraps around.
- [react-searchkit/components.md](react-searchkit/components.md) — the
  `withState` HOC used to refresh results after a bulk action completes.

## Two independent implementations exist — know which one you're using

There is **no single shared bulk-actions system**. Two separate
implementations exist with the same shape but are not interchangeable:

1. **`react-invenio-forms`'s exported version** — `SearchResultsBulkActionsManager`,
   `SearchResultsRowCheckbox`, `SearchResultsBulkActions`, `BulkActionsContext`.
   `invenio_administration` wires this in as its search app's context
   wrapper (see below), but the base admin package's default result item and
   layout **don't actually render any checkbox or bulk-action toolbar** —
   the machinery is present but unused until a specific admin resource adds
   `SearchResultsRowCheckbox`/`SearchResultsBulkActions` itself. Don't assume
   bulk actions "just work" on every admin search page — they're an
   opt-in extension point.
2. **`invenio_communities`'s own local reimplementation** — same
   component names and shape, hand-copied rather than imported from
   `react-invenio-forms` (`members/components/bulk_actions/`). If you're
   working inside `invenio_communities`, import from there, not from
   `react-invenio-forms` — the two are not the same objects and mixing them
   won't share context correctly.

## The shape (same in both implementations)

- **`BulkActionsContext`** — `{ bulkActionContext, addToSelected, allSelected, setAllSelected, selectedCount }`.
- **`SearchResultsBulkActionsManager`** — the stateful `Provider`. Tracks a
  plain object map keyed by row id, `{ [rowId]: { selected, data } }`, plus
  `selectedCount`/`allSelected`. `addToSelected(rowId, data)` toggles one
  row; `setAllSelected(value, global)` bulk-toggles everything.
- **`SearchResultsRowCheckbox`** — put one in each result-row component;
  self-registers into the context on mount via a `rowId`/`data` prop pair.
- **`SearchResultsBulkActions`** — the "N selected" toolbar: a select-all
  checkbox plus a `Dropdown` of bulk actions, disabled while
  `selectedCount === 0`.

## Wiring a new search page for bulk actions

1. Wrap the whole search app in a `SearchResultsBulkActionsManager` by
   passing it (optionally combined with other context providers) as the 5th
   argument (`ContainerComponent`) of `createSearchAppInit` — see
   [search.md §2](search.md#2-entrypoint-pattern-used-across-every-invenio-package)
   for that argument's role. Real example
   (`invenio_administration/src/search/SearchBulkActionContext.js`):

   ```js
   export class SearchBulkActionContext extends Component {
     render() {
       const { children } = this.props;
       return (
         <NotificationController>
           <SearchResultsBulkActionsManager>{children}</SearchResultsBulkActionsManager>
         </NotificationController>
       );
     }
   }
   // wired in: createSearchAppInit(defaultComponents, true, "invenio-search-config", false, SearchBulkActionContext);
   ```

2. Put `<SearchResultsRowCheckbox rowId={result.id} data={result} />` inside
   your custom `"ResultsList.item"` override (see
   [search.md §6](search.md#6-customizing-the-result-item)).
3. Render `<SearchResultsBulkActions bulkDropdownOptions={[{key, value, text}, ...]} optionSelectionCallback={(value, selectedMap, count) => ...} />`
   somewhere in the layout.
4. Inside the callback, act on the selected rows (`selectedMap` is the
   `{rowId: {selected, data}}` map), then **refresh the search results and
   clear the selection**:

   ```js
   // real pattern from invenio_communities' ManagerMemberBulkActions
   await this.cancellableAction.promise;
   updateQueryState(currentQueryState); // react-searchkit's withState — re-fetches
   setAllSelected(false, true);         // clears the selection
   ```

   `updateQueryState`/`currentQueryState` come from react-searchkit's
   `withState` HOC (see
   [react-searchkit/components.md](react-searchkit/components.md#connection-pattern))
   — a bulk-action component is typically `withState`-wrapped for exactly
   this reason, in addition to consuming `BulkActionsContext`.

If a bulk action needs a confirmation step first (the common case for a
destructive bulk action), open it from the `optionSelectionCallback` using
the modal idiom in [action-modals.md](action-modals.md) rather than firing
the action immediately.
