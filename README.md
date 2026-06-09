# Contribution 1: Unify & de-duplicate JSON metadata validation (zarr-python)

**Contribution Number:** 1  
**Student:** Yong-Shin Jiang
**Issue:** https://github.com/zarr-developers/zarr-python/issues/3285  
**Status:** Phase I — Complete

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

(Exact paths to confirm in Phase II.)
- The metadata parsing/validation helpers (the `parse_*` functions, likely under
  `src/zarr/core/metadata/`).
- Their call sites across the metadata (v2/v3) handling code.
- Any existing typing/metadata utilities that should be reused rather than
  duplicated.

---

## Reproduction Process

> Refactor issue — no bug to reproduce. Phase II will instead map the current
> duplication and call sites.

### Environment Setup

[Phase II — set up the zarr-python dev environment following the project's
contributing guide.]

### Investigation Steps (planned)

1. Locate every `parse_*` helper in the metadata layer.
2. Catalog the duplicated validation logic and what each helper checks.
3. Identify all call sites that would migrate to the unified path.

### Findings

- **My findings:** [Phase II]

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

**Understand:** Metadata validation is duplicated across `parse_*` helpers;
consolidate it into one shared, type-driven path. Maintainer's priority:
de-duplication.

**Match:** Study how the existing `parse_*` helpers and any current
typing/metadata utilities work, and reuse them rather than reinventing.

**Plan:**
1. Map all `parse_*` helpers and their call sites.
2. Design a single shared `parse_json(value, type_annotation)`-style entry point.
3. **Open a draft PR showing the direction and tag @d-v-b for early feedback**
   (per his request) before polishing.
4. After the direction is confirmed, migrate call sites and remove duplicates.
5. Keep existing metadata tests green; add tests for the unified validator.

**Implement:** [Links to branch/commits — Phase II]

**Review:** [Self-review — follows CONTRIBUTING, no behavior change, tests pass — Phase II]

**Evaluate:** [How I verified equivalence after de-dup — Phase II]

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
