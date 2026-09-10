# react-searchkit — search API contract

Read this when wiring a `ReactSearchKit` instance to a backend, writing a
custom `searchApi`/`suggestionApi`, or debugging a request/response shape
mismatch.

## 1. The `searchApi` contract

`<ReactSearchKit searchApi={...}>` needs an object exposing an async
`.search(stateQuery)` that resolves to `{ aggregations, hits, total, extras? }`.
The default implementation is `InvenioSearchApi`, but any object satisfying
this contract works — useful if you're pointing search at a non-Elasticsearch
backend.

## 2. `InvenioSearchApi`

Wraps `axios`. Construction requires `config.axios.url` (throws otherwise):

```js
new InvenioSearchApi({
  axios: { url: "/api/records", headers: { Accept: "application/json" } },
  invenio: { requestSerializer: InvenioRequestSerializer /* default */ },
});
```

**Request serialization** — `serialize(stateQuery)` builds the axios
`params` querystring from the current `query` slice: `q` (query string),
`sort` (with a `-` prefix for descending), `page`, `size`, plus filter params
flattened from the `filters` array. Two serializer variants exist,
selectable via `config.invenio.requestSerializer`:

- `InvenioRequestSerializer` — the default, generic invenio-search-ui shape.
- `InvenioRecordsResourcesRequestSerializer` — for the newer
  invenio-records-resources-style query params.

Both explicitly validate filter values: pushing a plain string (instead of
an array) into `query.filters` throws
`Filter value "..." in query state must be an array` — see
[aggregations.md](aggregations.md#filter-data-model) for the required shape.

**Request cancellation** — every call to `.search()` cancels any previous
in-flight request via an `axios.CancelToken` before firing the new one. This
is Invenio's substitute for time-based debouncing: rapid-fire dispatches
(e.g. from a facet being clicked repeatedly) are safe because only the last
request's response is ever used, but **every dispatch still hits the
network** — there is no built-in delay. If you want to avoid a network call
per keystroke on a live-search field, add your own debounce at the UI layer
before dispatching (see [components.md](components.md#searchbar) for why
the built-in `SearchBar` doesn't need this itself).

**Response serialization** — `InvenioResponseSerializer.serialize(payload)`:

```js
serialize(payload) {
  const { aggregations, hits, ...extras } = payload;
  return { aggregations: aggregations || {}, hits: hits.hits, total: hits.total, extras };
}
```

So the backend response must look like an Elasticsearch/OpenSearch response:
`{ hits: { hits: [...], total: N }, aggregations: {...}, ...anythingElse }`.
Any other top-level keys land in `extras`, which `executeQuery` merges back
into `query` state as `newQueryState` — this is how a suggest/correct-style
API can echo back adjusted query params (e.g. a corrected spelling) that the
UI should adopt.

## 3. Request cancellation vs. `RequestCancelledError`

`executeQuery`'s error handling explicitly special-cases and swallows
`RequestCancelledError` (exported from the package) rather than dispatching
`RESULTS_FETCH_ERROR` for it — a cancelled-because-superseded request must
never flash an error state. If you write a custom `searchApi`, throw/reject
with this same error class for cancellations, or your custom search will
show spurious error flashes whenever a newer request supersedes an older one.

## 4. OpenSearch variant

`OSSearchApi` / `OSRequestSerializer` / `OSResponseSerializer` are a raw
OpenSearch-flavored contrib variant (no Invenio-specific request/response
massaging) — use these only if talking to OpenSearch directly rather than
through an Invenio REST API.

## 5. Suggestions / autocomplete

`InvenioSuggestionApi` extends `InvenioSearchApi` and powers
`AutocompleteSearchBar` (see [components.md](components.md#autocompletesearchbar)).
It requires additional config:

```js
suggestionApi: {
  axios: { url: "/api/records/_suggest" },
  invenio: {
    suggestions: {
      queryField: "suggest",      // param name sent to the backend
      responseField: "suggest",   // field read back out of the response
    },
  },
}
```

## Key exports

`InvenioSearchApi, InvenioSuggestionApi, InvenioRequestSerializer,
InvenioRecordsResourcesRequestSerializer, InvenioResponseSerializer,
OSSearchApi, OSRequestSerializer, OSResponseSerializer, RequestCancelledError`.
