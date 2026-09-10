# The async-action + confirmation-modal idiom

Read this before building a button that performs an async API action,
optionally confirms first, shows loading/error state, and notifies on
success — accepting/declining a request, deleting a resource, restoring a
record, changing a role, etc. There is **no single shared component** for
this across the ecosystem; instead, the same architecture is hand-rolled
three times with variations. Imitate the shape below rather than
searching for one canonical "AsyncActionButton" export.

## The recurring shape

- A **Controller** component (often a class, holding the state) owns
  `{ modalOpen, loading, error }` and a `performAction(...)` method, and
  exposes all of it via a React context.
- A **Trigger**/**Button** component reads `loading`/`modalOpen` from that
  context and opens the modal on click.
- A **Modal** component renders the confirmation body plus Cancel/Confirm,
  shows `error` inline, and disables/spins the confirm button while
  `loading`.
- On success: close the modal, call a `successCallback` (typically a search
  results refresh via react-searchkit's `updateQueryState`, see
  [bulk-actions.md](bulk-actions.md)), and — in an admin-embedded context —
  call `addNotification(...)` (see
  [reusable-widgets.md §1](reusable-widgets.md#1-notificationcontext--the-global-toast-system)).
- Every layer (Button, Modal, ModalBody) is typically `Overridable`-wrapped,
  so a downstream package can replace one piece without rebuilding the
  whole flow.

## Three real implementations, in increasing generality

**1. `invenio_administration`'s Delete flow** (single, fixed action) —
`src/actions/DeleteModalTrigger.js` + `DeleteModal.js`. `DeleteModalTrigger`
owns `modalOpen` state and renders both the trigger button and the modal.
`DeleteModal.handleOnButtonClick` calls the delete API, then
`addNotification({title, content, type: "success"})` and `successCallback()`
on success, or sets local `error` state rendered via `<ErrorMessage>` on
failure. Use this shape when there's exactly one destructive action with no
variation.

**2. `invenio_administration`'s generic `ResourceActions`/`ActionModal`**
(configurable actions) — `src/actions/ResourceActions.js` + `ActionModal.js`.
`ActionModal` is just an `Overridable`-wrapped `Modal` shell.
`ResourceActions` decides per action whether to open a modal with a
generated form (if the action config has a `payload_schema`) or fire the
action directly with no confirmation. **Each action can plug an
`Overridable` slot** `InvenioAdministration.ResourceActions.ModalBody.<actionKey>`
to fully replace just the modal body. This is the extension point
`invenio_communities` uses to add a "restore this community" confirmation
(`administration/components/RestoreConfirmation.js`) **without building a
new trigger/modal pair at all** — it imports `NotificationContext` directly
from `invenio_administration` and plugs into the existing modal system.
**Prefer this over building a new action UI whenever you're adding an
action to an existing admin resource.**

**3. `invenio_requests`'s `RequestActionController`** (multiple actions on
one resource) — `request/actions/RequestActionController.js` +
`RequestAction.js` + `RequestActionModal.js` + `RequestActionModalTrigger.js` +
`RequestActionButton.js`. One controller, mounted once per request, provides
`RequestActionContext` covering **all** of that request's possible actions
(accept/decline/cancel/...), derived from the request's own
`links.actions`: `{ modalOpen: {actionId: bool}, toggleModal(actionId, val), performAction(action, commentContent), loading, error }`.
Each action gets its own `RequestAction` (comment box + modal) built from
the shared, `Overridable`-wrapped `RequestActionButton`/`RequestActionModal`/
`RequestActionModalTrigger` — so a downstream package can override just one
action's button or modal without touching the controller. **Use this shape
for a brand-new feature where one resource can have several distinct
actions** — it's the most reusable and cleanly generalizes past a single
delete/restore action.

## Simpler variant for a single, low-stakes confirm

`invenio_requests/components/modals/BaseModal.js` + `DeleteConfirmationModal.js`
is a small, generic confirm modal taking just
`{contentText, action, isLoading, error, headerText, cancelButtonText, actionButtonText}`
— no context/controller layer at all. Reach for this instead of the
Controller-context architecture above when there's exactly one action, used
in exactly one place, with no need for other components to trigger the same
modal.

## Escalating to a "type to confirm" danger-zone pattern

For a highly destructive action (deleting a resource that cascades to other
data), a plain confirm modal isn't enough — see the two-tier pattern in
[reusable-widgets.md §6](reusable-widgets.md#6-danger-zone--type-to-confirm-delete-pattern):
a simple confirm button for low-risk deletes, escalating to a modal that
fetches and shows the live impact (record/member counts) and requires typing
the resource's exact name/slug before the destructive button un-disables.
