# File upload & management (deposit forms)

Read this when a form needs to let users upload/manage files (not just
metadata) — e.g. a deposit form's file attachment step, or any new feature
needing drag-and-drop upload with progress and quota limits. This is a
substantial, largely self-contained subsystem living in
`invenio_rdm_records`'s deposit code, independent from the metadata field
patterns in [forms.md](forms.md) and `references/react-invenio-forms/`:

- [react-invenio-forms/components.md](react-invenio-forms/components.md) —
  the metadata field components this subsystem is *not* built from.
- [react-invenio-forms/ui-primitives.md](react-invenio-forms/ui-primitives.md) —
  `FilesList`, the read-only/compact file display this subsystem contrasts
  with (see §"Read-only file display" below).

## Two parallel, swappable implementations

Both consume the **same Redux `files` slice** and expose the **same prop
surface** (quota, permissions, `filesLocked`, `importParentFiles`, ...), so a
new repository can pick either as a starting template, or reuse just the
Redux contract with a fully custom upload UI.

- **`FileUploader`** (`react-dropzone`-based) — the simpler, drag-and-drop
  implementation. No chunking/resumability; the whole file is uploaded in
  one request.
- **`UppyUploader`** — a full Uppy (`@uppy/react`, `@uppy/aws-s3-multipart`)
  integration for chunked, resumable, S3-multipart uploads. Needed for large
  files or unreliable connections; requires backend support for multipart
  transfers (`transfersConfig.transferType`).

Choose `UppyUploader` when large files or resumable uploads matter and the
backend supports S3-multipart transfers; choose `FileUploader` for a simpler
setup with no multipart requirement.

## The shared Redux contract

`src/deposit/state/reducers/files.js` / `state/actions/files.js` define an
`UploadState` enum (`isPending`, `isUploading`, `isFinished`, `isFailed`) and
the thunks both uploaders dispatch into: `uploadFiles`, `deleteFile`,
`importParentFiles`, plus the multipart-specific `initializeFileUpload`,
`uploadPart`, `finalizeUpload`, `setUploadProgress`, `saveAndFetchDraft`.
**This is the contract you must replicate if reusing either uploader's UI
outside `invenio_rdm_records`**, or the contract to implement from scratch
if building a custom upload UI that still wants to reuse `FilesListTable`
(below).

`utils.js` exports `getFilesList(files)`, normalizing the Redux files-entries
map into `{filesList, filesNamesSet, filesSize}` — the shape every
consuming component works from rather than reading the raw entries map
directly.

## `FileUploader` (drag-and-drop)

`src/deposit/fields/FileUploader/`:

- **`FileUploaderComponent`** owns the business rules: max-files/max-storage
  quota checks, duplicate-filename detection, empty-file rejection, a
  "metadata-only record" toggle, an "import files from previous version"
  banner, and a **locked-files state** for published records (swaps in
  `EditFilesAccordion` for the unlock-and-modify flow instead of a normal
  dropzone).

  ```js
  const dropzoneParams = {
    onDropAccepted: (acceptedFiles) => {
      const maxFileNumberReached = filesList.length + acceptedFiles.length > quota.maxFiles;
      const acceptedFilesSize = acceptedFiles.reduce((t, f) => t + f.size, 0);
      const maxFileStorageReached = filesSize + acceptedFilesSize > quota.maxStorage;
      const { duplicateFiles, emptyFiles, nonEmptyFiles } = /* bucket by name-collision / zero-size */;
      // shows Message warnings, then uploadFiles(formikDraft, filesToUpload)
    },
    multiple: true, noClick: true, noKeyboard: true,
  };
  ```

- **`FileUploaderArea`** — the actual dropzone target plus **`FilesListTable`**
  (per-file row: default-preview radio, filename + checksum, human-readable
  size, progress bar, cancel/delete icon) and **`FileUploadBox`** (the
  drag-and-drop CTA). `FilesListTable` is reused verbatim by `UppyUploader`.
- **`FileUploaderToolbar`** — "Metadata-only record" checkbox plus
  storage/file-count usage labels and a "Manage storage" button.
- **`QuotaManager`/`QuotaDisplay`** — a self-contained "increase your
  storage quota" panel: fetches a `quota_increase` link, POSTs
  `{quota_size}`, renders a bar chart of default/additional/remaining
  storage. Reusable as-is for any "request more of resource X" UI, not just
  file storage.
- **`FileModification`/`ModificationModal`/`EditFilesAccordion`** — the
  "unlock published files for correction" flow: a button opens a modal
  requiring justification before re-enabling file edits on an already
  published record.

Every visual section is wrapped in its own `Overridable` id under
`InvenioRdmRecords.DepositForm.FileUploader.*` — that's the extension point
for a downstream repo customizing one part of the uploader without forking
the whole component (see
[conventions.md §1](conventions.md#1-overridable-components--the-central-customization-mechanism)
for the general mechanism).

## `UppyUploader` (chunked/resumable)

`src/deposit/fields/UppyUploader/`:

- **`UppyUploader.js`** renders Uppy's `<Dashboard>` bound to a memoized
  `Uppy` instance, reusing `FilesListTable`/`FileUploaderToolbar` from the
  `FileUploader` package. Configures `restrictions`
  (`maxNumberOfFiles`/`maxTotalFileSize`/`minFileSize`), an i18n bridge
  (`useUppyLocale`, syncs Uppy's own locale strings with `i18next`), an
  `ImageEditor` plugin, and duplicate-file rejection via `onBeforeFileAdded`.
- **`RDMUppyUploaderPlugin`** — a custom subclass of `@uppy/aws-s3-multipart`'s
  `AwsS3Multipart` plugin that redirects Uppy's part-upload/completion
  lifecycle callbacks into the Redux thunks above, computes an MD5 checksum
  per part (via `hash-wasm`) for integrity checking, and currently disables
  Uppy's native resumable "list parts" capability (not yet supported
  server-side). This is the concrete pattern to imitate for **bridging any
  third-party upload library's plugin lifecycle to a Redux-backed upload
  state machine**:

  ```js
  const [uppy] = useState(() =>
    new Uppy({ debug: false, autoProceed: false, restrictions, locale })
      .use(RDMUppyUploaderPlugin, {
        limit: fileUploadConcurrency, transferType: transfersConfig.transferType,
        isTransferSupported, quota,
        initializeFileUpload, finalizeUpload, saveAndFetchDraft, setUploadProgress, uploadPart,
        abortUpload: (file) => deleteFile(file),
        checkPartIntegrity: true,
      })
      .use(ImageEditor)
  );
  ```
- **`error.js`** — typed errors (`FileSizeError`, `InvalidPartNumberError`,
  `SignedUrlExpiredError`) surfaced through Uppy's own error UI.

## Read-only file display (not the uploader)

For a **compact, read-only** list of already-uploaded files (e.g. an
attachment list on a request/comment, not an editable deposit step), use
`react-invenio-forms`'s much smaller `FilesList` component instead of either
uploader — see
[react-invenio-forms/ui-primitives.md](react-invenio-forms/ui-primitives.md#fileslist).
