# Test conventions across oarepo libraries

## Baseline conventions (every oarepo-* repo)

Confirmed consistent across oarepo-model, oarepo-doi, and oarepo-glitchtip:

- `tests/` mirrors the package's own module structure (`tests/settings/test_service.py` next to
  `oarepo_doi/settings/service.py`), not a flat dump - a new test file goes where its subject module's
  shadow directory says it should, creating that subdirectory (with an `__init__.py`) if it doesn't exist yet.
- Test files carry the same SPDX headers as source files (see the top-level SKILL.md gotchas) - don't skip
  headers on tests on the assumption they're exempt.
- **Match fixture weight to what the test actually needs.** Not every test needs a full `pytest-invenio` app
  with DB/search - oarepo-doi's `tests/conftest.py` defines a minimal fixture (`Flask("oarepo-doi-tests")`
  wrapped in `app.app_context()`) for tests that only need config/context, and mocks the external DataCite
  HTTP client via `monkeypatch.setattr(client, "get_doi_settings", lambda record: ...)` rather than hitting a
  real service or spinning up a full app. Reach for the heaviest fixture (a real app + DB + search, needing
  `./run.sh start` first) only when the behavior under test genuinely depends on them.
- Test an extension's wiring, not just its logic in isolation: verify the entry-point-registered pieces
  actually do what they claim once `init_app`/`finalize_app` has run (e.g. a hook registered in
  `finalize_app` fires on a request), not only that the standalone function returns the right value.
- A DB-backed package's `alembic/` migrations get their own test (see oarepo-doi's `test_alembic.py`) -
  verifying migrations actually apply cleanly, not just that the revision files parse.

## oarepo-model-specific layer

If you're testing `oarepo-model` itself, or the preset/customization/datatype parts of another repo that
defines its own (see [model_architecture.md](model_architecture.md)), read
**[model_testing.md](model_testing.md)** as well - it covers the unit/integration test-layer split and the
session-scoped model-building fixtures, layered on top of everything below. A plain extension package (no
`Preset`/`Customization`/`DataType` subclasses) has no use for it.

## When to reach for `@pytest.mark.parametrize`

Parametrize when two or more test **bodies** differ only in the literal values plugged in - not when they
exercise genuinely different code paths that happen to look similar. Concrete signals it's time to
parametrize:

- **Two test classes/files that mirror each other 1:1**, differing only in which format/backend/type name is
  used (e.g. a `TestFromJson`/`TestFromYaml` pair with the same five test names in the same order; a
  `test_load_datatypes_from_json`/`test_load_datatypes_from_yaml` pair with the same payloads). Merge into
  one parametrized test/class over `(loader, ...)` instead of maintaining two copies that can silently drift.
- **A handful of tests that assert the same shape of thing for different inputs**: `test_X_rejects_a`,
  `test_X_rejects_b`, `test_X_rejects_c` with one bad value each → `@pytest.mark.parametrize("bad_value", [a, b, c])`.
  Accept/reject pairs (`test_min_rejects_too_few` / `test_min_accepts_exact_count`) are a good fit for
  `@pytest.mark.parametrize(("value", "should_raise"), [...])`.
- **A single test function with several independent inline `assert` statements** for different inputs of the
  same function (e.g. six `assert convert_to_python_identifier(x) == y` lines back to back) - split into
  parametrize cases so a failure on case 2 doesn't hide whether cases 3-6 still pass.
- **Coverage gaps that a parametrize would close for free**: if `int`/`float` have an exclusive-bound test
  but `long`/`double` don't, parametrizing over all four types is both less code and more coverage - check
  for this whenever you're about to parametrize something that currently only covers a subset of a type
  family.

**Don't** parametrize away tests whose bodies differ in what they actually assert or set up (different
mocks, different number of assertions, a fundamentally different scenario) just to reduce line count - that
produces a parametrize block nobody can read. If the shared 80% is real (e.g. `removes_key_from_params`,
`invalid_value_raises` are structurally identical across several geo/coordinate-system param interpreters)
but each case also has genuinely unique edge cases (antimeridian wrap for one, RA wrap for another), extract
only the shared-contract tests into a parametrized suite and keep the unique edge cases as separate,
non-parametrized tests in their own file/section.

## Writing assertions that can actually fail

- Never write `assert "expected thing" in output or "generic substring" in output` - the `or` branch usually
  makes the assertion pass regardless of whether the first check is true (e.g. `"class" in output` is true
  for almost any Python source dump; `"type" in parsed` is true for almost any JSON Schema fragment). Assert
  the specific string/value the test's name promises to check.
- A test for a `--flag`/option must assert something that would differ *because of* the flag, not just
  `exit_code == 0`. If a filter flag exists (e.g. "dump only generated schemas"), assert both that the
  expected item is present *and* that the excluded item is absent - a smoke test that only checks the command
  didn't crash won't catch the filter itself breaking.
- Don't assert against state you didn't set up in the test (e.g. a hardcoded list of *every* session-scoped
  model fixture defined anywhere in the suite, checked via `in result.output`). That couples the test to
  collection/fixture-execution order elsewhere in the suite and to unrelated modules' fixture names. Build
  (or reuse) exactly the fixture(s) the test needs and assert against that.

## Tracking a known-blocked feature: `xfail(strict=True)`

When a feature is genuinely blocked on something outside this repo (an upstream package that doesn't support
it yet), don't leave it untested or silently skipped - write the test for the real desired behavior and mark
it `@pytest.mark.xfail(strict=True)` with a docstring explaining the blocker. `strict=True` means the test
starts failing loudly (XPASS) the moment the real fix lands, forcing whoever notices to remove the marker and
finish the wiring, instead of the feature landing and nobody noticing the test was still (accidentally)
green. See `tests/api_tests/test_ui_links.py::test_ui_links_user_listing_has_self_html` for the pattern.

## Docstrings

Every test/class docstring in this codebase explains **why the behavior is being pinned**, not what the code
under test literally does line-by-line - e.g. "'id' is non-searchable here because the target may not be
introspectable yet (e.g. a still-building self-reference)" rather than "tests that id is not searchable".
Follow that convention: if a reader can't tell from the docstring *why* this assertion exists (a bug it
regression-tests, a non-obvious design decision, a contract another module relies on), add one line
explaining it.
