# Testing oarepo-model (and repos with their own presets/customizations/datatypes)

**Scope:** read this only when testing `oarepo-model` itself, or the parts of another repo that define their
own `Preset`/`Customization`/`DataType` subclasses (see [model_architecture.md](model_architecture.md)) - a
plain extension package like oarepo-doi or oarepo-glitchtip has none of this and should follow
[testing.md](testing.md) alone.

## Layout and what each layer is for

```
tests/
├── conftest.py            # shared fixtures, incl. _build_model() and every session-scoped model
├── customizations/        # unit tests: Customization.apply() against a builder built with MagicMock(model)
├── datatypes/              # unit tests: DataType methods against a real DataTypeRegistry, no app/DB
└── api_tests/              # integration tests: a real Flask app, DB, and search backend
```

**Match the layer to the claim you're testing.** `tests/customizations/*` and `tests/datatypes/*` should
never need `app`/`search_clear`/a running service - if a customization test needs those, it's testing
integration behavior and belongs in `api_tests/` instead (or the unit test is over-scoped and should mock
more). Conversely, don't add an `api_tests/` integration test for something a unit test already pins - e.g.
`tests/customizations/test_facet_group.py` unit-tests `AddFacetGroup`'s dict-writing logic with a mocked
builder, while `tests/api_tests/test_facets.py` integration-tests that faceted search actually filters
records end-to-end. Both exist because they check different things (construction logic vs. wired-up
runtime behavior), not because one is redundant with the other - don't delete either kind assuming
duplication; check what specifically each one pins first.

## Building a model in a test

Never call `oarepo_model.api.model(...)` and `.register()` per-test for anything you plan to reuse - SQLAlchemy
maps each model's tables once per process, and building/registering the same model shape twice across
different session fixtures will raise a mapper-configuration error. All session-scoped model fixtures live in
the **top-level `tests/conftest.py`** and funnel through the shared helper:

```python
def _build_model(name, types, presets, customizations=(), **kwargs):
    """Build, register and time a session-scoped test model."""
    ...

@pytest.fixture(scope="session")
def empty_model(model_types):
    from oarepo_model.presets.records_resources import records_resources_preset
    return _build_model("test", model_types, [records_resources_preset, ui_links_preset])
```

When you need a new model shape for a test, add a new `scope="session"` fixture in `conftest.py` (or the
relevant subpackage's `conftest.py` if it's local to one test module) using `_build_model` - don't hand-roll
`model(...)`/`.register()`/timing/logging again, and don't build a fresh model per-test-function unless you
have a specific reason it can't be shared (e.g. it must observe mutable module-level state cleared per test,
as in `test_user_customization_order.py`).

For unit tests that don't need a full model (customization/datatype tests), build a bare
`InvenioModelBuilder(MagicMock(), MagicMock())` (or `MagicMock()` type registry only) directly - see any file
in `tests/customizations/` for the pattern.

For everything else - what to parametrize, how to write assertions that can fail, `xfail(strict=True)`,
docstring conventions - see [testing.md](testing.md); it applies here too, this file only adds the
model-specific layer on top.
