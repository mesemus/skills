# react-searchkit — architecture

Read this when you need to understand *why* a `react-searchkit` component
behaves the way it does, when building a fully custom component that talks
to the store directly, or when debugging why state isn't updating/isn't
shared the way you expect.

## 1. It's a Redux app, not a plain Context reducer

`<ReactSearchKit>` (the root component) builds an `appConfig` from its props
(`searchApi`, `suggestionApi`, `urlHandlerApi`, `searchOnInit`,
`initialQueryState`, `defaultSortingOnEmptyQueryString`) and calls
`createStoreWithConfig(appConfig)`, which creates a real `redux` store with
`redux-thunk` middleware — and injects `appConfig` as the **thunk extra
argument**. This is how action creators reach `searchApi`/`urlHandlerApi`
without prop-drilling them through every component:

```js
// any thunk, simplified
export const executeQuery = () => (dispatch, getState, config) => {
  const { searchApi, urlHandlerApi } = config; // the injected appConfig
  ...
};
```

## 2. Store shape

```js
combineReducers({
  app: appReducer,       // { hasUserChangedSorting, initialSortBy, initialSortOrder }
  query: queryReducer,   // { queryString, sortBy, sortOrder, page, size, filters, hiddenParams, layout, suggestions }
  results: resultsReducer, // { loading, data: { hits, total, aggregations }, error }
});
```

Every built-in component reads exactly the slice it needs via `connect()`
(see [components.md](components.md)). If you write a fully custom component
that needs broad read access instead of a couple of specific fields, prefer
the `withState()` HOC (§5) over hand-rolling your own `connect()` mapping.

## 3. Render tree

`ReactSearchKit.render()` wraps `children` in, from outside in:

1. `AppContext.Provider` — provides `{ appName, buildUID, nextComponentIndex }`
   (plain React context, **not** used for query/results state — only for the
   override-id/DOM-id helpers below).
2. `<Provider store={this.store}>` (react-redux) — the actual state.
3. `Bootstrap` — a lifecycle component: dispatches `onAppInitialized` on
   mount, listens to `window.onpopstate` (browser back/forward) and an
   optional custom `queryChanged` window event (§6), and re-runs the query
   from the URL when either fires.
4. An `Overridable` wrapper around `children`, from the separate
   `react-overridable` package (see the skill's
   [conventions.md](../conventions.md#1-overridable-components--the-central-customization-mechanism)
   for that mechanism in general).

## 4. Query flow

Any UI action (typing in `SearchBar`, clicking a facet checkbox, changing
page) dispatches a thunk that:

1. Updates the relevant slice of `query` state.
2. Updates the URL via `urlHandlerApi` (see
   [url-state.md](url-state.md)).
3. Dispatches `RESULTS_LOADING`.
4. Calls `config.searchApi.search(queryState)`.
5. Dispatches `RESULTS_FETCH_SUCCESS` with `{aggregations, hits, total}`, or
   `RESULTS_FETCH_ERROR`.

`executeQuery` (the thunk that does all of this) also special-cases
`RequestCancelledError`, silently ignoring it rather than surfacing it as a
result error — see [api.md](api.md#request-cancellation) for why that
matters.

## 5. `buildUID` — multi-instance support

```js
// util.js
export function buildUID(elementName, overridableId = "", appName = "") {
  const _overridableId = overridableId ? `.${overridableId}` : "";
  const _appName = appName ? `${appName}.` : "";
  return `${_appName}${elementName}${_overridableId}`;
}
```

`ReactSearchKit` exposes a bound version on `AppContext`:

```js
buildUID: (element, overrideId) => buildUID(element, overrideId, appName),
nextComponentIndex: () => `${this.appName}_${this.componentIndex++}`,
```

Every internal `Element` subcomponent calls
`useContext(AppContext).buildUID("ComponentName.element", overridableId)` to
compute the id it passes to `<Overridable id={...}>`. `buildUID` serves
**two separate purposes**, both governed by the single `appName` prop on
`<ReactSearchKit>`:

- **Override namespacing** — prefixes the Overridable id so two
  `ReactSearchKit` trees on one page can register different overrides for,
  say, `SearchBar.element` without colliding.
- **DOM id uniqueness** — `nextComponentIndex()` generates ids like
  `${appName}_N` used for facet checkbox `id`/`htmlFor` pairs, needed because
  two instances render structurally identical checkbox trees; without a
  distinct `appName` these ids would collide across instances even though
  React itself wouldn't complain.

**Practical rule: if more than one `<ReactSearchKit>` renders on a page, give
each a distinct `appName`.** There is no shared state between instances
regardless — each builds its own store — so `appName` (plus typically a
distinct `searchApi` URL) is purely what keeps their DOM ids and override
registrations from colliding, not a mechanism for sharing anything between
them.

For the Invenio-specific wiring of this (`createSearchAppInit`'s `multi`
flag, `appId` sourced from the page's JSON config), see
[../search.md §3](../search.md#3-appname--multi--required-for-more-than-one-search-app-per-page).

## 6. Cross-instance events

`Bootstrap` also wires an optional custom `queryChanged` window event
(`events.js`): dispatching `onQueryChanged(payload)` with `payload.appName`
lets external code (e.g. a non-React widget) trigger a specific
`ReactSearchKit` instance to re-run its query. `Bootstrap.onQueryChanged`
ignores events whose `appName` doesn't match its own context's `appName`, so
this only works for cross-instance triggering when the `appName`s line up.

## 7. Known footgun: duplicate library copies break `AppContext`

`AppContext = React.createContext({})` relies on referential identity. If
your build ends up bundling **two separate copies** of `react-searchkit`
(a classic `npm link`/monorepo/vendoring pitfall — e.g. vendoring the
package into a theme while also having it as a normal dependency elsewhere),
`useContext(AppContext)` in components from one copy won't see the provider
from the other, and `buildUID`/`nextComponentIndex` silently become
`undefined`. If overrides mysteriously never match or facet checkboxes get
duplicate DOM ids, check for a duplicate-copy situation before assuming a
logic bug.
