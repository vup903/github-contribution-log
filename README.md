# Contribution 1: Unify & de-duplicate JSON metadata validation (zarr-python)

**Contribution Number:** 1  
**Student:** Yong-Shin Jiang
**Issue:** https://github.com/zarr-developers/zarr-python/issues/3285  
**Fork branch:** https://github.com/vup903/zarr-python/tree/fix-issue-3285  
**Status:** Phase III — Complete

---

## Why I Chose This Issue

Zarr is a foundational piece of the scientific-Python and ML-data stack — it's
the chunked-array storage layer behind tools like xarray and Dask, so
contributing here means working on infrastructure that real data and ML
pipelines depend on. This issue is a focused refactor: the library currently
validates its JSON metadata through a set of scattered, overlapping `parse_*`
helpers, and the goal is to de-duplicate that logic into a single,
type-annotation-driven validation path. I chose it because it fits my Python
background well, sits in the data-infrastructure area I want to deepen as a CS
master's student, and is the kind of clean refactor where I can demonstrate good
API design and runtime-typing skills while making the codebase genuinely
simpler. The maintainer (d-v-b) confirmed that de-duplication is the biggest
win, which gives me a clear, well-scoped target.

---

## Understanding the Issue

> Note: this is a **refactoring** issue, not a bug — there is no failing
> behavior to reproduce. The work is consolidating duplicated validation logic.

### Problem Description

zarr-python validates its JSON metadata (array/group metadata fields) through a
number of separate `parse_*` helper functions. These helpers re-implement
similar validation logic in multiple places, so the same "check a value against
its expected type" behavior is duplicated rather than living in one shared,
reusable place.

### Expected Behavior (desired end state)

A single, shared validation path — e.g. a `parse_json(value, type_annotation)`
style entry point — that validates a JSON metadata value against its declared
type annotation, with clear error messages. The scattered `parse_*` helpers are
consolidated so the logic is defined once and reused.

### Current Behavior

Validation logic is spread across multiple `parse_*` helpers with overlapping
responsibilities, making the metadata layer harder to maintain and extend, and
allowing the same logic to drift between copies.

### Affected Components

Confirmed in Phase II by grepping `^def parse_` across `src/zarr` (~40 helpers
in ~14 files):
- `src/zarr/core/common.py` — `parse_order`, `parse_bool`, `parse_shapelike`,
  `parse_name`, `parse_enum`, ...
- `src/zarr/core/metadata/v3.py` — `parse_zarr_format`, `parse_node_type_array`,
  `parse_dimension_names`, `parse_codecs`, ...
- `src/zarr/core/metadata/v2.py` — `parse_zarr_format` (a near-duplicate of the
  v3 version), `parse_filters`, `parse_compressor`, ...
- `src/zarr/codecs/{blosc,gzip,zstd}.py` — `parse_clevel`, `parse_gzip_level`,
  `parse_zstd_level` (all "is this an int?" checks).
- Call sites across the v2/v3 metadata handling code that would migrate to the
  unified path.

---

## Reproduction Process

> Refactor issue — no bug to reproduce. Phase II will instead map the current
> duplication and call sites.

### Environment Setup

- **OS:** Windows 11.
- **Python:** the repo requires `>=3.12` (`.python-version` = 3.12,
  `pyproject.toml` `requires-python = ">=3.12"`). My machine had 3.11, so I
  installed Python 3.12.10.
- **Build/test tool:** `hatch` (envs defined in `pyproject.toml`; `uv.lock`
  present).

Setup commands:

```powershell
py -3.12 -m pip install hatch
# hatch was installed to .../Python312/Scripts which was not on PATH, so I
# invoke it as a module instead:
py -3.12 -m hatch env show
py -3.12 -m hatch env run --env test.py3.12-optional run
```

**Friction encountered & fixes:**
1. The `hatch.exe` shim was placed in a Scripts directory that is not on PATH,
   so `hatch` was "command not found". Fix: call it as `py -3.12 -m hatch ...`
   (no PATH edit required).
2. The `run` script (`pytest --ignore tests/benchmarks`) has no `{args}`
   placeholder, so passing a single test path to it is ignored and the full
   suite is collected. To run a focused file, invoke pytest directly in the env:
   `py -3.12 -m hatch run test.py3.12-optional:pytest tests/test_common.py -q`.
3. Collecting the full suite currently errors out during collection on
   `tests/test_group.py::test_group_name_properties: duplicate parametrization
   of 'store'` under pytest 9.1.0. This is a **pre-existing** issue unrelated to
   this contribution; I'll keep it in mind for Phase III and run focused test
   files / report it upstream if it persists.

**Environment verified working:** `py -3.12 -m hatch run
test.py3.12-optional:pytest tests/test_common.py -q` → **33 passed**. The env
builds, installs zarr in dev mode (Python 3.12.10, pytest 9.1.0), and runs
tests.

### Steps to Reproduce (demonstrating the duplication)

This is a refactor, so "reproduction" means showing the scattered, duplicated
validation logic rather than triggering a bug.

1. From the repo root, count the parsers: `grep -rn "^def parse_" src/zarr`
   → ~40 `parse_*` functions across ~14 files.
2. Open `src/zarr/core/common.py`: `parse_order` (L208) and `parse_bool` (L214)
   are each the same "check value, return it, else raise" boilerplate.
3. Compare `src/zarr/core/metadata/v2.py::parse_zarr_format` (L280) with
   `src/zarr/core/metadata/v3.py::parse_zarr_format` (L49): two nearly identical
   literal checks duplicated across two files.
4. Look at `src/zarr/codecs/{blosc,gzip,zstd}.py`: `parse_clevel`,
   `parse_gzip_level`, `parse_zstd_level` are all independent "is this an int?"
   checks.
5. **Expected end state:** one shared `parse_json(value, type_annotation)`
   handles all of these categories.
6. **Actual:** every field has its own hand-written parser, duplicating logic
   that can drift between copies.

### Findings

- **My findings:** The duplication falls into a small set of recurring
  patterns — literal checks, primitive (`isinstance`) checks, sequence→tuple
  coercion, and mapping/TypedDict validation. These map cleanly onto the
  `parse_json` categories the issue proposes (primitives, sequences, unions,
  literals, TypedDict, `Mapping[str, T]`), which confirms a single dispatch
  function can replace most of them. The two `parse_zarr_format` copies are the
  clearest example of logic that has already been duplicated across files.

### Reproduction Evidence

- **Working branch:** https://github.com/vup903/zarr-python/tree/fix-issue-3285

---

## Solution Approach

### Analysis

Each metadata field/type currently has its own ad-hoc parsing/validation helper,
so there is no single source of truth for "validate this value against its
declared type." The result is duplicated logic that can drift over time — which
is exactly the de-duplication the maintainer flagged as the biggest win.

### Proposed Solution

Consolidate the scattered `parse_*` helpers into a single shared,
type-annotation-driven validation entry point, then migrate existing call sites
to it and remove the duplicates. Per the maintainer's request, I'll first open a
**draft PR** that demonstrates this direction and get early feedback before
polishing. Scope stays on de-duplication; I won't expand it into a larger
metadata redesign.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Metadata validation is duplicated across ~40 `parse_*` helpers;
consolidate them into one shared, type-annotation-driven path
`parse_json(value, type_annotation)` that returns data assignable to the
annotation (coercing where sensible, e.g. `[1,2,3] -> (1,2,3)`) or raises a
useful error. Scope is limited to the JSON types the issue lists: primitives
(`None/str/int/float/bool`), `Sequence[T]`→tuple, unions/`Optional`,
`Literal[...]`, `TypedDict`, and `Mapping[str, T]`. Maintainer's priority:
de-duplication.

**Match:** Existing helpers are the templates — `check_literal` ≈
`common.py::parse_order` (L208), primitive check ≈ `common.py::parse_bool`
(L214), sequence→tuple ≈ `metadata/v3.py::parse_dimension_names` (L116),
literal-int ≈ `metadata/v3.py::parse_zarr_format` (L49). Prior art: the
maintainer's exploratory runtime type checker in PR
https://github.com/zarr-developers/zarr-python/pull/3400.

**Plan:**
1. Add `src/zarr/core/json_parse.py` with `parse_json` plus sub-parsers for
   literal / primitive / union / tuple / sequence / mapping / TypedDict.
   Critical edge case: `bool` is a subclass of `int`, so check `bool` before
   `int`.
2. Add `tests/test_json_parse.py` covering each category (valid passes, invalid
   raises, sequence→tuple coercion, bool/int, Optional, nested TypedDict).
3. As a proof of direction, migrate a small pilot set — `parse_order`,
   `parse_bool`, and `parse_zarr_format` — to call `parse_json`, keeping their
   public signatures and behavior identical.
4. **Open a draft PR showing the direction and tag @d-v-b for early feedback**
   (per his request) before migrating the remaining call sites.
5. After the direction is confirmed, migrate the rest and remove duplicates.

**Files I expect to touch:**
- ADD `src/zarr/core/json_parse.py`
- ADD `tests/test_json_parse.py`
- ADD a changelog fragment under `changes/`
- EDIT (pilot only) `src/zarr/core/common.py`, `src/zarr/core/metadata/v3.py`

**Implement:** [Phase III] Branch:
https://github.com/vup903/zarr-python/tree/fix-issue-3285

**Review:** Self-review against `.github/CONTRIBUTING.md` and
`docs/contributing.md`; add a `changes/` fragment; ensure `prek`/pre-commit
hooks (ruff, mypy) pass; conventional commits + PR linking #3285.

**Evaluate:** Unit tests for every category in `parse_json`; the pilot migration
must keep behavior identical, verified by running
`py -3.12 -m hatch env run --env test.py3.12-optional run` and confirming the
full suite stays green (no regressions).

---

## Testing Strategy

### Unit Tests (added)

New file `tests/test_json_parse.py` — 94 tests covering every `parse_json`
dispatch category:
- [x] Primitives pass; wrong types raise. Includes the bool/int edge case
      (`parse_json(True, int)` raises; `parse_json(1, bool)` raises).
- [x] `Literal` membership; `True` does not match `Literal[1, 2]`.
- [x] Unions / `Optional` (both `typing.Union` and `X | Y` spellings); no-match
      raises an aggregated error.
- [x] `tuple` (fixed-length, variadic, empty); `Sequence`/`list` coerced to a
      tuple; `Mapping[str, T]` returns a dict.
- [x] `TypedDict` required/optional/`NotRequired`/nested — incl. correct
      `NotRequired` handling under `from __future__ import annotations`.
- [x] Fallback `TypeError` for unsupported annotations; error messages name the
      offending value and expected type.

### Regression / Integration Tests (existing suite)

Each migration batch was validated against the project's own suites with no
regressions:
- [x] Pilot + Batch 1 (literals): `test_common`, `test_config`, `test_metadata`,
      `test_codecs`, `test_chunk_grids`, `test_json_parse` → **1196 passed**.
- [x] Batch 2 (codec primitives): `test_codecs`, `test_json_parse` →
      **794 passed**.
- [x] Full `uv run --frozen mypy` (190 source files) clean after every batch.
- [x] `ruff check` + `ruff format` (pinned v0.15.15) clean on all changed files.

### Tooling / Validation

- Lint and pre-commit.ci CI checks on the draft PR are **green**.
- Commands used locally:
  `py -3.12 -m hatch run test.py3.12-optional:pytest <paths> -q`,
  `py -3.12 -m uv run --frozen mypy`,
  `py -3.12 -m uv tool run ruff@0.15.15 check/format`.

### Known unrelated failure

The full-suite CI **Test** jobs are red due to a pre-existing collection error
in `tests/test_group.py` (`duplicate parametrization of 'store'`) and
`tests/test_docs.py` under the newer pytest — present on `main`, unrelated to
this change. The Lint/type checks (which gate this work) are green.

---

## Implementation Notes

### Phase III Progress

**What I built:**
- `src/zarr/core/json_parse.py` — the unified `parse_json(value, type_annotation)`
  validator, dispatching on `typing.get_origin`/`get_args` to sub-parsers for
  primitives, `Literal`, unions/`Optional`, `tuple` (fixed + variadic),
  `Sequence`/`list` (coerced to tuple), `Mapping`/`dict`, and `TypedDict`. Scope
  stays on the JSON types the issue lists.
- `tests/test_json_parse.py` — 94 unit tests (see Testing Strategy).
- `changes/3285.feature.md` — changelog fragment (project convention is `.md`).
- **Migrated 16 of the ~42 scattered `parse_*` helpers** to delegate to
  `parse_json`, in three reviewable batches:
  - **Pilot (3):** `common.py::parse_order`, `common.py::parse_bool`,
    `metadata/v3.py::parse_zarr_format`.
  - **Batch 1 — literals (7):** `config.py::parse_indexing_order`,
    `group.py::parse_node_type`, `metadata/v3.py::parse_node_type_array`,
    `chunk_key_encodings.py::parse_separator`, `group.py::parse_zarr_format`,
    `metadata/v2.py::parse_zarr_format`, `common.py::parse_name`.
  - **Batch 2 — codec primitives (6):** `zstd.py::parse_checksum`,
    `blosc.py::{parse_clevel,parse_blocksize,parse_typesize}`,
    `gzip.py::parse_gzip_level`, `zstd.py::parse_zstd_level`.
- Deferred (noted as follow-up in the PR): the `sequence→tuple`, `mapping`,
  `union`, and `other` helpers, which dispatch to registries/factory methods and
  fall outside the issue's stated JSON-type scope.

**Challenges faced & decisions:**
- **Behavior preservation across exception types.** Several helpers raise
  domain-specific errors (`MetadataValidationError`, `NodeTypeValidationError`)
  or specific messages that tests assert on. `parse_json` raises plain
  `ValueError`/`TypeError`, so each migration wraps the call and re-raises the
  original error type/message — public signatures and observable behavior are
  unchanged.
- **Circular imports.** `parse_json` is imported function-locally inside each
  migrated helper (the metadata/codec modules load early), avoiding import
  cycles.
- **bool vs int.** `parse_json(x, int)` deliberately rejects `bool` (a `bool` is
  an `int` subclass). The old `isinstance(x, int)` accepted it. I verified no
  caller/test passes a bool to the migrated int parsers, so this is a deliberate,
  more-correct strictening rather than a regression.
- **`TypedDict` + `NotRequired` under stringized annotations.** Reading
  `__required_keys__` misclassifies a class-syntax `NotRequired` key as required
  when `from __future__ import annotations` is active (which zarr uses
  everywhere). Reworked `_parse_typeddict` to resolve hints with
  `get_type_hints(..., include_extras=True)` and read the `Required`/`NotRequired`
  wrappers + `__total__` directly.
- **mypy friction.** `get_origin(hint) is Required` tripped mypy's
  comparison-overlap/unreachable checks (fixed by typing the local as `Any`), and
  `return parse_json(...)` from `-> int` helpers tripped `no-any-return` (fixed
  with annotated local assignments).

### Code Changes

- **Branch:** https://github.com/vup903/zarr-python/tree/fix-issue-3285
- **Files added:** `src/zarr/core/json_parse.py`, `tests/test_json_parse.py`,
  `changes/3285.feature.md`
- **Files modified:** `core/common.py`, `core/config.py`, `core/group.py`,
  `core/chunk_key_encodings.py`, `core/metadata/v2.py`, `core/metadata/v3.py`,
  `codecs/blosc.py`, `codecs/gzip.py`, `codecs/zstd.py`
- **Key commits:**
  - `2b57be8` Add unified parse_json runtime type checker
  - `0eba162` Migrate parse_order, parse_bool, parse_zarr_format (pilot)
  - `df1a0cc` Fix lint; make TypedDict NotRequired robust under future annotations
  - `78874b3` Annotate origin as Any to satisfy mypy in _parse_typeddict
  - `730274a` Migrate Batch 1 literal parsers
  - `6bde3f4` Migrate Batch 2 codec primitive parsers

---

## Pull Request

**PR Link:** Draft PR opened against `zarr-developers/zarr-python` from
branch `fix-issue-3285` (proof-of-direction draft, per maintainer request).

**PR Description:** Summarizes the unified `parse_json` module, the test suite,
and the incremental migration; asks for direction feedback on module
location/name, the exception-handling split, and `TypedDict` handling.

**Maintainer Feedback:**
- 2026-06-08: @d-v-b confirmed the biggest win is **de-duplication** and asked
  for a **draft PR showing the intended direction** as a starting point.
- [awaiting] Feedback on the draft PR direction.

**Status:** Draft PR open; Lint + pre-commit CI green; awaiting maintainer
direction feedback before migrating the remaining (deferred) call sites.

---

## Learnings & Reflections

### Technical Skills Gained

[Phase IV]

### Challenges Overcome

[Phase IV]

### What I'd Do Differently Next Time

[Phase IV]

---

## Resources Used

- Issue: https://github.com/zarr-developers/zarr-python/issues/3285
- Repository: https://github.com/zarr-developers/zarr-python
- Documentation: https://zarr.readthedocs.io
- Contributing guide: see the project's contributing docs in the repo
