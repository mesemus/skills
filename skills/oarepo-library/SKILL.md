---
name: oarepo-library
description: Use when writing, reviewing, or debugging code in a CESNET oarepo-* library repository (oarepo-model, oarepo-doi, oarepo-glitchtip, oarepo-runtime, oarepo-ui, oarepo-vocabularies, oarepo-requests, etc.). Covers the shared ./run.sh/oarepo-cli developer workflow (venv, start/stop services, test, lint, license-headers), the generic Flask-extension architecture every oarepo library follows, oarepo-model's preset/customization/datatype builder architecture (also used by consumers like oarepo-rdm to build dynamic models), and this ecosystem's test conventions. Trigger on "./run.sh", "add a preset", "write a customization", "add a datatype", "add an Ext/extension", "review this diff/PR in an oarepo-* repo", or "run the tests for this library".
license: MIT
compatibility: Requires Python 3.14 and the project's own ./run.sh (uv-based bootstrap); Docker for integration tests.
---

# oarepo library development

Use the `ponytail full` and `caveman` skills if available.

CESNET's oarepo-* packages (oarepo-model, oarepo-doi, oarepo-glitchtip, oarepo-runtime, oarepo-ui, ...) share
one developer workflow (`run.sh` wrapping `oarepo-cli library ...`) and one underlying pattern: a Flask/Invenio
extension registered through entry points. **Most oarepo-* repos are plain extensions of that shape** (see
`references/architecture.md`) - `oarepo-doi`, `oarepo-glitchtip`, and similar packages have no notion of
"presets" or "datatypes" at all. **oarepo-model is the exception**: it *is* a framework for building dynamic
Invenio record models out of declarative presets, and that framework (`references/model_architecture.md`) is
also imported and driven by other packages (e.g. `oarepo-rdm`) to assemble their own models - so read the
model-architecture reference whenever the repo in front of you imports `oarepo_model` or defines
`Preset`/`Customization`/`DataType` subclasses, not only when you're inside `oarepo-model` itself.

## Workflow: `./run.sh`

`run.sh` is boilerplate copied verbatim across oarepo-* repos (confirmed byte-for-byte identical, modulo a
one-line vendoring tweak, across oarepo-model/oarepo-doi/oarepo-glitchtip): on first use it bootstraps a
`uv`-managed venv under `.tools/venv`, installs `oarepo-cli`, and forwards every argument to
`oarepo-cli library <args>`. **Always run `./run.sh --help` (or `./run.sh <command> --help`) first** - the
command set is the actual source of truth and evolves; do not assume the table below is exhaustive or that
flags haven't changed. If `run.sh` itself looks like it needs a behavior change specific to this repo, that's
almost always the wrong fix - look at `pyproject.toml` config or `oarepo-cli` itself instead (see Gotchas).

| Command | What it does |
|---|---|
| `./run.sh venv` / `install` | Create/verify `.venv` (editable install by default; `--no-editable` builds a wheel; `-f` recreates from scratch) |
| `./run.sh start` / `stop` | Start/stop Docker services (DB, search, S3, MQ, cache) for integration tests; writes `.env-services` |
| `./run.sh test [--with-coverage] [--skip-services] [-q] [pytest args...]` | Runs pytest; starts/stops services around the run unless `--skip-services`; anything after the flags is passed straight to pytest |
| `./run.sh lint` / `./run.sh check` | `lint` auto-fixes (ruff check --fix, ruff format, ty check --fix) then a license-header check and a `from __future__ import annotations` check, stopping at first failure; `check` is the same pipeline read-only (`lint --no-fix`) - use `check` in CI |
| `./run.sh format [--no-fix] [paths...]` | `ruff format` + `ruff check --fix`; `--no-fix` previews only |
| `./run.sh license-headers [--deep] [-o ORG]` | Adds SPDX headers to files missing `Copyright (c)`; **`--deep` additionally verifies the year range against git history** and exits 1 on mismatches - run this after rebasing or before opening a PR |
| `./run.sh shell` | Interactive shell inside the project's venv |

### Running tests directly

The project's own venv lives at **`.venv`** (not `.tools/venv`, which only holds `oarepo-cli` itself). Prefer
`.venv/bin/python -m pytest ...` when you need pytest flags `run.sh test` doesn't expose (e.g. `-k`, `--lf`,
`--cov=... --cov-report=term-missing`; check `[tool.pytest.ini_options] testpaths` in `pyproject.toml` for
where tests live - it's `["tests"]` in some repos, `["tests", "src"]` in others that also doctest source).
Integration tests need Docker services up (`SQLALCHEMY_DATABASE_URI`, OpenSearch, S3 - written to
`.env-services` by `./run.sh start`); source that file or export the vars before invoking pytest directly.
**If a previous run left services in a weird state (stale indices, leftover DB rows from a crashed run), do
`./run.sh stop && ./run.sh start` before re-running** - don't debug flaky integration failures without first
ruling out stale service state. Not every repo's test suite needs Docker at all - a small extension package
may run entirely against a lightweight `Flask()` app fixture with mocked external services; check what the
existing `conftest.py` fixtures actually spin up before assuming you need `start`/`stop`.

## Architecture

Read **[references/architecture.md](references/architecture.md)** for the pattern every oarepo library
follows regardless of size: the `Ext` class (`__init__(app=None)` → `init_app`/`init_config` →
`app.extensions[key] = self`) registered via `invenio_base.apps`/`invenio_base.api_apps` entry points, the
`proxies.py` convention, `config.py` defaults merged with `setdefault`, and how package layout
(flat `<pkg>/` vs `src/<pkg>/`) is a per-repo choice hatchling auto-detects either way - don't assume one.

If the repo also imports `oarepo_model` (builds an Invenio record model out of presets rather than being a
plain extension), read **[references/model_architecture.md](references/model_architecture.md)** as well -
before writing or reviewing a new `Preset`/`Customization` subclass, before adding a new `DataType`, or when
a build fails with `PartialNotFoundError`/`ApplyCustomizationError`/a `PresetDeclarationWarning` or
`PostBuildMutationWarning`. **The single most important rule from that reference, worth knowing without
opening it:** every `Preset` must declare in `provides`/`modifies` every partial it creates or touches, and
every `Customization` must declare `modifies` (or `modifies_own_name = True`) - the sorter uses these
declarations, not list order, to decide build order, and getting it wrong doesn't fail loudly (yet): it just
warns and leaves ordering to chance.

## Testing conventions

Read **[references/testing.md](references/testing.md)** before writing new tests or reviewing a test PR - it
covers the baseline conventions shared by every oarepo library (tests mirror the package's own module
structure, SPDX headers, prefer a minimal `Flask()`/mocked-service fixture over a full app when the code
under test doesn't need one), when to reach for `@pytest.mark.parametrize` instead of copy-pasting
near-identical test bodies, and the assertion patterns to avoid (weak
`"x" in output or "generic-word" in output` checks that can't fail).

If the repo also has its own `Preset`/`Customization`/`DataType` subclasses (see the Architecture section
above), read **[references/model_testing.md](references/model_testing.md)** as well - it layers on the
`tests/customizations`+`tests/datatypes` (unit, mocked builder) vs. `tests/api_tests` (integration, real
app+DB+search) split and the session-scoped model-fixture rules. Skip it for a plain extension package.

## Gotchas

- `run.sh` is generic boilerplate meant to be copied to new oarepo-* repos unchanged - if you need to change
  its behavior for this repo specifically, that's very likely the wrong file to edit; look at `pyproject.toml`
  config or `oarepo-cli` itself instead.
- Requires **Python 3.14** exactly (`requires-python = ">=3.14,<3.15"` in `pyproject.toml`, confirmed the same
  across oarepo-model/oarepo-doi/oarepo-glitchtip) - a `.venv` built with a different interpreter will look
  fine until a 3.14-only syntax feature (e.g. bare `except A, B:`) or dependency pin fails.
- Package layout is **not** consistent across repos: oarepo-model uses `src/oarepo_model/`, oarepo-doi and
  oarepo-glitchtip use a flat `oarepo_doi/`/`oarepo_glitchtip/` at the repo root. Hatchling auto-detects
  either with no extra `pyproject.toml` config in any of these repos - don't assume `src/` layout, check.
- `ty.toml` and the `lint`/`check` pipeline (ruff + ty + license headers + future-annotations check) are
  generated/enforced identically across repos (verified byte-identical `ty.toml` in all three) - a lint
  failure is very unlikely to be a repo-specific config issue; look at the actual code first.
- All source/test files carry SPDX headers (`# SPDX-FileCopyrightText: <year(s)> <holder>` /
  `# SPDX-License-Identifier: MIT`), not the old boilerplate comment block, in every repo checked. Run
  `./run.sh license-headers --deep` after touching a file's first commit year or rebasing across a year
  boundary - it's the only way to catch a stale year range.
- A file can carry **multiple** `SPDX-FileCopyrightText` lines (e.g. CESNET *and* University of West Bohemia
  for jointly authored files) - never assume there's exactly one and never drop one while normalizing headers.
