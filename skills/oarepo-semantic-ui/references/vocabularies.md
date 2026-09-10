# `oarepo_vocabularies_ui` — vocabulary management and picking

Read this before building anything that manages vocabulary terms (create/
edit/browse) or that lets a *different* form pick a term from a vocabulary.
**This file corrects a natural assumption**: despite the vocabulary data
model being hierarchical, there is no tree-editor widget here.

## The three apps

`oarepo_vocabularies_ui` ships three independent entry-point apps, all built
on `@js/oarepo_ui` (forms/search), not a separate framework:

- **`form/`** — Formik CRUD editor for a *single* vocabulary term (used for
  both create and edit), with per-vocabulary-type field-set overrides
  (`VocabularyFormFieldsAwards`/`Names`/`Funders`/`Affiliations`). A normal
  record-editor page, not an in-page tree editor.
- **`search/`** — the admin *listing* page for one vocabulary type
  (`/vocabularies/<type>`), an ordinary `createSearchAppsInit` app themed
  for vocabularies (breadcrumbs, a "New item" sidebar button gated on
  `SearchConfigurationContext.permissions.can_create`).
- **`detail/`** — a **second**, smaller search app
  (`DetailSearchApp.jsx`) embedded on a single term's detail page, listing
  that term's direct **children** as ordinary paginated search results.

Map: create/edit a term → `form/`; browse/manage the flat list of a
vocabulary type → `search/`; see one term's children → `detail/`; embed a
vocabulary picker in some *other* record's deposit form → `VocabularyField`
(below).

## Hierarchy is display-only — there is no tree widget

There is **no recursive/nested tree-rendering component anywhere in this
package** — no drag-and-drop reordering, no expand/collapse node tree, no
move/reparent UI. If you were expecting something like `invenio_collections`'
nested-tree pattern (documented in the sibling skill's
`reusable-widgets.md`), it doesn't exist here. Hierarchy is instead handled
through three separate, lightweight, purely display/navigation mechanisms:

1. **Flat listing + breadcrumbs.** The admin search page is a plain flat
   results list; each result item renders a `Breadcrumb` built from
   `result.hierarchy.ancestors`/`hierarchy.title`.
2. **A per-term "descendants" search app.** `detail/DetailSearchApp.jsx` is
   a fully separate `ReactSearchKit` instance on a term's detail page,
   showing children as another flat, paginated list with a
   `totalDescendants` count — not an expandable tree. Navigating deeper
   means clicking into a child's own detail page, which mounts its own copy
   of this same app.
3. **"Add child" is a URL query parameter on the existing create form**,
   not a tree-node action. The create form reads `?h-parent=<id>`:

   ```jsx
   // form/FormAppLayout.jsx
   const searchParams = new URLSearchParams(location.search);
   const newChildItemParentId = searchParams.get("h-parent");
   ```

   and the save thunk attaches the parent when creating:

   ```js
   // form/state/deposit/actions.js
   if (newChildItemParentId) {
     draftToSave = { ...draftWithUUID, hierarchy: { parent: newChildItemParentId } };
   }
   ```

   `form/components/CurrentLocationInformation/` shows one of three
   context messages (new top-level item / new child item, fetching the
   parent's breadcrumb via `useQuery` / editing a level-N item) as
   **informational text only** — not an interactive control.

There's no move/reparent, reorder, or bulk-delete UI in this JS layer at
all. If you need to build hierarchical management UI for something else,
don't copy this package's approach as a "how OARepo does trees" reference —
build the recursive-tree pattern from the sibling skill's
`reusable-widgets.md` instead, or a purpose-built one; this package
deliberately avoids that complexity in favor of page-navigation.

## `VocabularyField` — the generic, reusable picker

`form/components/VocabularyField/VocabularyField.jsx` is what you actually
want when a *different* form needs to let the user pick a term from some
vocabulary. It wraps react-invenio-forms' `RemoteSelectField`
(not `AutocompleteDropdown`) and adds OARepo-specific plumbing:

```jsx
<VocabularyField vocabularyName="languages" fieldPath="metadata.language" />
```

points at `/api/vocabularies/languages` with no further configuration. Key
behaviors beyond plain `RemoteSelectField`:

- **Value shape normalization** — stored as `{id}` (single) or `[{id}, ...]`
  (multiple), matching Invenio-RDM's relation-reference shape; the field
  translates to/from the plain id strings the underlying dropdown works
  with.
- **`ui.*` initial-suggestion bootstrapping** — derives a `ui.<path>` lookup
  from `fieldPath` (swapping `metadata.` → `ui.`) to seed the dropdown's
  initial display value from the record's server-rendered UI serialization
  (`{id, title_l10n}`), avoiding an extra fetch on load.
- **Hierarchy-aware suggestion rendering** — `showLeafsOnly` plus shared
  `utils.js` helpers (`processVocabularyItems`/`serializeVocabularySuggestions`)
  render breadcrumb-style multi-level labels for hierarchical vocabularies
  and can filter suggestions to leaf nodes only.
- **Free-text additions** via `serializeAddedValue`, for vocabularies that
  allow ad hoc values.

Use this as the default for "pick from vocabulary X" fields; only reach for
the narrower, hand-written `RemoteSelectField` configs (like
`VocabularyFormFieldsAwards`/`Names` use for funder/affiliation-by-name
lookups) when the API/value shape genuinely doesn't fit `VocabularyField`'s
assumptions.

## Other reusable pieces worth knowing

- **`VocabularyMultilingualInputField`** — a generic "array of `{lang,
  name}` pairs ↔ object keyed by lang" multilingual title editor, built on
  `ArrayField` + `LanguageSelectField` + `oarepo_ui/util`'s
  `array2object`/`object2array`. Reusable for any multilingual title/name
  field, not just vocabularies.
- **`ClipboardCopyButton`-based identifier display** — vocabulary result
  items use `@js/oarepo_ui/components/ClipboardCopyButton` to render
  identifier lists with copy-to-clipboard and auto-linking — see
  [reusable-widgets.md](reusable-widgets.md).
- **"Derive id from another field" hook idiom** — the Funders/Names/Awards
  field sets each auto-derive a record `id` from other fields (e.g.
  `scheme:identifier`) on create only, not update — a repeatable pattern if
  you need similar slug-derivation behavior elsewhere.
