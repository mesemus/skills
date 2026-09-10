---
name: invenio-semantic-ui
description: Write React components for the Invenio digital-repository ecosystem's Semantic UI theme — deposit/edit forms (react-invenio-forms + Formik) and search/listing pages with facets and aggregations (react-searchkit). Use when writing or editing any *.js/*.jsx under an invenio_* package's assets/semantic-ui/js folder, a deposit form, a metadata field, a search results page, an aggregation/facet, or a "SearchApp"/"DepositForm" component. Covers component-authoring principles and library conventions only, not build/webpack wiring or Python-side registration.
metadata:
  author: miroslav.simek@cesnet.cz
---

# Invenio React + Semantic UI components

This skill covers how to *write* React components in the Invenio ecosystem:
deposit/edit forms and search/listing pages built on Semantic UI React. It
does not cover building assets, registering blueprints, or otherwise wiring
a component into a running Invenio instance — that's a separate concern.

## The stack

- **React 16**, class and function components both common (no hooks-only
  convention — see [conventions.md](references/conventions.md)).
- **`semantic-ui-react`** for all visual primitives (`Form`, `Grid`, `Item`,
  `Card`, `Modal`, ...). No custom CSS framework, no Tailwind.
- **Formik** for form state, wrapped by **`react-invenio-forms`**, which
  supplies ready-made Semantic-UI-flavored fields (`TextField`, `SelectField`,
  `ArrayField`, `AccordionField`, ...). Build forms by composing these, not by
  hand-rolling `<Field>` + `<Form.Input>` yourself.
- **`react-searchkit`** for search/listing pages — a self-contained Redux app
  exposing `SearchBar`, `Sort`, `ResultsList`, `BucketAggregation`,
  `Pagination`, etc. Build search pages by composing these.
- **`react-overridable`** — the customization mechanism used everywhere: any
  built-in component can be swapped by a downstream deployment via a dotted
  string ID, without forking. This is a first-class concept here, not an
  edge case — see §"Overridable components" below.
- **`i18next`**, one instance per Python package, gettext-style (translation
  key = literal English string).
- **No TypeScript.** `PropTypes` everywhere.

Load the reference files as needed — they contain the concrete APIs, code
patterns, and gotchas that make the difference between idiomatic and
almost-right code. Each one names its own sub-references and exact "read
this when" trigger at the top, so following one link is usually enough to
find the next.

**Writing a form or search page (the common case) — start here:**

- **[references/forms.md](references/forms.md)** — building deposit/edit
  forms: field components, `ArrayField` repeatable-groups pattern, error/
  severity shape, section accordions, multi-action submit buttons, custom
  fields. Read before writing or editing any form.
- **[references/search.md](references/search.md)** — building search/
  listing pages: `ReactSearchKit` composition, aggregation/facet config,
  custom result items, custom facets, multi-app pages. Read before writing
  or editing any search page or facet.
- **[references/conventions.md](references/conventions.md)** — the
  overridable-components mechanism in full, i18n rules, HTTP client, and
  notification/confirmation-dialog conventions. Read before writing any
  reusable component, or if unsure how a component's props/config reach it.

**Anything else the task touches — check before building from scratch
(principle 7):**

- **[references/action-modals.md](references/action-modals.md)** — a
  button that calls an API and needs a loading/error/confirm state.
- **[references/bulk-actions.md](references/bulk-actions.md)** — checkbox
  row-selection plus a "do X to selected results" toolbar.
- **[references/file-upload.md](references/file-upload.md)** — file
  upload/management (`FileUploader` vs `UppyUploader`, quota).
- **[references/reusable-widgets.md](references/reusable-widgets.md)** — a
  catalog of smaller pieces (loaders, error boundaries, inline async-save
  feedback, danger-zone deletes, responsive action collapsing, ...).

## Core principles

1. **Compose the provided components; don't reinvent their plumbing.**
   `react-invenio-forms` and `react-searchkit` already solve Formik/Redux
   wiring, error display, URL sync, and pagination. Writing a form or search
   page is almost entirely about *composition and configuration* of existing
   building blocks, plus a handful of small glue components (a resource-type
   dropdown, a custom result-item renderer). Reaching for raw Formik `Field`
   or raw Redux in this codebase is a sign you've missed an existing
   component.

2. **Everything is addressed by `fieldPath` (forms) or config objects with an
   `aggName`/`field` (search).** There is no other addressing scheme. When
   composing a repeatable field or a nested facet, always derive child paths/
   names from what the parent component's render-prop or config gives you.

3. **Validation is server-authoritative for forms.** Real deposit forms in
   this ecosystem have no Yup `validationSchema` — `required` on individual
   fields is the extent of client-side validation; everything else comes back
   from the API as `{field, messages, severity}` and gets mapped into
   Formik's error object. Don't introduce a Yup schema as the default
   approach; follow the field-error-mapping pattern in
   [forms.md](references/forms.md#8-validation-is-server-authoritative--dont-reach-for-yup)
   instead.

4. **Errors and facets are not plain strings/values — expect structure.**
   Form field errors can be `{message, severity, description}`, not just a
   string. Search filters are always arrays (`["aggName", "value"]`, or
   nested for hierarchical facets), never bare strings. Code that assumes the
   simpler shape will work until the first severity-tagged error or nested
   facet appears, then break confusingly.

5. **Customization happens via `react-overridable` IDs, not forking.** Before
   writing a component you expect a deployment might want to customize
   (a form field, a result-item renderer, a whole layout), check whether the
   surrounding code already wraps it in an `Overridable`/`Overridable.component`
   — if so, preserve that wrapping. When adding a genuinely new customizable
   piece, wrap it the same way, using the existing dotted-namespace ID
   convention. Full detail: [conventions.md](references/conventions.md#1-overridable-components--the-central-customization-mechanism).

6. **i18n strings are static literals passed to a per-package `i18next.t()`.**
   Never build a translation key by concatenation, and always import the
   `i18next` instance belonging to the package you're editing (not a
   different one you happened to see in another file).

7. **Check the reusable-component catalog (second group above) before
   building common UI from scratch.** Confirmation modals, async action
   buttons, bulk/multi-select search actions, file upload, and small pieces
   like loaders/error boundaries/danger-zone deletes all already exist in
   some form across the ecosystem. These are hand-rolled patterns repeated
   with variation, not always literal shared components — imitate the
   closest existing shape rather than inventing a new one.

## Writing a deposit/edit form — quick procedure

1. Identify the record's field paths you need to expose (e.g.
   `metadata.title`, `metadata.creators`, `custom_fields.imprint:publisher`).
2. For scalar fields, use `TextField`/`SelectField`/`RemoteSelectField`/
   `RichInputField` directly with `label`, `helpText`, `required` as needed.
3. For repeatable data (creators, identifiers, dates, related works), use
   `ArrayField` with a render-prop building one `GroupField` row per item —
   see the worked example in [forms.md §5](references/forms.md#5-repeatable-groups--arrayfield-there-is-no-arrayfielditem).
4. If a field needs a vocabulary lookup or any non-trivial record↔form value
   mapping, write a small serialize/deserialize pair rather than binding the
   field to the raw record shape.
5. Group fields into `AccordionField` sections, each declaring its
   `includesPaths` so validation errors surface as section-level badges.
6. If the form needs more than one submit action (Save / Publish / Preview),
   use the shared-context pattern in
   [forms.md §9](references/forms.md#9-multiple-submit-intents-from-one-formik-form)
   rather than trying to give Formik multiple `onSubmit`s.
7. Wire up error handling by mapping the API's field-error response onto
   Formik's `errors`/`initialErrors`, not by writing a Yup schema.

## Writing a search/listing page — quick procedure

1. Confirm the target really needs its own `<ReactSearchKit>` tree — some
   packages only define facet *config* and rely on another package's generic
   search app (see [search.md §8](references/search.md#8-gotchas-checklist)).
2. Compose the standard layout: `SearchBar`, `Sort`, `ResultsList`/
   `ResultsGrid` (wrapped in `ResultsLoader`), `Pagination`,
   `BucketAggregation` per facet, `ActiveFilters`, `EmptyResults`, `Error`.
3. Customize the per-hit rendering by registering a component for
   `"ResultsList.item"` (and `"ResultsGrid.item"` for grid view) — not via a
   render prop.
4. For each facet, use the `{aggName, field, title, childAgg?}` config shape;
   reach for a hand-rolled `withState`-based component only when the UX needs
   exclusive/tab-like selection instead of independent checkboxes (see
   [search.md §5](references/search.md#5-aggregations--facets)).
5. If more than one search app can appear on the same page, pass a distinct
   `appName` and prefix every override-map key with it — see
   [search.md §3](references/search.md#3-appname--multi--required-for-more-than-one-search-app-per-page).
6. Don't hand-roll URL state, pagination math, or debounced request
   cancellation — the library already does all three.

## Gotchas worth remembering up front

- `FastField`/`optimized={true}` can silently suppress needed re-renders
  inside repeatable fields — default to off unless verified safe.
- A custom `onChange` on `SelectField` must call `setFieldValue` itself; the
  field won't do it for you once you override `onChange`.
- `required` + `disabled` together defeats HTML5 required validation.
- Aggregation buckets may arrive as an array or an object keyed by bucket key
  — normalize both if bypassing `BucketAggregation`.
- Under `multi=true` search apps, every override-map key needs the
  `${appName}.` prefix; under `multi=false`, it must not have one.

See the reference files for the full, sourced detail behind every point
above.
