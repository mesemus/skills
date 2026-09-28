# oarepo-model's builder architecture

**Scope:** this document describes `oarepo_model`'s own internals - the preset/customization/datatype
framework. It applies to the `oarepo-model` repository itself, and equally to *any other* package that
imports `oarepo_model` to assemble a dynamic Invenio record model (e.g. `oarepo-rdm` builds its RDM record
model this way) - if a repo has `Preset`, `Customization`, or `DataType` subclasses, or calls
`oarepo_model.api.model(...)`, this document's contract applies to it even though the class definitions live
in a different package. A repo with none of those (a plain Flask extension - see
[architecture.md](architecture.md)) has no use for anything below.

A model is not a static class hierarchy - `oarepo_model.api.model(...)` runs a list of **presets** against
an `InvenioModelBuilder`, each preset yielding **customizations**, each customization mutating one or more
**partials** (named, typed, lazily-built pieces of the eventual model: classes, lists, dicts, modules, files).
Everything below exists to make that process orderable, debuggable, and safe for third-party presets to
extend without stepping on each other.

## Partials and the builder

`InvenioModelBuilder` (`src/oarepo_model/builder.py`) owns a `dict[str, Partial]`. Concrete partial kinds:

| Partial | Builder methods | Built value |
|---|---|---|
| `BuilderClass` | `add_class`, `get_class` | a dynamically created `type` |
| `BuilderClassList` | `add_class_list`, `get_class_list` | `list[type]` |
| `BuilderList` | `add_list`, `get_list` | plain `list` |
| `BuilderDict` | `add_dictionary`, `get_dictionary` | plain `dict` |
| `BuilderModule` | `add_module`, `get_module` | a fake module (`SimpleNamespace`-like) with callables/attrs |
| `BuilderFile` / `BuilderSymbolicLink` | `add_file`/`add_symlink`, `get_file` | JSON/text file content collected into the model's virtual package |

Every `add_*` takes `exists_ok=True` to fetch-or-create instead of raising `AlreadyRegisteredError`. Every
`get_*` raises `PartialNotFoundError` (a `ModelBuildError`) if the partial doesn't exist or is the wrong kind
- **use `get_*`, not `add_*(..., exists_ok=True)`, when you only ever expect to read/append to something
another preset must have already created.** `AddToDictionary`/`AddToList`/`AddFacetGroup` all follow this
rule: they call `get_dictionary`/`get_list` and raise loudly if the target is missing, rather than silently
conjuring it into existence.

`BuilderClass.mixins`/`.base_classes`/`.fields` are `_GuardedList`/`_GuardedDict`: mutating them (directly,
or via `add_mixins`/`add_base_classes`/`add_field`) works fine before the partial is built. **After
`build()` has run for that partial, the same mutation is silently a no-op except for a `PostBuildMutationWarning`**
- the data is still lost, the warning only makes the loss visible. Never write code that mutates a partial's
raw container hoping it "usually runs early enough" - declare the dependency instead (see below) so the
builder guarantees it runs before the partial is built.

## Presets: `provides` / `modifies` / `depends_on` / `only_if`

```python
class Preset:
    provides: tuple[str, ...] = ()    # partial names this preset creates
    modifies: tuple[str, ...] = ()    # partial names this preset mutates (does not create)
    depends_on: tuple[str, ...] = ()  # partial names this preset only reads (fully built, read-only)
    only_if: tuple[str, ...] = ()     # partial names that must exist for this preset to run at all
```

`sorter.sort_presets` uses these four tuples - not the order presets are listed in `presets=[...]` - to build
a dependency graph and topologically sort: the creator of a partial always runs before every preset that
`modifies` it, and a preset with `depends_on=("X",)` gets `X` eagerly built (and thus frozen) before its
`apply()` even runs, so `dependencies["X"]` inside `apply()` is always the fully-settled value. `only_if`
prunes a preset out entirely if the named partial was never created by anything else in the model (used for
optional cross-cutting features like `DraftsUILinksPreset`, which only makes sense when a `"Draft"` partial
exists).

**Declare every partial you touch.** A preset that creates `"FacetGroups"` but forgets to list it in
`provides` still works today, but: nothing else can correctly declare a dependency on it, `only_if=("FacetGroups",)`
presets waiting on it are silently skipped, and `check_preset_declarations` (run automatically after every
build) emits a `PresetDeclarationWarning` naming the offending preset. Same for `modifies`: a preset that
writes into a dict/list without declaring `modifies` leaves its ordering relative to other writers to chance.

Minimal example (`FinalizationPreset`, `src/oarepo_model/presets/records_resources/finalizers.py`):
```python
class FinalizationPreset(Preset):
    provides = ("api_finalizers", "app_finalizers", "finalizers")

    def apply(self, builder, model, dependencies):
        yield AddModule("finalizers", exists_ok=True)
        yield AddList("api_finalizers")
        yield AddList("app_finalizers")
        ...
```
Every partial the preset's `apply()` creates via `yield AddList/AddModule/...` is listed in `provides`.

### The `feature_preset(...)` factory

Most `<X>FeaturePreset` classes (records, files, drafts-records, drafts-files, custom-fields, relations, ui,
internal-relations) are one call to `feature_preset(feature_key, version, base=...)`
(`src/oarepo_model/presets/records_resources/ext.py`) instead of a hand-written `Preset` subclass - it builds
a mixin that merges `{feature_key: {"version": version}}` into the `Ext` class's `model_arguments["features"]`
and prepends it via `PrependMixin("Ext", ...)`. **Reach for this factory instead of writing a new
`<X>FeaturePreset` class by hand** when adding a feature-flag-style preset; only write a bespoke `Preset` when
the feature needs more than "record one version string under `features`". Similarly, `file_record_preset`/
`file_metadata_preset` (`presets/records_resources/files/`) and `PathDumperExtPreset`
(`presets/records_resources/records/path_dumper_ext.py`) are factories for "one file/media-file record
variant" and "one dumper extension over a set of model paths matching a datatype" respectively - check
whether your new preset is actually an instance of one of these shapes before writing a new class.

## Customizations: `modifies` / `modifies_own_name`

```python
class Customization:
    modifies_own_name: bool = False   # True: the partial this customization modifies is self.name

    @property
    def modifies(self) -> tuple[str, ...]:
        ...  # override for anything not covered by modifies_own_name
```

A `Customization.__init__(name)` stores `self.name` - the constructor-given identifier - but that is *not*
automatically what the customization modifies. Declare one of:
- `modifies_own_name = True` (class attribute) when the partial is exactly `self.name` (the common case: see
  `AddClassField`, every `Add*`/`AddTo*` customization in `src/oarepo_model/customizations/`),
- `modifies = ("FixedPartialName",)` (class attribute) for a customization that always touches the same
  partial regardless of its constructor args (e.g. `AddFacetGroup.modifies = ("FacetGroups", "DraftFacetGroups")`),
- override the `modifies` property for any other per-instance case (e.g. `RelationFieldCustomization.modifies = ("relations",)`
  while `self.name` is the relation's field name, used for a different purpose).

Falling back to neither (the old implicit `(self.name,)` default) still works but logs a one-time
`DeprecationWarning`-style message per class and is planned for removal - **never write a new
`Customization` subclass without one of the three declarations above.**

`AddEntryPoint` is the deliberate exception: entry points are collected straight onto the builder
(`builder.entry_points`) and written out at the very end of `build()`, not through any partial, so it
declares `modifies = ()`.

## Ordering guarantee for user-supplied customizations

`model(..., customizations=[...])` accepts a list of customizations applied on top of the presets. Each one
is pulled forward and applied immediately before the first preset whose `depends_on` names one of the
partials in that customization's `modifies` - so e.g. `AddToList("order_test", "item")` runs before any
preset that declared `depends_on=("order_test",)`, guaranteeing that preset sees the user's addition. A
customization with no matching depender runs last, in the order it was passed. See
`tests/customizations/test_user_customization_order.py` for the executable specification of this rule -
read it before changing anything in `api.py`'s pre-preset customization handling or `sorter.py`.

## Datatypes

`DataType` subclasses (`src/oarepo_model/datatypes/`) implement one JSON-model field type (`keyword`, `int`,
`object`, `array`, `pid-relation`, ...) and are looked up by name through `DataTypeRegistry`
(`datatypes/registry.py`), populated from `entrypoints.py`'s `DATA_TYPES` dict plus anything a model's own
`types=[...]` argument registers. Each `DataType` implements a handful of `create_*` methods
(`create_marshmallow_field`/`create_marshmallow_schema`, `create_ui_marshmallow_fields`/
`create_ui_marshmallow_schema`, `create_json_schema`, `create_mapping`, `get_facet`) - the base class
(`datatypes/base.py`) provides sane composition (e.g. reading `marshmallow_field`/`marshmallow_validate`
from the element dict) so a new type usually only needs to override `marshmallow_field_class`,
`jsonschema_type`, `mapping_type`, and whichever `create_*` methods actually differ from the default object/
array/scalar behavior. `ObjectDataType`/`ArrayDataType` (`datatypes/collections.py`) are the two composite
base classes almost everything else builds on (nested objects via `_get_properties`, arrays via
`_get_items` - override these, not the whole `create_*` method, when only the "how do I find my
children" logic differs).

The `"name[]"` property-name shortcut (`"tags[]": {"type": "keyword"}` meaning an array of keywords named
`tags`) is expanded once, centrally, in `DataTypeRegistry._unwind_shortcuts_in_properties` - a new `DataType`
never needs to special-case it.
