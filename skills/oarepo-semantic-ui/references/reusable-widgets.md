# Catalog of smaller reusable OARepo components

Small, generic, model-agnostic components worth knowing exist before
rebuilding them. For the large subsystems, see [forms.md](forms.md),
[search.md](search.md), and [vocabularies.md](vocabularies.md). For a real
model's search result item composing several of these together, see
[wiring.md](wiring.md) and the `ResultsListItem.jsx` it references.

## `SearchItemCreators` (base RDM, reused everywhere)

`@js/invenio_app_rdm/utils` exports `SearchItemCreators` — the standard
creators/contributors list renderer for a search result item (name list
with an "others" link/overflow handling). Every real per-model
`ResultsListItem` seen in practice imports this rather than hand-rendering
`result.ui.creators`/`result.ui.contributors` — use it instead of
re-deriving creator-list formatting from scratch.

## The "expandable description" pattern

Not a shipped component, but a recurring, worth-copying pattern for
truncating a long text block with a "show more/less" toggle, seen in real
`ResultsListItem` implementations:

```jsx
const ExpandableDescription = ({ description }) => {
  const [expanded, setExpanded] = useState(false);
  const { allowedHtmlTags } = useContext(SearchConfigurationContext);
  const needsTruncation = description.length > MAX_DESCRIPTION_LENGTH;
  const html = expanded || !needsTruncation
    ? sanitizeHtml(description, { allowedTags: allowedHtmlTags })
    : sanitizeHtml(description.substring(0, MAX_DESCRIPTION_LENGTH) + "...", { allowedTags: allowedHtmlTags });
  return (
    <Item.Description>
      <span dangerouslySetInnerHTML={{ __html: html }} />
      <button type="button" aria-expanded={expanded} aria-label={expanded ? "Show less" : "Show more"}
        onClick={() => setExpanded((e) => !e)}>
        <Icon name={expanded ? "chevron up" : "chevron right"} />
      </button>
    </Item.Description>
  );
};
```

Two details worth keeping if you copy this: sanitize with `sanitize-html`
using `allowedHtmlTags` read from react-searchkit's
`SearchConfigurationContext` (from `@js/invenio_search_ui/components`) —
don't hardcode an allowed-tags list, use whatever the search app was
configured with — and set `aria-expanded`/`aria-label` on the toggle for
accessibility.

## `ClipboardCopyButton`

`oarepo_ui/components/ClipboardCopyButton.jsx` — a generic "copy to
clipboard" React button (`<ClipboardCopyButton copyText="...">`). Has a
**1:1 server-rendered twin**: `templates/components/ClipboardCopyButton.jinja`
(same `data-clipboard-text` contract, same `.copy-button` CSS classes/click
animation, wired by the plain-JS `components/clipboard.js`
`initCopyButtons`/`deinitializeCopyButtons`). Use the React version inside
a React tree, the JinjaX macro inside a server-rendered page — both produce
identical markup/behavior, so pick whichever matches the surrounding page's
rendering model rather than mounting a whole React island just for one copy
button.

## `IdentifierBadge`

`oarepo_ui/components/IdentifierBadge.jsx` (+ Storybook story) — renders a
creator/contributor identifier badge (ORCID, DOI, ...) with its icon
auto-resolved from `/static/images/{scheme}.svg`, an optional link wrapper,
and a tooltip. Also has a **1:1 JinjaX twin**
(`templates/components/IdentifierBadge.jinja`, same `identifier`/
`creatibutorName`/`fallbackImage` prop shape). Generic and reusable for any
record model with identifier-shaped data — reach for this instead of
hand-rolling identifier-icon rendering (the sibling skill's
`makeIdEntry`/`makeSubheader` helpers solve a related but narrower problem —
building rich *dropdown option* content, not a standalone badge).

## `Disabled`

`oarepo_ui/components/Disabled.jsx` — literally `export const Disabled = ()
=> null;`. Register this as an override for any overridable slot you want
to **hide entirely** in a given deployment, rather than writing your own
no-op component each time:

```js
overrideStore.add("SomePrefix.SomeSlot", Disabled);
```

## `FacetsButtonGroupNameToggler` (`oarepo_dashboard`)

`oarepo_dashboard/dashboard_components/search/FacetsButtonGroupNameToggler.jsx` —
a generic, model-agnostic react-searchkit widget: a button group that lets
the user toggle *which facet name* is active in the current query (e.g.
switching between two mutually-exclusive filters representing the same
concept — "records I created" vs "records shared with me" in a personal
results list). Built on `withState`:

```jsx
export const FacetsButtonGroupNameToggler = withState(FacetsButtonGroupNameTogglerComponent);
```

Takes `toggledFilters` (`{filterName, text}[]`) and an optional
`keepFiltersOnUpdate`. Despite living in the "dashboard" package, this is a
plain, reusable react-searchkit facet component — usable on any faceted
search page, not dashboard-specific. Note the rest of `oarepo_dashboard` is
thin (a styling-only entry point plus this one component) — don't expect a
larger "dashboard framework" here.

## `record-sharing.js` — thin bootstrap, not a new component

`oarepo_ui/components/record-sharing.js` mounts base `invenio_app_rdm`'s
own `ShareButton` (`@js/invenio_app_rdm/landing_page/ShareOptions/ShareButton`)
into a `#recordSharing` div reading `data-record`/`data-permissions`/
`data-groups-enabled`. Not a new component — just an OARepo-side mount
point. If you need record-sharing UI, use base RDM's `ShareButton` directly
rather than looking for an OARepo-specific implementation.

## Page-chrome utilities (not record-model-specific)

- `burgermenu.js` — vanilla jQuery mobile nav toggle
  (`#invenio-burger-menu-icon`/`#invenio-nav`), Escape-key handling.
- `filepreview.js` — vanilla JS wiring `.openPreviewIcon` clicks to open a
  `#preview-modal` Semantic UI modal with an iframe pointed at
  `data-preview-link`. Reusable for any file-list UI needing a quick
  preview modal without a full previewer integration.

## `ExportDropdown.jsx` — present but dormant

`oarepo_ui/components/ExportDropdown.jsx` exists but is **not** wired into
`components/index.js`, not exported, and not its own webpack entry — the
live `#recordExportDownload` mount point is currently served by base RDM's
own `invenio_app_rdm` export dropdown instead. Treat this file as a
prepared-but-inactive override, not something currently in use — don't
assume it's the component actually rendering export dropdowns without
checking the current wiring first.

## `@oarepo/file-manager` — single-file upload/edit dialog

A standalone npm package (`@oarepo/file-manager`, not part of `oarepo_ui`'s
own source), built on Uppy, providing a **per-file** metadata-driven
add/edit dialog — distinct from the sibling skill's `FileUploader`/
`UppyUploader` (which handle the *bulk*, multi-file deposit upload step).
Also supports extracting images out of uploaded PDFs (`pdf-lib`/`pngjs`/
`pako`).

```jsx
import FileManagementDialog from "@oarepo/file-manager";

<FileManagementDialog
  config={{ record }}
  modifyExistingFiles
  allowedFileTypes={...}
  allowedMetaFields={[{ id: "caption", defaultValue: "", isUserInput: true }, ...]}
  autoExtractImagesFromPDFs={false}
  locale="en_US" // or "cs_CZ"
  startEvent="edit-file" // | "upload-file-without-edit" | "upload-images-from-pdf"
  onCompletedUpload={(result) => ...}
  TriggerComponent={MyTriggerButton}
/>
```

**Where it's actually used**: exactly one place —
`oarepo_ui/forms/components/FilesField/FilesFieldWrappers.jsx` wraps it as
`FileUploadWrapper`/`FileEditWrapper`, consumed by `FilesFieldButtons.jsx`'s
per-file `EditFileButton`/`UploadFileButton`. **Neither CCMM,
`oarepo_rdm_ui`, `oarepo_requests`, nor `oarepo_dashboard` import it** — the
main deposit-form "Files" tab in all of those still uses base RDM's
`UppyUploader` for the bulk multi-file flow (see the sibling skill's
`file-upload.md`). Reach for `@oarepo/file-manager` specifically when you
need a single-file add/edit dialog with per-file metadata and optional
PDF-image extraction, outside the main bulk-upload step — not as a
replacement for `UppyUploader`.
