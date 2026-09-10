# Worked example: building a model on `oarepo_ui`/`oarepo_rdm_ui` (CCMM)

Read this when building a new model's deposit-form sections from scratch —
it's the best concrete template to imitate. CCMM (`ccmm_invenio`) is a real
metadata profile (used by Czech National Repository Platform repositories)
built entirely on the mechanisms in [forms.md](forms.md); this file shows
what that looks like in practice, including one genuinely elaborate custom
field.

## The section-object pattern in practice

Every CCMM field group is a plain object, assembled into one array
(`forms/CCMMSections.js`) that becomes the model's `sections` prop:

```js
export const CCMMFunding = {
  key: "funding",
  label: i18next.t("Funding"),
  component: (tabConfig) => {
    const { overridableIdPrefix } = tabConfig.formConfig;
    return (
      <Overridable id={buildUID(overridableIdPrefix, "Funding")} {...tabConfig}>
        <FundingField fieldPath="metadata.funding" />
      </Overridable>
    );
  },
  includesPaths: ["metadata.funding"],
};

export const CCMMSections = [
  CCMMCommunityAndAccess, CCMMFiles, CCMMGeneralInformation,
  CCMMFunding, CCMMAlternativeIdentifiers, CCMMRelatedWorks,
];
```

`forms/index.js` (the model's actual webpack entry) pairs this with a
serializer and hands both to the shared shell — see
[forms.md §1](forms.md#1-one-generic-shell-model-supplied-everything-else)
for the full entry-point code.

## Three-level override chain

Every section wraps its content in `Overridable`, which means the same
field can be overridden at up to three layers, and it matters which one you
actually need to touch:

1. **Base `invenio_rdm_records` component** — e.g.
   `InvenioRdmRecords.DepositForm.DatesField.DateField` (documented in the
   sibling skill).
2. **An `oarepo_rdm_ui` preset section** — reusable wholesale or
   overridden by a specific model.
3. **The model's own section** (e.g. CCMM's) — itself wrapped in
   `Overridable id={buildUID(overridableIdPrefix, "...")}`, so a specific
   *instance* of a CCMM-based repository can override it again without
   forking CCMM.

Before writing a new override, identify which of these three layers
actually needs changing — overriding layer 3 when the real customization
belongs at layer 1 (or vice versa) means your change either doesn't apply
broadly enough or fights the model's own intended customization point.

## Illustrative field 1 — mixing building blocks from three sources

`CCMMGeneralInformation` composes a tab from base RDM fields, an
`oarepo_rdm_ui` component, and an `oarepo_ui` field, side by side:

- `PIDFieldList` from `@js/oarepo_rdm/form/components` (`oarepo_rdm_ui`).
- `TitlesField`/`ResourceTypeField`/`PublisherField`/etc. straight from
  `@js/invenio_rdm_records` (base skill).
- `EDTFSingleDatePicker`/`CreatibutorsField` from `@js/oarepo_ui/forms`.

This is the template for "assemble a tab from mixed-provenance building
blocks, each independently overridable" — don't assume all the pieces in
one section have to come from the same package.

## Illustrative field 2 — extending, not reimplementing, a preset

`CCMMCommunityAndAccess` shares the same base
(`CommunityHeader`/`AccessRightField`) as `oarepo_rdm_ui`'s
`RDMCommunityAndAccess` preset, but *extends* it rather than rewriting it:
adds `saveOnTabChange: true`, a custom `sectionCompletion` function blending
Formik-field completion with community-selection state, and swaps in
`apiConfigs`/`overriddenComponents` props to customize the community
picker's search API and rendering. **Pattern**: start from the closest
`oarepo_rdm_ui` preset section, then layer your model's specific behavior
on top via props, rather than copy-pasting and modifying the whole section.

## Illustrative field 3 — a fully custom field (reorder + bulk import)

`RelatedResourceField/` is the most elaborate custom field in CCMM,
capturing related-resource citations/DOIs/relation-types. Worth studying in
full as a template for any similarly complex repeatable field:

- **Reorderable list**: Formik's `FieldArray` + `react-dnd`
  (`DndProvider`/`HTML5Backend`) for drag-reorder, with each row
  (`RelatedResourceFieldItem`) opened for editing in a `RelatedResourceModal`.
- **Bulk "Load from DOI" import** (`LoadFromDoiModal.jsx`): parses pasted
  DOI URLs/text (`extractDois`, capped at `MAX_DOIS_PER_BATCH`), POSTs each
  to a **CCMM-specific backend endpoint** (`/api/related-records`) via
  `@tanstack/react-query`'s `useMutation`, runs results through the same
  `SchemaField` deserializer the top-level form uses (documented in-code as
  necessary to avoid a save-merge bug — reuse the form's own deserializer
  for any field that injects data outside the normal Formik `onChange`
  path), and reports per-DOI success/failure/duplicate state in the UI.
- **Reaching into the surrounding form from inside a field**:
  `useFormConfig()` and `useDepositFormAction()` (from `@js/oarepo_ui/forms`)
  let a field component read global form config (e.g. vocabularies) and
  trigger a Redux `save()` action — the general hook-based bridge between a
  leaf field and the surrounding Formik/Redux deposit form. Use these
  instead of prop-drilling form-level state/actions into a deeply nested
  field.

## The Files tab: reuse, then add one guard

`CCMMFiles.js` is near-identical to `oarepo_rdm_ui`'s `RDMFiles` preset
section (both wrap base RDM's `UppyUploader`, documented in the sibling
skill), plus one addition: a `lockTabChange` guard blocking tab navigation
while any file is mid-upload (checked against Redux `state.files.entries`
status), and a `sectionCompletion` via `oarepo_ui`'s
`computeFilesSectionCompletion`. This is the minimal-diff pattern for
"reuse a preset section almost as-is, add one behavioral guard."
