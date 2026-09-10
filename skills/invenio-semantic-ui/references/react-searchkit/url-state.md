# react-searchkit — URL state synchronization

Read this when a search page needs to support bookmarking/sharing, when
browser back/forward doesn't behave as expected, or when you need a custom
URL parameter shape.

## No router dependency

`react-searchkit` does not depend on `react-router` or any router library.
`UrlHandlerApi` manipulates `window.history` directly, and is **enabled by
default**:

```js
ReactSearchKit.defaultProps.urlHandlerApi = {
  enabled: true,
  overrideConfig: {},
  customHandler: null,
};
```

## Reading state from the URL

`UrlHandlerApi.get(queryState)`:

1. Parses `window.location.search` with `qs`.
2. Maps URL param keys back to state keys via a configurable
   `urlParamsMapping` (default below).
3. Validates each value with `UrlParamValidator`.
4. Merges the result into `queryState`.
5. Immediately calls `replaceHistory` to canonicalize the URL (so a
   malformed/partial URL becomes a clean one without adding a history entry).

Default `urlParamsMapping`:

| state key | URL param |
|---|---|
| `queryString` | `q` |
| `sortBy` | `sort` |
| `sortOrder` | `order` |
| `page` | `p` |
| `size` | `s` |
| `layout` | `l` |
| `filters` | `f` |
| `hiddenParams` | `hp` |

## Writing state to the URL

- `set(queryState)` — pushes a new history entry (`window.history.pushState`)
  unless `keepHistory: false`, in which case it replaces instead.
- `replace(queryState)` — always `window.history.replaceState`.

`executeQuery` calls `updateURLParameters` before firing the request,
choosing push vs. replace based on `shouldReplaceUrlQueryString`/
`shouldUpdateUrlQueryString` flags — you generally don't need to touch this
directly as long as you dispatch through the provided action creators
(`updateQueryString`, `updateQueryFilters`, `updateQueryPage`, ...) rather
than mutating state ad hoc.

## Filter serialization format

Filters are serialized to a **human-readable colon/plus string**, not JSON:

```
["type", "photo", ["subtype", "png"]]  →  "type:photo+subtype:png"
```

The separator between filters (`+` above) is configurable via
`urlFilterSeparator`.

## Browser back/forward

`Bootstrap` (see [architecture.md](architecture.md#3-render-tree)) wires
`window.onpopstate` to re-run `updateQueryStateFromUrl()`, so navigating back
reloads both the query state and the results — this is automatic and
requires no extra wiring in a normal search page.

## Composition root

There is no `createSearchAppInit`-equivalent helper inside `react-searchkit`
itself; `<ReactSearchKit>` *is* the top-level composition root for a single
search instance. `initialQueryState` seeds defaults (used e.g. to preselect
a filter or sort order before any user interaction), and
`defaultSortingOnEmptyQueryString` lets you force a different sort when the
query string is empty vs. non-empty — tracked via
`state.app.hasUserChangedSorting` so it won't override a sort the user
explicitly picked. The Invenio-specific bootstrap helper that reads a
Jinja-rendered config and mounts `<ReactSearchKit>` for you is
`createSearchAppInit`, from `invenio_search_ui` — see
[../search.md §2](../search.md#2-entrypoint-pattern-used-across-every-invenio-package).
