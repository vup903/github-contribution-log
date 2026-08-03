# Contribution 3: Support Amazon Bedrock for trace-based LLM-judge evaluation (mlflow/mlflow)

**Contribution Number:** 3
**Student:** Yong-Shin Jiang
**Issue:** https://github.com/mlflow/mlflow/issues/24033
**Fork branch:** _TBD (will be created after triage/assignment)_
**PR:** _TBD_
**Status:** Phase I — Issue selected; claim comment posted (2026-07-29), awaiting triage/`ready` label and assignment (no response as of 2026-08-03)

---

## Why I Chose This Issue

For this cycle I widened the search beyond a single repo: I screened 11
high job-signal AI/LLM repos in parallel (google/adk-python,
pydantic/pydantic-ai, openai/openai-agents-python, BerriAI/litellm,
vllm-project/vllm, stanfordnlp/dspy, run-llama/llama_index, mlflow/mlflow,
huggingface/huggingface_hub, mem0ai/mem0, confident-ai/deepeval) through
the same five-gate process I used for Contribution 2, looking specifically
for issues that would still be unclaimed by the time I could act on them.

mlflow/mlflow#24033 stood out. MLflow's GenAI evaluation/judge framework is
squarely in the LLM-observability category I'm targeting for AI/LLM
application roles, and `make_judge(...)`'s trace-based mode (the judge
inspects full agent spans, tool calls, retrieval results) is the framework's
most powerful evaluation primitive. It silently fails with a
`NotImplementedError` for any `bedrock:/` model URI, which forces AWS-only
teams to either downgrade to a weaker judge mode or bring in a non-AWS
provider key just to evaluate a Bedrock-hosted agent.

The claim situation is what made this the strongest pick out of everything
the scout found. A community member (Shlok148Dev) posted a full root-cause
analysis and proposed patch on 2026-06-22, asked to be assigned, and got
zero maintainer response for over five weeks — no `ready` label, no
assignment, no PR ever opened. That matches the exact "polite takeover"
trigger from my issue-selection process (claimant quiet 1-2+ weeks,
unassigned, no PR), rather than a live race against an active claimant. I
also verified against current `main` before claiming: the bug still
reproduces, and reading the actual code turned up a real technical question
Shlok's proposal didn't address (see Solution Approach below), so my claim
comment adds something rather than just restating his analysis.

I also specifically re-checked google/adk-python (my Contribution 2 repo)
this cycle, since I wanted to see if a second, different-subsystem
contribution there was viable. It wasn't right now — every well-scoped
issue I found was already claimed or PR-swarmed within hours, a step up in
competitiveness even from Contribution 2's baseline. I'm keeping one weak
backup there (#3246, blocked on a stale competing PR) to recheck in a few
weeks.

---

## Understanding the Issue

### Problem Description

`mlflow.genai.judges.make_judge(...)` supports two evaluation modes:
inputs/outputs judges (routed through LiteLLM) and trace-based
"Agent-as-a-Judge" judges (routed through MLflow's own AI Gateway adapter,
so the judge can inspect full trace spans). The trace-based mode raises
`NotImplementedError: AmazonBedrockProvider does not implement
get_endpoint_url` for any `bedrock:/<model-id>` URI. The same URI works
fine in inputs/outputs mode, and other providers (`anthropic:/`,
`openai:/`) work fine in trace-based mode — only the Bedrock +
trace-based-judge combination fails.

### Expected Behavior

`make_judge(..., model="bedrock:/<model-id>")` with a `{{ trace }}`
placeholder should evaluate successfully, the same as it does for other
providers, when Bedrock API-key (bearer token) authentication is
configured.

### Current Behavior

Verified directly against `main` (2026-07-29). The trace-based judge path
goes through `mlflow.genai.judges.adapters.gateway_adapter.GatewayAdapter`,
which resolves an HTTP endpoint via:

```python
# mlflow/genai/judges/adapters/gateway_adapter.py
endpoint = base_url or provider_instance.get_endpoint_url("llm/v1/chat")
```

and then issues a raw HTTP request against that endpoint, expecting an
OpenAI-chat-completions-shaped response. `AmazonBedrockProvider`
(`mlflow/gateway/providers/bedrock.py`) does not override
`get_endpoint_url`, so the base class raises:

```python
# mlflow/gateway/providers/base.py
def get_endpoint_url(self, route_type: str) -> str:
    raise NotImplementedError(f"{self.__class__.__name__} does not implement get_endpoint_url")
```

Notably, `AmazonBedrockProvider` already has a *working* bearer-token chat
path used elsewhere in the gateway (`_chat_with_token`, via the Bedrock
Converse API) — it's specifically the raw-HTTP `get_endpoint_url` path that
`GatewayAdapter` depends on that's missing. See Solution Approach for why
this matters.

### Affected Components

- `mlflow/gateway/providers/bedrock.py` — `AmazonBedrockProvider` (missing
  `get_endpoint_url` override)
- `mlflow/genai/judges/adapters/gateway_adapter.py` — the call site that
  depends on it
- `tests/gateway/providers/test_bedrock.py` — existing test pattern to
  extend

### Community Context (from the thread)

- 2026-06-22 — Shlok148Dev posted a detailed root-cause analysis and a full
  proposed patch (implement `get_endpoint_url` to return a new
  OpenAI-compatible "bedrock-mantle" endpoint under bearer-token auth,
  expose a `headers` property, and route `adapter_class` to `OpenAIAdapter`
  under token auth), asked to be assigned, and posed two open design
  questions. No maintainer response since — no `ready` label applied, no
  assignment, no PR opened, 5+ weeks of silence as of 2026-07-29.

---

## Reproduction Process

> Phase II. I verified the code-level defect directly (see Current Behavior
> above) rather than running a live reproduction, since exercising the
> actual `NotImplementedError` requires a Bedrock-hosted trace + API-key
> credentials I don't have. Planned approach after assignment:

### Environment Setup (planned)

- **OS:** Windows 11, no GPU needed (this is a pure request-routing bug).
- **Python:** per repo requirements (`pyproject.toml`).
- **Live Bedrock verification:** I don't have AWS/Bedrock credentials, so
  end-to-end validation will lean on mocked HTTP/boto3 requests following
  the existing pattern in `tests/gateway/providers/test_bedrock.py` — I'll
  flag this in the PR and ask for a real-Bedrock spot-check during review,
  same approach I used for the missing-Vertex-environment gap in
  Contribution 2.

### Steps to Reproduce (planned)

1. Construct an `AmazonBedrockProvider` with bearer-token auth configured.
2. Call `get_endpoint_url("llm/v1/chat")` directly — confirm it currently
   raises `NotImplementedError`.
3. Exercise `GatewayAdapter._invoke_and_handle_tools` against a mocked
   Bedrock provider instance and confirm the trace-based judge path
   currently fails at the same point.
4. After the fix, confirm both return a valid response instead of raising.

---

## Solution Approach

### Analysis

The reported fix (Shlok148Dev's proposal) adds a new raw-HTTP endpoint
shaped like OpenAI's chat-completions API and points `get_endpoint_url` at
it. That would work, but `AmazonBedrockProvider` already has a complete,
working bearer-token chat implementation (`_chat_with_token`) that goes
through the Bedrock Converse API — it's just that `GatewayAdapter` never
calls it, because it bypasses the provider's own `_chat()`/`adapter_class`
machinery entirely and does raw HTTP against `get_endpoint_url` instead.

### Open Design Question (asked in claim comment, awaiting maintainer read)

Rather than adding a new endpoint that mimics OpenAI's wire format (which
depends on an unverified AWS "bedrock-mantle" domain), would it make more
sense for the trace-based judge path to call through the provider's
existing `_chat()` method, which already has bearer-token auth working
through the Converse API? I raised this in the claim comment instead of
committing to Shlok's endpoint-based direction, per MLflow's own
`CONTRIBUTING.md` guidance to align on approach with a maintainer before
implementing changes to the gateway/adapter layer.

### Implementation Plan (UMPIRE, adapted — pending direction confirmation)

**Understand:** trace-based LLM-judge evaluation can't use `bedrock:/` URIs
because `AmazonBedrockProvider` doesn't expose an endpoint the
`GatewayAdapter`'s raw-HTTP call site can use.

**Match:** `OpenAIProvider.get_endpoint_url` is the shape `GatewayAdapter`
expects; `_chat_with_token`/`_get_token_auth_headers` are the existing
Bedrock bearer-token building blocks.

**Plan (direction A — reuse existing chat path):** teach `GatewayAdapter`
(or a thin provider-side shim) to call through the provider's own
`_chat()` when a raw endpoint isn't available, instead of assuming every
provider exposes `get_endpoint_url`.
**Plan (direction B — Shlok's proposal):** implement `get_endpoint_url` +
`headers` + `adapter_class` routing to `OpenAIAdapter` under bearer-token
auth, if a genuine OpenAI-compatible Bedrock endpoint exists.
Whichever direction the maintainers confirm, add unit tests to
`tests/gateway/providers/test_bedrock.py` covering the trace-based judge
path under bearer-token auth, and a non-token-auth case that fails clearly
(SigV4 signing isn't supported for this path).

**Implement / Review / Evaluate:** Phase III (after direction is
confirmed and the issue is assigned).

---

## Testing Strategy

- [ ] Unit tests in `tests/gateway/providers/test_bedrock.py` (bearer-token
      trace-judge path works; non-token-auth path fails with a clear error)
- [ ] Existing gateway/judge test suites stay green (no regressions)
- [ ] Mocked HTTP/boto3 verification (no live AWS account available);
      real-Bedrock spot-check requested during review

---

## Implementation Notes

_[Phase III]_

---

## Pull Request

**PR Link:** _TBD — this repo's contribution rules require maintainer
triage + the `ready` label + an assignment before a PR is opened; PRs
without triage may be auto-closed._

**Maintainer Feedback:**
- 2026-07-29: Posted claim comment — verified the bug still reproduces on
  current `main`, acknowledged Shlok148Dev's prior analysis and asked if
  they're still working on it, raised the reuse-existing-chat-path
  alternative to their proposed new-endpoint design, and asked for triage +
  `ready` label + assignment.
  (https://github.com/mlflow/mlflow/issues/24033#issuecomment-5127320038)
  Awaiting response.

- 2026-08-03: Status check, 4 days after claiming. No reply from Shlok148Dev, no
  maintainer response, no `ready` label, no assignee, and still zero linked PRs
  on the issue timeline — so the claim is uncontested but also hasn't gained
  traction yet.

  I checked whether the missing `ready` label is a rejection signal and concluded
  it isn't: of the currently open `domain/genai` issues, 51 lack the `ready`
  label versus 27 that have it, so roughly two thirds of live GenAI issues sit
  in the same state. Triage in this area *is* active (maintainer B-Step62 has
  been responding on GenAI issues through 2026-08-03, and `area/gateway` bugs
  filed in late July were `ready`-labeled within days), which suggests bug
  reports get triaged faster than feature requests rather than that this
  specific issue has been declined.

  Plan: hold off on a follow-up until roughly 2026-08-08 (about the 1-week
  mark), then post a short nudge tagging the active GenAI maintainer rather than
  re-pinging the thread generally. Per this project's own screening rules, a
  follow-up carrying new technical substance lands far better than a bare "any
  update?", so the intent is to pair the nudge with concrete findings on the
  open design question (reuse the provider's existing `_chat()`/Converse path
  vs. adding a new OpenAI-shaped endpoint).

---

## Learnings & Reflections

### Issue Selection (Phase I)

- Scouting many repos in parallel surfaced a pattern the single-repo
  approach in Contribution 2 didn't show as clearly: a fresh, unclaimed,
  well-written bug in a hot repo (e.g. a same-day vLLM or DSPy issue found
  this cycle) is a race against the clock, while a fully-specified issue
  that's gone quiet for weeks in an otherwise fast-moving repo is often the
  safer claim — the silence itself is the signal, not a red flag, as long
  as you verify the defect still exists before trusting it.
- Cross-repo screening also made repo-specific claim dynamics visible:
  several repos in this scout (litellm, deepeval, llama_index) are now
  effectively swarmed by automated bots or a handful of extremely fast
  repeat contributors racing every well-scoped issue within hours, which
  changes the realistic strategy from "find a clean bug" to "find the rare
  one the swarm missed" or "self-report a bug in an under-audited
  subsystem."
- Reading the actual current source before claiming (not just the issue
  text) paid off again: it both confirmed the bug is still live and
  surfaced a genuine alternative design worth raising, the same pattern
  that worked well in Contribution 2's claim comment.

### Technical Skills Gained

_[Phase III/IV]_

---

## Resources Used

- Issue: https://github.com/mlflow/mlflow/issues/24033
- Repository: https://github.com/mlflow/mlflow
- Contribution guide: `CONTRIBUTING.md` in the repo (design alignment
  expected before implementation for changes to critical internal
  abstractions like the gateway; issue must be triaged + `ready`-labeled +
  assigned before a PR)
- Prior community analysis: Shlok148Dev's comment, issue thread,
  2026-06-22
- Claim comment draft & rationale: `d:\AI301\issue-24033-claim-comment.md`
