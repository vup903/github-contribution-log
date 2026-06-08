# Contribution 1: Abort request on client disconnect (vLLM-Omni)

**Contribution Number:** 1  
**Student:** Ypng-Shin Jiang
**Issue:** https://github.com/vllm-project/vllm-omni/issues/1347  
**Status:** Phase I — Complete 

---

## Why I Chose This Issue

This issue asks vLLM-Omni to abort an in-flight request when the client
disconnects, so the server stops spending GPU compute on a result no one is
waiting for. It matters because abandoned long-running generations otherwise
hold onto scarce GPU memory and throughput — a real cost and reliability issue
for production serving. I chose it because it sits squarely in LLM/inference-
serving, the area I most want to deepen as a CS master's student, and because
the scope is clean: PR #1156 already merged the cancellation mechanism (a
`@with_cancellation` decorator plus a signal-based interrupt) for the
image-editing endpoint, so my task is to extend that proven pattern to the
generation/variation endpoints that still lack it. That lets me learn how a real
serving engine manages request lifecycles and async cancellation, while keeping
the change focused enough to land cleanly.

---

## Understanding the Issue

### Problem Description

When a client disconnects (or explicitly cancels) an in-flight inference
request, vLLM-Omni does not stop the work for every endpoint — the server keeps
running the generation and holding GPU resources for a result that will never be
delivered. Cancellation is currently wired into only the image-editing endpoint;
the generation/variation endpoints do not honor it.

### Expected Behavior

When the HTTP client disconnects, the server should abort the associated
request, stop the remaining diffusion/generation steps, and release the GPU
resources promptly — consistent with how the `edit_images` endpoint already
behaves via the `@with_cancellation` decorator and signal-based interrupt added
in #1156.

### Current Behavior

Disconnecting during a long-running generation/variation request does not stop
the work. The request continues to completion, unnecessarily consuming GPU
memory and throughput. Only `edit_images` currently supports cancellation.

### Affected Components

Based on the mechanism introduced in #1156 (exact paths/names to confirm in
Phase II):
- The API route handlers for the generation/variation endpoints (the serving
  layer that needs the cancellation decorator applied).
- The shared `@with_cancellation` decorator and the signal-based (SIGUSR1)
  interrupt flow that notifies worker processes.
- The worker/engine code that checks the interrupt flag and skips the remaining
  diffusion (DIT) steps.
- The post-cancellation GPU-memory release/cleanup path.

---

## Reproduction Process

> Phase II — no local environment is set up yet (Phase I is selection only).

### Environment Setup

[To be completed in Phase II — set up the vLLM-Omni dev environment following the
project's contributing guide.]

### Steps to Reproduce (planned)

1. Start the vLLM-Omni server with a diffusion model.
2. Send a long-running generation/variation request, then disconnect the client
   mid-request.
3. Observe that the server keeps processing and holds GPU resources instead of
   aborting (compare against the `edit_images` endpoint, which aborts correctly).

### Reproduction Evidence

- **Commit showing reproduction:** [Phase II]
- **Screenshots/logs:** [Phase II]
- **My findings:** [Phase II]

---

## Solution Approach

### Analysis

This is a coverage gap, not a broken mechanism: #1156 added a working
cancellation path, but only `edit_images` applies it. The generation/variation
endpoints never opt into the decorator/interrupt flow, so client disconnects are
not propagated to the worker.

### Proposed Solution

Reuse the cancellation mechanism merged in #1156 rather than building new
infrastructure: apply the `@with_cancellation` decorator to the
generation/variation endpoint handlers and ensure the
disconnect → interrupt-signal → worker interrupt-flag → GPU-release flow works
for those request paths. The change intentionally excludes the broader
cross-worker interruption-granularity redesign discussed in the RFC. Final scope
pending maintainer confirmation on #1347.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Client disconnects abort only `edit_images` today; extend the
same behavior to the generation/variation endpoints.

**Match:** PR #1156 is the reference implementation — it added the
`@with_cancellation` decorator + SIGUSR1 interrupt to `edit_images`. Mirror that
pattern.

**Plan:**
1. Confirm target endpoint(s) and scope with the maintainer on #1347.
2. Locate the generation/variation route handlers and the `@with_cancellation`
   utility.
3. Apply the cancellation decorator and wire the interrupt path for those
   handlers.
4. Verify GPU memory/resources are released on cancel.
5. Add or adapt a cancellation test mirroring the existing `edit_images`
   coverage.

**Implement:** [Links to branch/commits — Phase II]

**Review:** [Self-review checklist — follows CONTRIBUTING, matches the #1156
pattern, tests pass — Phase II]

**Evaluate:** [How I verified it works — Phase II]

---

## Testing Strategy

### Unit / Integration Tests (planned)

- [ ] Simulate a client disconnect on a generation request and assert the
      request is aborted.
- [ ] Assert GPU resources/memory are released after cancellation.
- [ ] Mirror any existing `edit_images` cancellation test for the new endpoint(s).

### Manual Testing

[Phase II — manual disconnect test and observed result.]

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

**PR Link:** [Phase III]

**PR Description:** [Adapted from the sections above when submitted.]

**Maintainer Feedback:**
- [Date]: [Summary of feedback]
- [Date]: [How I addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

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

- Issue: https://github.com/vllm-project/vllm-omni/issues/1347
- Reference PR (cancellation mechanism): https://github.com/vllm-project/vllm-omni/pull/1156
- vLLM-Omni repository: https://github.com/vllm-project/vllm-omni
- vLLM-Omni docs: https://docs.vllm.ai/projects/vllm-omni
