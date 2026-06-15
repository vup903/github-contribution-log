# Contribution 1: Unify & de-duplicate JSON metadata validation (zarr-python)

**Contribution Number:** 1  
**Student:** Yong-Shin Jiang
**Issue:** https://github.com/zarr-developers/zarr-python/issues/3285  
**Fork branch:** https://github.com/vup903/zarr-python/tree/fix-issue-3285  
**Status:** Phase II — Complete

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

### Unit Tests (planned)

- [ ] Valid values pass validation for each metadata type.
- [ ] Wrong types raise a clear, helpful error.
- [ ] Edge cases: `Optional`/`Union`/nested annotations behave correctly.

### Integration Tests (planned)

- [ ] Existing metadata round-trip tests still pass after the de-duplication.

### Manual Testing

[Phase II]

---

## Implementation Notes

### Week [X] Progress

[Phase II — what I built, challenges, decisions.]

### Code Changes

- **Files modified:** [Phase II]
- **Key commits:** [Phase II]
- **Approach decisions:** [Phase II]

---

## Pull Request

**PR Link:** [Draft PR — Phase II]

**PR Description:** [Adapted from the sections above; will open as a draft first
to show direction, per maintainer request.]

**Maintainer Feedback:**
- 2026-06-08: @d-v-b confirmed the biggest win is **de-duplication** and asked
  for a **draft PR showing the intended direction** as a starting point.
- [Date]: [Next feedback + how I addressed it]

**Status:** Planning → Draft PR (Phase II)

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
