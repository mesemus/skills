# Catalog of smaller reusable components

Small, generic (non-business-specific) components and idioms found across
`invenio_administration`, `invenio_requests`, `invenio_communities`,
`invenio_app_rdm`, `invenio_theme`, and `invenio_collections`, each compact
enough not to warrant its own file but worth knowing exist before rebuilding
the same thing. For the larger, dedicated subsystems, see
[file-upload.md](file-upload.md), [bulk-actions.md](bulk-actions.md), and
[action-modals.md](action-modals.md).

## 1. `NotificationContext` — the global toast system

Full API (`invenio_administration/src/ui_messages/{context.js,messages.js}`):

```js
export const NotificationContext = React.createContext({
  notifications: {}, addNotification: () => {}, removeNotification: () => {},
});
```

`addNotification({ title, content, type })` — `type: "success"` renders a
`SuccessMessage` that **auto-dismisses after 5 seconds**; anything else
renders an `ErrorMessage` that **persists until the user dismisses it**.
Notification IDs are auto-incrementing integers assigned internally, not by
the caller. To get this for free in a new page/app: wrap the app root in
`<NotificationController>` (it renders both the notification container div
and `children`), then anywhere below, read `NotificationContext` (class:
`static contextType = NotificationContext`; function: `useContext`) and call
`this.context.addNotification({...})`.

**Reuse status**: this is a real, importable, drop-in system — but in
practice only `invenio_administration` and code embedded inside it (e.g.
`invenio_communities`'s admin-surface `RestoreConfirmation.js`, which does
`import { NotificationContext } from "@js/invenio_administration"`) actually
use it. Everywhere else in `invenio_communities` and all of `invenio_requests`,
components instead keep local `{ loading, error }` state and render an
inline `ErrorMessage` directly in the modal/form — no global toast, no
auto-dismiss. If you're adding a feature outside the admin app, either
explicitly import `NotificationContext` from `invenio_administration` (it
has no admin-specific coupling) or follow the simpler local-error-state
convention already used in that package — don't assume a global toast is
available by default.

## 2. `Loader`, `ErrorPage`, `ErrorBoundary`

Two near-identical `Loader` components exist
(`invenio_administration/src/components/Loader.js`,
`invenio_requests/components/Loader.js`) — a trivial
`isLoading ? <spinner> : children` wrapper, `Overridable`-registered as
`"Loader"`/`"Admin.Loader.layout"`. `ErrorPage`
(`invenio_administration/src/components/ErrorPage.js`) is a generic
full-page error state (`errorCode`, `errorMessage`, boolean `error` flag,
else renders `children`). `ErrorBoundary`
(`invenio_requests/components/ErrorBoundary.js`) is a real React error
boundary (`componentDidCatch`) wrapping a generic error display, registered
as `"ErrorBoundary"`. Reuse these three directly as the loading/error/crash
scaffolding for any new top-level React root, rather than reinventing the
same three states.

## 3. `SuccessIcon` + `ErrorPopup` — inline per-row async feedback

`invenio_communities/members/components/{SuccessIcon.js,ErrorPopup.js}` —
`SuccessIcon` shows a green checkmark for a configurable `timeOutDelay`
then auto-hides; `ErrorPopup` is a `Popup` bound to an error string, shown
next to a control. Combined in `ActionDropdown.js` (§4) to give **inline,
per-row** feedback for an async save — the idiom to reach for when a global
toast (§1) would be overkill for a small, in-place edit (e.g. a dropdown in
one table row), as opposed to a page-level action.

## 4. `ActionDropdown` — generic inline async-select

`invenio_communities/members/components/{ActionDropdown.js,dropdowns.js}`
(`RoleDropdown`/`VisibilityDropdown` are built on it). Fully generic: a
`Dropdown` whose `onChange` fires an async `action(resource, value)`, shows
`loading` on the dropdown itself, and shows `SuccessIcon`/`ErrorPopup` (§3)
next to it. Takes `options`, `action`, `successCallback`,
`optionsSerializer` as props — no business logic baked in. This is the
smallest reusable building block for "a dropdown that saves immediately on
change," reusable for any single-field inline edit in a table row.

## 5. Context-provider-wraps-API-client pattern

`invenio_communities/api/{invitations/InvitationsContextProvider.js, members/MembersContextProvider.js, membershipRequests/MembershipRequestsContextProvider.js}`
all follow the same idiom: construct one API client instance in the
provider's constructor from a resource prop (e.g. the current `community`),
expose it as `{ api: this.apiClient }` via context, and have children read
it via `static contextType`. Named pattern worth reusing: **construct an API
client once at the search-app/feature root and put it in a context**,
rather than prop-drilling a client instance through several levels of
search-result/row components.

## 6. Danger-zone / "type to confirm" delete pattern

`invenio_communities/settings/profile/DangerZone.js` gates rename/delete
behind `permissions.can_rename`/`can_delete` inside a `Segment
className="negative"` section. Two escalating tiers exist:

- **`DeleteButton.js`** — a simple confirm-then-delete button, for low-risk
  deletes (e.g. removing a profile picture).
- **`DeleteCommunityModal.js`** — for a highly destructive, cascading
  delete: fetches live counts of affected records/members when the modal
  opens, requires checking several acknowledgement checkboxes **and** typing
  the exact resource slug before the destructive button un-disables.

Use the simple tier by default; escalate to the typed-confirmation tier only
when the delete has real, non-obvious blast radius the user should see
before confirming.

## 7. `LogoUploader` — single-image upload widget

`invenio_communities/settings/profile/LogoUploader.js` — a complete
`react-dropzone` + `Image` (with a cache-busting query-param trick on
re-upload) + delete-button pattern for "upload/replace/delete one image via
drag-drop or click," decoupled from community specifics except for the API
calls it makes. Good template for any single-image-upload widget (avatars,
logos, thumbnails) — don't rebuild drag-and-drop-plus-preview from scratch
for a single image when this pattern already exists.

## 8. `AppMedia`/`Media`/`MediaContextProvider` — responsive branching

`invenio_theme/Media.js` wraps `@artsy/fresnel`'s `createMedia`:

```js
export const breakpoints = { void: 0, mobile: 320, tablet: 768, computer: 1280, largeScreen: 1680, widescreen: 1920 };
export const AppMedia = createMedia({ breakpoints });
```

Standard usage (`invenio_requests/components/ModalTriggers.js`), for
rendering full buttons on wider screens and collapsing the same actions into
a dropdown on mobile:

```jsx
const { MediaContextProvider, Media } = AppMedia;
<MediaContextProvider>
  <Media greaterThanOrEqual="tablet"><RequestDeclineButton {...props} /></Media>
  <Media at="mobile"><Dropdown.Item icon="cancel" content={i18next.t("Decline")} /></Media>
</MediaContextProvider>
```

This is the standard idiom for "collapse a row of action buttons into a
dropdown on narrow screens" — reach for it instead of hand-rolling CSS media
queries for responsive action bars.

## 9. `CopyButton` and the "format-switchable content" pattern

`invenio_app_rdm/components/CopyButton.js` — a copy-to-clipboard button with
a `Popup` confirmation ("Copied!") that auto-dismisses after 1.5s; supports
copying literal `text` or fetching content from a `url` first. Combined with
a format-selector dropdown, it forms a recurring "render content in format
X, let the user switch X, let them copy it" pattern, seen in
`landing_page/RecordCitationField.js` (citation style dropdown, debounced
re-fetch, loading skeleton, MathJax retypeset) and `landing_page/ExportDropdown.js`
(export format dropdown + download + copy). Reuse `CopyButton` directly, and
imitate this dropdown-plus-copy composition for any "same content, several
representations" feature.

## 10. `ModalContext`/`ModalContextProvider` — one shared modal for a list

`invenio_communities/members/components/modal_manager/` — a generic
single-active-modal manager
(`{modalOpen, modalMode, modalAction, member}` +
`openModal({modalAction, modalMode, member})`/`closeModal()`), letting any
row in a list trigger the *same* shared modal instance instead of every row
mounting its own modal. Use this for long lists where mounting N modals (one
per row) would be wasteful — mount one modal at the list root and have rows
call `openModal(...)` with the row's data.

## 11. Recursive, self-nesting tree items

`invenio_collections/collections/components/NestedCollectionItem.js` — a
`React.memo`-wrapped, self-recursive component rendering a title, a count
label, and an action-menu dropdown, then recursing into `children` with an
incrementing `nestingLevel` for indentation:

```jsx
const NestedCollectionItem = memo(({ collection, allCollections, nestingLevel = 1, ...rest }) => (
  <div className="nested-collection-item" data-nesting-level={nestingLevel}>
    {/* header + action Dropdown */}
    {collection.children?.map((childSlug) => (
      <NestedCollectionItem
        key={childSlug}
        collection={allCollections[childSlug]}
        nestingLevel={nestingLevel + 1}
        {...rest}
      />
    ))}
  </div>
));
```

Paired with `components/ReorderableList.js` (a `Dimmer`/`Loader` overlay
showing "Saving order..." while a reorder request is in flight). This is a
generic template for **any** hierarchical management UI (categories,
nested groups, org charts) — not specific to collections — combined with a
reusable "saving, please wait" overlay for drag-reorder-then-persist
interactions.
