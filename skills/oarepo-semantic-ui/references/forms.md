# OARepo deposit forms — the `DepositFormApp` shell and "sections"

Read this file when writing or editing a deposit-form section for any
OARepo-based model, or when deciding how much of base Invenio's deposit
form machinery to reuse. Assumes you already know react-invenio-forms'
field components and Formik basics (sibling `invenio-semantic-ui` skill) —
this file covers only the OARepo layer on top. For the Python config class,
webpack entry, `pyproject.toml` entry-points, and template files that
actually get a form entry point running end-to-end, see
[wiring.md](wiring.md) — this file covers the React-side shell only.

## 1. One generic shell, model-supplied everything else

Base `invenio_app_rdm` has exactly one `RDMDepositForm` component and one
webpack bundle. OARepo instead ships a **model-agnostic `DepositFormApp`**
(`oarepo_ui/forms/`) that every model instantiates itself, from its own tiny
webpack entry, supplying model-specific data as plain props:

```js
// a model's forms/index.js — the entire per-model wiring
import { DepositFormApp, parseFormAppConfig } from "@js/oarepo_ui/forms";
import { CCMMDepositRecordSerializer, CCMMSections } from "@js/ccmm_invenio/forms";

const { rootEl, config, ...rest } = parseFormAppConfig();
const recordSerializer = new CCMMDepositRecordSerializer(
  config.default_locale, config.custom_fields.vocabularies
);
ReactDOM.render(
  <DepositFormApp
    config={config} {...rest}
    sections={CCMMSections}
    recordSerializer={recordSerializer}
    componentOverrides={componentOverrides}
    useWizardForm
  />,
  rootEl
);
```

`DepositFormApp` accepts `sections`, `recordSerializer`, `apiClient`,
`fileApiClient`, `draftsService`, `filesService`, `depositService`,
`depositReducer`, `filesReducer`, `configureStore`, and `componentOverrides`
as **constructor-injectable props**, all defaulting to base
`invenio_rdm_records`' own implementations (`RDMDepositApiClient`,
`RDMDepositFileApiClient`, `RDMDepositRecordSerializer`, ...) when a model
doesn't supply its own. This is the injection point that lets radically
different models (different serializer, different Redux reducer, different
API client) share one component tree instead of each forking
`RDMDepositForm`.

`parseFormAppConfig()` reads the same hidden-input DOM convention as base
Invenio (`getInputFromDOM("deposits-record")` etc. — see
[conventions.md](conventions.md#the-jinjax-to-react-bridge)); the bridge
itself is unchanged, only the shell that consumes the parsed config differs.

## 2. "Sections" — the atomic authoring unit

A section is a plain object, not a component:

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
```

Models assemble a flat array of these and pass it as `sections`:

```js
export const CCMMSections = [
  CCMMCommunityAndAccess, CCMMFiles, CCMMGeneralInformation,
  CCMMFunding, CCMMAlternativeIdentifiers, CCMMRelatedWorks,
];
```

- `key` — a stable identifier for tab/step navigation.
- `label` — the tab/step title.
- `component(tabConfig)` — a function (not a component instance) receiving
  the tab's config (including `formConfig.overridableIdPrefix`) and
  returning the section's JSX. Always wrap the returned JSX in
  `<Overridable id={buildUID(overridableIdPrefix, "YourKey")}>` (see
  principle 3 in [../SKILL.md](../SKILL.md)) so the section itself can be
  re-overridden by an instance-specific repository.
- `includesPaths` — field paths "owned" by this section, used the same way
  as base Invenio's `AccordionField` `includesPaths` (error badge
  aggregation), but now also driving the tab-level completion percentage
  (§4) and section-scoped error routing.

`oarepo_ui/forms/exampleSection.js` is the literal placeholder shown when a
model hasn't defined any sections yet — its scaffold text is the canonical
"how to write a section" reference if you're starting from nothing.

## 3. Ready-made section presets (`oarepo_rdm_ui`)

`oarepo_rdm_ui` is **not** a runtime multi-model dispatcher — despite the
name, it doesn't let one page serve several models simultaneously. It's a
**starter-kit library of prebuilt section arrays** at three completeness
tiers, meant to be imported wholesale or cherry-picked from:

```js
export const RDMMinimalSections  = [RDMCommunityAndAccess, RDMFiles, ExampleSection];
export const RDMBasicSections    = [RDMCommunityAndAccess, RDMFiles, RDMGeneralInformationBasic];
export const RDMCompleteSections = [
  RDMCommunityAndAccess, RDMFiles, RDMGeneralInformationComplete, RDMFunding, RDMAlternativeIdentifiers,
];
```

"Basic" vs "Complete" is a field-count tier (Basic: PID/Title/ResourceType/
PublicationDate/Creators only; Complete: additionally Descriptions/
Contributors/Publisher/Version/Languages/Subjects/Rights/Dates) — not
different data models. A model wanting a standard RDM-shaped form imports
one of these three arrays directly; a model wanting bespoke fields (like
CCMM) writes its own array, reusing individual exports from
`oarepo_rdm_ui/form/{sections,components}` (e.g. `PIDFieldList`) only where
useful — see [worked-example-ccmm.md](worked-example-ccmm.md) for exactly
that pattern. `RDMMinimalRecordSerializer` is a paired, deliberately-blank
serializer subclass (`extends RDMDepositRecordSerializer` with an empty
`depositRecordSchema` getter) for the minimal tier.

**Known dead code**: `oarepo_rdm_ui/index.js` re-exports `./detail`, but no
`detail/` directory exists in this package — a stale export left over from
a refactor. Don't go looking for a "detail" module here.

## 4. The tab/wizard layer

Base RDM's deposit form is a single scrolling page of `AccordionField`
sections. OARepo adds an entire tab/wizard navigation layer on top of the
same "sections" array, present in `oarepo_ui/forms/` and absent from base
Invenio:

- `TabForm` / `WizardFormLayout` / `FormSteps` / `FormTabs` / `TabContent` —
  render the sections as either a flat tab bar or a step-by-step wizard
  (`useWizardForm` prop on `DepositFormApp`), with URL `?tab=` sync.
- `lockTabChange` — a section can block navigating away (e.g. CCMM's Files
  section blocks tab changes while any file is mid-upload, checked against
  Redux `state.files.entries` status).
- `saveOnTabChange` — a section can trigger an autosave when the user
  switches away from it (CCMM's Community-and-Access section uses this).
- `SectionCompletionBar` — a per-section completion percentage, computed
  from a `sectionCompletion` function a section can supply (blending
  Formik-field completion with arbitrary custom logic — e.g. CCMM blends
  field completion with community-selection state).
- `FormTabErrors` — aggregates errors per section (via `includesPaths`,
  through `findSectionIndexForFieldPath`/`getSubfieldErrors` in
  `forms/util.js`) to badge tabs with error counts, the tab-level analog of
  base `AccordionField`'s section error badges.

## 5. Schema-driven behavior: labels/help/required only, not widget choice

**There is no generic "pick a widget for this JSON-Schema type" component**
anywhere in `oarepo_ui/forms`. Every field in a section is a manually
chosen, ordinary react-invenio-forms/invenio_rdm_records/oarepo_ui field
component — auto-generation claims refer to a narrower mechanism:

```js
// forms/contexts.js / forms/util.js
const fieldData = getFieldData({ fieldPath, icon, ... }); // reads config.ui_model
const props = mergeFieldData(explicitProps, fieldData);   // explicit props win
```

`getFieldData(fieldPath)` looks up `label`/`help`/`hint`/`required`/`detail`
for that path out of `config.ui_model` — a UI-model tree built **server-side**
from the model's JSON Schema/UI-schema (Python:
`RecordsUIResourceConfig`/`resource.py` sets `form_config["ui_model"]`).
Every OARepo field wrapper (`TextField`, `StringArrayField`,
`MultilingualTextInput`, `I18nTextInputField`) calls this and merges it with
whatever you pass explicitly. **What auto-populates**: label text, help
text, placeholder, required-ness. **What doesn't auto-populate**: which
component to render — that's still a manual choice baked into the section
source. Don't assume changing a model's JSON Schema alone will change which
widget appears; you still edit the section file for that.

## 6. OARepo-specific field components

Beyond what react-invenio-forms/invenio_rdm_records already provide
(`oarepo_ui/forms/components/`):

- **Multilingual string convention** — not present in base RDM at all.
  `MultilingualTextInput` wraps react-invenio-forms' `ArrayField`; each row
  is a `LanguageSelectField` plus either `I18nTextInputField` or
  `I18nRichInputField` (rich text via `OarepoRichEditor`). `I18nString` is
  the read-only counterpart. Models a JSON shape `[{lang, value}, ...]` for
  "translatable string" fields.
- **`StringArrayField`** — a convenience add/remove text-array widget on
  top of Formik's `FieldArray`, going through `getFieldData()`. react-invenio-forms'
  own `ArrayField` is lower-level/render-prop-based; this is the "just give
  me a list of strings" shortcut OARepo adds.
- **`ArrayFieldItem`** — a thin standard wrapper around react-invenio-forms'
  `GroupField` giving a consistent remove-button/highlight UX; reused by
  `StringArrayField` and `MultilingualTextInput`.
- **`AccessRightField`, `FundingField`, `CreatibutorsField`,
  `IdentifiersField`, `EDTFDatePickerField/*`, `FilesField/*`** are mostly
  thin adaptation shims re-exporting the equivalent `@js/invenio_rdm_records`
  components (e.g. wrapped in a `Field` + `I18nextProvider`), not
  independent reimplementations — read the base skill's field docs for
  their real behavior.
- Custom-fields themselves are **not** reimplemented — Python's
  `RecordsUIResourceConfig.custom_fields()` still assembles the standard
  `invenio-records-resources` custom-fields UI config, so on the JS side
  it flows through the same `CustomFields` component documented in the
  sibling skill.

## 7. API/serializer layer — thin, no new base class

`oarepo_ui/api/` contains no OARepo API-client abstraction at all — just
two serializer subclasses (`api/recordSerializer.js`):

- **`OARepoDepositSerializer extends DepositRecordSerializer`** — adds
  `removeEmptyValues`, `removeKeysFromNestedObjects` (strips
  react-invenio-forms' internal `__key`), `removeNullAndInternalFields`,
  and a `deserializeErrors` that accepts the richer `{message, severity,
  description}` shape alongside plain strings.
- **`EmptyDepositRecordSerializer extends RDMDepositRecordSerializer`** —
  `_pick`s only a fixed whitelist of top-level keys, for models whose
  record shape diverges from RDM's.

`DepositFormApp` imports and instantiates base RDM's HTTP/service classes
directly (`RDMDepositApiClient`, `RDMDepositFileApiClient`,
`RDMDepositDraftsService`, `RDMDepositFilesService`, `DepositService`,
`DepositBootstrap`), just parameterized with a model-specific `createUrl`
built server-side from the model registry (not hardcoded to the RDM records
endpoint) — see §1's injectable-props list. There is no `withCancel`-style
wrapper layer beyond what react-invenio-forms already provides; ad hoc axios
calls (e.g. file-import in `FilesField`) use OARepo's own bare
`httpApplicationJson`/`httpVnd` instances from `oarepo_ui/util.js` — see
[conventions.md](conventions.md#utiljs-grab-bag).
