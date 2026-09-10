# react-invenio-forms — non-field UI primitives

`react-invenio-forms` exports more than form fields (see
[components.md](components.md) for those). This file covers the rest of its
exports that are generic, reusable UI primitives — worth knowing so you
reach for these instead of rebuilding an avatar-with-fallback, an accessible
tooltip, or a rich autocomplete-option renderer from scratch.

## `Image`

`<img>` wrapper with automatic fallback-on-error, and an optional "load the
fallback first, then swap to the real `src` once it loads" mode. Toggles
`.placeholder`/`.fallback_image` CSS classes for loading states. Props:
`src`, `fallbackSrc` (default `/static/images/square-placeholder.png`),
`alt`, `loadFallbackFirst`. The standard avatar/logo/thumbnail primitive
across the ecosystem — used internally by `UserListItemCompact` and
`AffiliationsSuggestions`. Use this instead of a raw `<img>` anywhere the
URL might 404.

## `FilesList`

Renders an array of already-uploaded files as removable Semantic `Label`
chips (icon + `"name (size)"`, clickable through to `links.download_html`).
Props: `files` (`{file_id, original_filename, size, links.download_html}[]`),
`onFileDelete`. A **read-only/compact** file display — distinct from the
full drag-and-drop deposit uploader (see [../file-upload.md](../file-upload.md)).
Good for e.g. an attachment list on a request or comment; used by
`invenio_requests`' `TimelineEventBody.js`.

## `GridResponsiveSidebarColumn`

Renders a `Grid.Column` twice: once as a slide-out `Sidebar` (mobile/tablet,
with a focus-managed close button) and once as a static column (desktop) —
so callers get a responsive facets/filters sidebar without hand-building the
breakpoint logic. Props: `mobile`/`tablet`/`computer`/`widescreen`/
`largeScreen`, `width`, `open`, `onHideClick`, `ariaLabel`. This is what
every search app layout (`SearchApp.js` and its overrides — see
[../search.md](../search.md)) uses to host the facets column. If building a
custom search layout, reuse this rather than a plain `Grid.Column`.

## `toRelativeTime(timestamp, language)`

`"3 days ago"`-style relative timestamp (Luxon under the hood). Takes an
ISO timestamp string and a language code (pass `i18next.language`). Used
everywhere a timestamp is shown compactly (request timelines, admin search
rows). Trivial but avoid reimplementing.

## `UserListItemCompact`

Standard "avatar + name (+ Group/You labels) + affiliation meta line" row,
optionally linking to a detail view. Props: `id`, `user`
(`{profile, username, links.avatar, type, is_current_user}`),
`linkToDetailView`. Directly reusable for any "list of users" UI —
membership pickers, moderation queues, admin user search rows.

## `AffiliationsSuggestions(creatibutors, isOrganization)`

Not a component — a **serializer function** that turns raw vocabulary/
affiliation records into `RemoteSelectField`-ready option objects with rich
`<Header>`/`<Header.Subheader>` content (identifier icons via `makeIdEntry`,
a location/type summary via `makeSubheader`). Wrapped in an
`Overridable id="ReactInvenioForms.AffiliationsSuggestions.content"`. The
reusable takeaway is the *pattern*: building rich dropdown-option content
plus an override point for it — copy this shape for any custom vocabulary/
person picker, not the affiliations domain itself.

## `makeIdEntry` / `makeSubheader`

Helpers used inside `AffiliationsSuggestions`: `makeIdEntry` renders a
clickable identifier icon+link for known schemes (ORCID/GND; skips
ROR/ISNI/GRID); `makeSubheader` builds a location/type/affiliation summary
string. Exported standalone so a custom vocabulary/person picker can reuse
the same scheme→icon mapping instead of reimplementing it.

## `AutocompleteDropdown`

A **generic** autocomplete field: point it at any Invenio REST endpoint
(`autocompleteFrom`) and it wires up `RemoteSelectField` with a default
`title_l10n`/`id` suggestion serializer and the
`application/vnd.inveniordm.v1+json` Accept header. Props: `fieldPath`,
`autocompleteFrom` (URL), `autocompleteFromAcceptHeader`, `multiple`,
`clearable`, `label`/`icon`. The component to reach for when you need
"autocomplete from a vocabulary API" without hand-building a
`RemoteSelectField` config each time — most relevant to custom-fields
authors (see [custom-fields.md](custom-fields.md)).

## `SubjectAutocompleteDropdown`

A pre-built `RemoteSelectField` wrapper hitting `/api/subjects`, supporting
a `limitTo` scheme prefix (queries as `` `${limitTo}:${query}` ``) and
free-text additions. Props: `fieldPath`, `limitTo`, `multiple`,
`allowAdditions`, `noQueryMessage`. A concrete, narrower example next to
`AutocompleteDropdown`'s generic form — useful as a template for a
similarly-scoped vocabulary field.

## `InvenioPopup`

An accessibility-hardened wrapper around Semantic UI's `Popup`: forces
`on={["hover", "focus"]}` (plain `Popup` triggers aren't keyboard-focusable
by default), clones the trigger with `role="button" tabIndex={0} aria-label`,
and wraps content in `<p role="tooltip" aria-live="polite">`. Props:
`trigger`, `content`, `ariaLabel`, `position`, `inverted`, `hoverable`,
`size`. Use this instead of a raw `Popup` for any tooltip — it's the
accessible-tooltip idiom used throughout this library (e.g. inside
`FeedbackLabel`, see [errors-and-labels.md](errors-and-labels.md)).
