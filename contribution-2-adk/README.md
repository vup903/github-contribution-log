# Contribution 2: Fix LiteLlm dropping native tools through proxy gateways (google/adk-python)

**Contribution Number:** 2
**Student:** Yong-Shin Jiang
**Issue:** https://github.com/google/adk-python/issues/6091
**Fork branch:** _TBD (will be created after assignment)_
**PR:** _TBD_
**Status:** Phase I — Issue selected; claim comment posted (2026-07-13), awaiting maintainer response/assignment

---

## Why I Chose This Issue

My goal for this cycle was a contribution with strong signal for AI/LLM
application engineering roles. google/adk-python is Google's official Agent
Development Kit — the Python framework for building LLM agents on Gemini and,
via its LiteLLM integration, on any provider. This issue sits exactly at that
integration boundary: ADK's `LiteLlm` wrapper silently drops native/built-in
tools (e.g. `google_search`) when requests route through a proxy gateway, so
agents lose capabilities with no error. It is a serialization-boundary bug —
data silently lost between two systems — which matches the skills I built in
my first contribution (zarr-python #4063).

I selected it through a systematic screening process (a five-gate checklist I
encoded as a reusable skill): job-signal fit, skills/environment fit (pure
Python, no GPU, Windows-testable), repo health (adk-python merges daily,
including community PRs), issue quality, and claim status. The claim-status
gate mattered most: in today's ecosystem, well-specified bugs in popular AI
repos attract competing PRs within hours. This issue survived that window —
it was opened 2026-06-12, a collaborator outlined the fix on 2026-06-15, the
original reporter went silent for four weeks, the issue was closed for
inactivity and then deliberately **reopened by a Google engineer on
2026-07-10**, with the collaborator explicitly inviting anyone to pick it
back up ("Feel free to reopen or tag us... happy to pick it back up!"). Zero
linked PRs existed at claim time. A maintainer-confirmed bug with a blessed
fix direction and no active claimant is close to the ideal shape for a
course-cycle contribution.

---

## Understanding the Issue

### Problem Description

When ADK's `LiteLlm` model integration is used through a proxy gateway
(e.g. an OpenAI-compatible gateway in front of Vertex AI or OpenAI), tools
that do not carry `function_declarations` — native/built-in tools such as
`google_search` — are silently dropped from the outgoing request. They never
reach the gateway. The agent behaves as if the tool doesn't exist, with no
error or warning.

### Expected Behavior

All tools configured on the request — function tools *and* native tools —
should be serialized into the LiteLLM request so the downstream
provider/gateway can act on them (or fail loudly if unsupported).

### Current Behavior

In `_get_completion_inputs` (`src/google/adk/models/lite_llm.py`, the
"Convert tool declarations" block, currently L2359–2367 on `main`), the
entire tool-conversion step is gated on
`llm_request.config.tools[0].function_declarations` being truthy:

```python
tools: Optional[List[Dict[str, Any]]] = None
if (
    llm_request.config
    and llm_request.config.tools
    and llm_request.config.tools[0].function_declarations
):
  tools = [
      _function_declaration_to_tool_param(tool)
      for tool in llm_request.config.tools[0].function_declarations
  ]
```

Two symptoms follow:
1. A `Tool` carrying only native tools (no `function_declarations`) produces
   `tools=None` — native tools silently dropped (the reported bug).
2. Any `Tool` beyond index 0 is ignored entirely — a second symptom of the
   same gate that I spotted while verifying the mechanism on `main`, and
   flagged in my claim comment so it can be covered by the same fix.

### Affected Components

- `src/google/adk/models/lite_llm.py` — `_get_completion_inputs` (the fix)
- `tests/unittests/models/test_litellm.py` — regression coverage (to add)
- Affects at least Vertex AI (Gemini) via OpenAI-compatible proxy and OpenAI
  paths, per the issue report.

### Maintainer Context (from the thread)

- 2026-06-15 — collaborator @surajksharma07 root-caused the gate and outlined
  the fix: loop over tools, keep function tools on the existing conversion
  path, and serialize native tools via
  `tool.model_dump(by_alias=True, exclude_none=True)` into the same combined
  list; asked for testing across both provider paths before a PR.
- 2026-07-10 — collaborator closed for reporter inactivity but confirmed the
  bug still exists in v2.4.0; the same day, Google engineer @sasha-gitg
  reopened the issue, keeping it alive for someone to pick up.

---

## Reproduction Process

> Phase II. Planned approach (to be executed after assignment):

### Environment Setup (planned)

- **OS:** Windows 11
- **Python:** per repo requirements (`pyproject.toml`)
- **Repo:** fork of google/adk-python; local shallow clone already inspected
  at `d:\AI301\adk-python`
- **E2E harness:** local `litellm proxy` to observe the outgoing request body
  on the wire (OpenAI path); mocked OpenAI-compatible backend for the
  Vertex-shaped path (no Vertex environment available — disclosed in the
  claim comment; will request a real-Vertex check during review)

### Steps to Reproduce (planned)

1. Configure an ADK agent with `LiteLlm` pointing at a local litellm proxy.
2. Attach a native tool (e.g. `google_search`) — observe the request body
   contains no tools.
3. Attach `[Tool(native), Tool(function_declarations=[...])]` — observe that
   only index-0 handling occurs (second symptom).
4. Unit-level: call `_get_completion_inputs` directly and inspect the
   returned `tools` value for native-only / mixed / multi-`Tool` configs.

---

## Solution Approach

### Analysis

The tool-conversion gate conflates "has tools" with "has function
declarations on the first Tool". Native tools have no
`function_declarations`, so the gate treats them as "no tools". The fix is to
enumerate all `Tool` objects and dispatch per kind, not per index-0 shape.

### Proposed Solution (following the collaborator's outline)

Replace the gate with a loop over all `llm_request.config.tools`:
- function declarations → existing `_function_declaration_to_tool_param`
- native tools → `model_dump(by_alias=True, exclude_none=True)`
- both merged into the single `tools` list passed to `litellm.completion`

Open design question (asked in claim comment, awaiting maintainer read):
for providers that reject native tool schemas, should mixed native+function
requests still send both, or drop native tools with a warning?

### Implementation Plan (UMPIRE, adapted)

**Understand:** native tools are silently dropped at the LiteLLM
serialization boundary; fix must preserve existing function-tool behavior.

**Match:** the existing `_function_declaration_to_tool_param` conversion path
and the reporter's confirmed-working subclass workaround are the templates;
the collaborator's outline is the blessed direction.

**Plan:**
1. Rewrite the "Convert tool declarations" block as a loop over all tools.
2. Add unit tests: native-only, mixed native+function, multiple `Tool`
   objects, no-tools regression — asserting exactly what reaches
   `litellm.completion`.
3. E2E: local litellm proxy on the OpenAI path; mocked OpenAI-compatible
   backend for the Vertex-shaped path.
4. PR with the testing-plan section required by the contribution guide;
   sign Google CLA (bot-driven at PR time).

**Implement / Review / Evaluate:** Phase III (after assignment).

---

## Testing Strategy

- [ ] Unit tests in `tests/unittests/models/test_litellm.py` (native-only /
      mixed / multi-Tool / no-tools)
- [ ] Existing test suite green (no regressions in function-tool behavior)
- [ ] Wire-level check via local litellm proxy (OpenAI path)
- [ ] Mocked OpenAI-compatible backend for Vertex-shaped path; real-Vertex
      verification requested during review

---

## Implementation Notes

_[Phase III]_

---

## Pull Request

**PR Link:** _TBD_

**Maintainer Feedback:**
- 2026-07-13: Posted claim comment — verified the gate on current `main`,
  flagged the second symptom (index-0-only handling), laid out the fix +
  test plan per the collaborator's outline, disclosed the missing Vertex
  environment, and asked one scoped design question. Awaiting response.

---

## Learnings & Reflections

### Issue Selection (Phase I)

- Popular AI repos are swarmed: well-specified fresh bugs attract competing
  PRs within hours, and detailed issue reporters usually intend to fix their
  own reports. The viable pool is (a) maintainer-authored unclaimed issues,
  (b) stalled design discussions, and (c) abandoned claims that maintainers
  deliberately keep alive — this issue is type (c).
- Claim-status verification requires reading **full** comment text and the
  issue timeline (cross-referenced PRs, including closed ones); truncated
  summaries mislead.
- Repo health screening eliminated entire ecosystems up front (autogen:
  maintenance mode; letta/chainlit/ragas: stalled; OpenAI/Anthropic SDKs:
  codegen-closed), saving effort that would have died in review queues.

### Technical Skills Gained

_[Phase III/IV]_

---

## Resources Used

- Issue: https://github.com/google/adk-python/issues/6091
- Repository: https://github.com/google/adk-python
- Contribution guide: `CONTRIBUTING.md` in the repo (comment-first rule for
  non-help-wanted issues; testing-plan section required in PRs; Google CLA)
- Collaborator's fix outline: issue thread, 2026-06-15
- Claim comment draft & rationale: `d:\AI301\issue-6091-claim-comment.md`
