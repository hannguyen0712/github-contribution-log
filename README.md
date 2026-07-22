# Contribution #1: Langfuse Integration with apache/burr

**Contribution Number:** 1

**Student:** Han Nguyen

**Issue:** [\[Langfuse Integration\]  ](https://github.com/apache/burr/issues/206)

**Status:** Phase 1 - Complete. Phase 2 - Complete. Phase 3 - Complete. Phase 4 - Complete (PR #844 open, awaiting maintainer review).

---

## Why I Chose This Issue

I chose this issue since I am looking to build AI engineering and software engineering skills. I have taken relevant courses at university, and I'm looking to put the skills into practice. The issue involves a trace-based Langfuse integration for Burr, and Burr's hook system makes it a clean place to learn how to map an application's execution model (runs, steps, state reads and writes) onto an external tracing model built on OpenTelemetry, and I am drawn to the design judgment the maintainer flagged: deciding what belongs in a span, how to represent state observations, and where to draw the line between a minimal first version and the richer features layered on top.

What I hope to get out of it is depth in LLM observability and OpenTelemetry specifically, which is a skill set I expect to matter more and more as agent systems move from demos into things people actually deploy. I want to learn how a mature framework like Burr structures its tracking client so that a new integration follows existing patterns rather than fighting them, and I want practice scoping a net-new feature into a reviewable first PR with tests and an example, then iterating with a maintainer rather than dropping a single large change. Contributing to an Apache project also gives me a chance to work on a real review bar, which is the part of open source I find most valuable.

---

## Understanding the Issue

### Problem Description

Burr applications have no built-in way to send execution traces to Langfuse, a popular LLM observability platform. Today, if you want to inspect what a Burr app did across a run (which steps were executed, in what order, what state each step read and wrote, where time or errors occurred), you either rely on Burr's own telemetry UI or wire up tracing by hand. There is no integration that automatically captures a Burr run as a structured trace in Langfuse, which is where many teams already centralize observability for their LLM and agent systems. The maintainer opened this issue to close that gap. Burr's hook system makes it possible to emit one trace per application run, one span per step, and observations on state, but that integration does not exist yet and has to be built.

### Expected Behavior

A user should be able to enable Langfuse tracing on a Burr application with minimal setup and have each run automatically captured as a structured trace. Concretely, one trace per application run, one span per step (action) within that run, and relevant state reads and writes surfaced as observations on the corresponding spans. Errors during a step should be recorded on the span rather than silently lost, and the integration should follow Burr's existing hook and tracking-client patterns so it composes cleanly with the rest of the framework. A new contributor should be able to look at an example and see how to attach the integration to their own app.

### Current Behavior

There is no Langfuse integration in Burr. To get Burr execution data into Langfuse today, a user has to instrument tracing manually, mapping Burr's run and step lifecycle onto Langfuse traces and spans themselves. Burr exposes the necessary information (run boundaries, per-step execution, state I/O) through its hook system, but nothing in the codebase translates that into Langfuse traces automatically, so the data either stays in Burr's own telemetry UI or requires custom glue code per project.

### Affected Components

The integration centers on Burr's hook system (the lifecycle hooks that fire on application and step boundaries) and follows the pattern established by the existing tracking client, which the issue references as the model to mirror. Because the maintainer confirmed an OpenTelemetry-based approach is acceptable, any existing OTel tracing support in Burr is also in scope to build on rather than duplicate. New code would likely live under an integrations or plugins module (for example, a burr/integrations/langfuse.py or similar, following however Burr organizes external integrations), plus an example under examples/ and accompanying docs.


---

## Reproduction Process

### Environment Setup

I set up the development environment on macOS (Apple Silicon). It took a few iterations to get a clean dev install, and a couple of the issues are worth recording since another contributor on macOS is likely to hit the same ones.

My first attempt used the system Python 3.9 from the Xcode command line tools, and Burr failed to launch on import because `burr/cli/__main__.py` uses `dict | None` union syntax in a runtime-evaluated annotation, which only works on Python 3.10+. Burr's contributing docs state it supports Python 3.9, so this is a genuine incompatibility (tracked separately as a candidate bug). I moved to Python 3.12 via Homebrew and installed Burr in editable mode from a clone, using the `[developer]` extra the contributing guide specifies, which adds the test runner and pre-commit hooks.

```bash
brew install python@3.12
python3.12 -m venv .venv
source .venv/bin/activate
pip install -e ".[developer]"
```

Installing from a clone with `-e` rather than the published wheel matters here, since the integration work means editing source and needing those changes picked up immediately. With that in place I confirmed the environment by running `hello-world-counter` end to end and watching the run land in Burr's telemetry UI. Working through the example also clarified how Burr separates its two state stores, the persister and the tracker, and how its lifecycle hooks fire on application and step boundaries. That hook surface is exactly what this contribution attaches to, so the setup doubled as orientation for the actual work.

### Baseline Behavior (Current State)

Since this is a net-new feature rather than a bug, there is no failing case to reproduce. The baseline is the absence of the integration: with a standard Burr install, there is no way to send execution traces to Langfuse. To confirm the gap, I attached Burr's existing tracking client to an app and verified that run and step data flows to Burr's own telemetry UI, but that no path exists to forward that same execution data (run boundaries, per-step execution, state reads and writes) to Langfuse without writing custom glue. This establishes both what Burr already exposes through its hooks and what the integration needs to add.

### Local Build Evidence
- **Fork branch:** [feature/langfuse-integration](https://github.com/hannguyen0712/burr/tree/feature/langfuse-integration) (implements issue #206)
- **Screenshots/logs:**
<img width="3020" height="1804" alt="image" src="https://github.com/user-attachments/assets/652cc224-b212-4f11-9c63-c6ec5b9404c7" />
<img width="2176" height="986" alt="image" src="https://github.com/user-attachments/assets/c7ca7295-7e37-4089-ac37-4825137365b4" />

### Findings

- Burr's hook system surfaces everything the integration needs: application-level boundaries for the trace, step-level boundaries for spans, and state I/O for observations.
- The existing tracking client is the right pattern to mirror, confirmed by the maintainer in the issue.
- An OpenTelemetry-based approach is acceptable, so existing OTel support is something to build on rather than duplicate.

---

## Solution Approach

### Analysis
This is a design problem rather than a root-cause problem, and the design landscape shifted meaningfully between the issue being filed (May 2024) and my picking it up. The original proposal was a dedicated Langfuse client mirroring `burr/tracking/client.py`, with the Langfuse client passed in as a runtime input, which depended on issue #205 (global inputs). #205 was closed as not planned, and in the meantime the Langfuse Python SDK was rebuilt on OpenTelemetry (v3 in mid-2025, currently v4.14). The v3+/v4 SDK registers a span processor on the global OTel tracer provider, which means any spans emitted through standard OTel tracers flow to Langfuse automatically, and Burr already has an `OpenTelemetryBridge` lifecycle hook that maps runs, steps, and tracer spans onto OTel spans. So the analysis reduced to: the integration should be a thin, Langfuse-aware layer over the existing OTel bridge, not a parallel client. This matches the direction I proposed on the issue and that the maintainer approved.

Three non-obvious discoveries came out of testing against the real SDK and real backend rather than just the docs:

1. **Langfuse v4 silently drops unknown spans.** The v4 SDK applies a default export filter (`is_default_export_span`) that only keeps spans created by the Langfuse SDK tracer, spans carrying `gen_ai.*` attributes, or spans from a hardcoded allowlist of known LLM instrumentors. Raw Burr spans match none of these, so a naive bridge would produce zero traces in the UI with no error anywhere. The fix is to inject a custom `should_export_span` filter into the Langfuse client that extends the default with Burr's tracer scope (with a graceful fallback on v3, where everything exports by default).
2. **A pre-existing bug in Burr core.** While testing error propagation I found that `_call_execute_method_pre_post` in `burr/core/application.py` initializes `exc = None` inside a `try/finally` with no `except` clause, so the `post_run_execute_call` hook *always* receives `exception=None`. This means even the existing `OpenTelemetryBridge` never marks a root span as errored when a run fails (Langfuse still flags the trace via child-observation errors). I prepared a four-line fix for each of the sync and async wrappers (`except BaseException as e: exc = e; raise`) with sync/async regression tests, but deliberately kept it out of the integration PR to keep the PR single-scope; it is documented in the PR's Notes with an offer to file it as an issue or separate PR, whichever the maintainer prefers.
3. **OTel attributes reject complex values.** Manual verification against Langfuse Cloud surfaced that attributes logged via `@trace()`/`log_attributes` with list-of-dict values (e.g. a `chat_history` argument) were dropped by the OTel SDK with a warning, since OTel attributes only accept primitives and flat sequences of primitives. The integration now JSON-serializes complex attribute values into observation metadata (`_convert_attribute` in `langfuse.py`), with a test asserting the round-trip. The underlying passthrough behavior exists in Burr's `convert_to_otel_attribute` too and affects the plain OTel bridge, another candidate for a small upstream report.

### Proposed Solution
A `LangfuseBridge` class in `burr/integrations/langfuse.py` that subclasses `OpenTelemetryBridge` and is attached via `.with_hooks(LangfuseBridge())`. It delivers everything the issue asked for:

- **One trace per application execution call** (`run`/`step`/`iterate`/`stream_result`/...), rooted at the `pre_run_execute_call` hook.
- **One span per step**, with the action's inputs and read state captured as the Langfuse observation input, and its result and written state as the observation output (using Burr's `serde` for serialization, and filtering out internal dunder inputs like `__tracer`).
- **One span per tracer span** opened through Burr's `__tracer` API, inherited from the parent bridge.
- **Observations at will**: attributes logged via `__tracer.log_attributes()` land as Langfuse observation metadata (complex values JSON-serialized; `langfuse.*` and `gen_ai.*` keys passed through unprefixed so users can deliberately set mapped attributes).
- **Access to the Langfuse client** via `bridge.langfuse_client`, e.g. for `flush()` in short-lived scripts or for scoring traces.

Two Burr-to-Langfuse mappings fell out naturally: `app_id` maps to the Langfuse session (so multiple execution calls of one application group together) and `partition_key` maps to the Langfuse user, both overridable via constructor args. These attributes are set on every span, not just the root, because Langfuse's session/user aggregations operate per-observation. A `capture_state=False` flag turns off state capture for applications with sensitive state, and `burr_span_export_filter` is exported publicly for users who construct their own `Langfuse` client. Because everything rides on the global OTel provider, any third-party LLM instrumentor (e.g. `opentelemetry-instrumentation-openai`) automatically appears as generations nested inside the correct Burr step spans, with prompts, completions, and token usage.

The core `application.py` exception-propagation fix was prepared and verified locally with its own regression tests, but is deliberately **not** part of PR #844: keeping the feature PR to a single goal makes it easier to review, and the maintainer gets to choose how the core fix is tracked (it is flagged in the PR's Notes with the fix ready to submit).

### Implementation Plan
Using UMPIRE framework (adapted):

**Understand:** Burr needs a hooks-based integration that maps its execution model onto Langfuse's trace model: execution call → trace, step → span, tracer span → nested span, state I/O → observation input/output, logged attributes → metadata, with the Langfuse client accessible and errors recorded.

**Match:** `OpenTelemetryBridge` in `burr/integrations/opentelemetry.py` already solves the hard span-lifecycle problems (context token stack, enter/exit pairing across hooks, error status). The traceloop integration shows the docs pattern for OTel-vendor pages; `require_plugin` in `burr/integrations/base.py` is the standard optional-dependency guard; `tests/integrations/test_burr_opentelemetry.py` sets the test style; `examples/integrations/` is the home for integration examples (exempt from the notebook/statemachine.png requirements in `validate_examples.py`).

**Plan:**
1. Add `burr/integrations/langfuse.py` with `LangfuseBridge(OpenTelemetryBridge)`, `burr_span_export_filter`, and serialization helpers.
2. Override `pre/post_run_execute_call`, `pre/post_run_step`, `pre_start_span`, and `do_log_attributes` to layer Langfuse attributes (`langfuse.observation.input/output`, `langfuse.observation.metadata.*`, `langfuse.session.id`, `langfuse.user.id`) onto the parent's span handling.
3. Add `tests/integrations/test_burr_langfuse.py` covering the span tree, attributes, and configuration.
4. Add the `langfuse` extra to `pyproject.toml` (and to the `tests` extra), `docs/reference/integrations/langfuse.rst` wired into the integrations index, and a runnable example under `examples/integrations/langfuse/`.
5. Separately (not in this PR): prepare the `exc = None` fix for both wrappers of `_call_execute_method_pre_post` in `burr/core/application.py` with two regression tests in `tests/core/test_application.py`, to be filed per the maintainer's preference.

**Implement:** Complete; branch [feature/langfuse-integration](https://github.com/hannguyen0712/burr/tree/feature/langfuse-integration), submitted as [PR #844](https://github.com/apache/burr/pull/844).

**Review:** Formatted and linted with the repo's pre-commit tooling (black at line length 100, isort with the black profile, flake8, trailing-whitespace/EOF checks, ASF license-header check) — all hooks pass. Docs page follows the traceloop/opentelemetry RST structure. Example placed where `validate_examples.py` permits a README + application.py without a notebook. PR uses the project's template with the acceptance checklist filled in.

**Evaluate:** Automated tests assert the full span tree and attributes against an in-memory OTel exporter (no credentials needed); manual verification ran the example against a free Langfuse Cloud project to confirm ingestion and rendering in their UI, since the automated tests mock the Langfuse client itself. Both are complete (see Testing Strategy).

---

## Testing Strategy

### Unit Tests
- [x] `test_burr_span_export_filter`: Burr-scoped spans pass the export filter, and spans matching Langfuse's default criteria (e.g. `gen_ai.*`) still pass.
- [x] `test_langfuse_bridge_rejects_client_and_kwargs`: passing both a pre-constructed client and constructor kwargs raises `ValueError`.

### Integration Tests
All run a real two-action Burr app (with a tracer span and logged attributes) against an in-memory OTel `TracerProvider`/`InMemorySpanExporter` with a mocked Langfuse client, so no network or credentials are needed in CI:
- [x] `test_langfuse_bridge_span_structure_and_attributes`: one root span per `run` call, step spans parented to the root, tracer span parented to its step, all in one trace; session/user attributes on every span; step observation input contains action inputs and read state (with `__tracer` filtered out); step output contains result and written state; trace-level output reflects final application state; `__tracer.log_attributes` lands in observation metadata, with complex values (a list of dicts) JSON-serialized and round-tripped through `json.loads`.
- [x] `test_langfuse_bridge_capture_state_false`: no observation input/output attributes anywhere when state capture is disabled.
- [x] `test_langfuse_bridge_session_and_user_overrides`: explicit `session_id`/`user_id` override the `app_id`/`partition_key` defaults.
- [x] `test_langfuse_bridge_records_exceptions`: a raising action marks the step span as errored. The root-span assertion is deliberately omitted for now, with a comment in the test marking it to strengthen once the core exception-propagation gap is fixed (this test is what surfaced that gap).

Prepared but not in PR #844 (pending maintainer direction on the core fix): two regression tests in `tests/core/test_application.py` (sync + async) asserting `post_run_execute_call` receives the actual exception when a step raises.

Results: 6/6 integration tests pass on the PR branch; `tests/core`, `tests/visibility`, and the existing OTel integration tests pass locally; all pre-commit hooks pass.

### Manual Testing
Complete. I ran `examples/integrations/langfuse/application.py` against a free Langfuse Cloud project with `opentelemetry-instrumentation-openai` installed, and verified in their UI (screenshots embedded in [PR #844](https://github.com/apache/burr/pull/844)):
1. One trace per `.run()` call appears in the trace list.
2. The trace tree nests correctly: root `run` span → `process_prompt`/`chat_response` step spans → `prepare_history` tracer span and `_query_openai` → the OpenAI generation with model, role-tagged prompt/completion messages, token usage (334 → 322, ∑ 656), and cost.
3. Step observation input/output render the read/written state; `history_length` appears in `prepare_history`'s metadata; the previously dropped `chat_history` argument now renders as JSON in `_query_openai`'s metadata (this run is what surfaced discovery #3 above; fixed and re-verified clean).
4. The Sessions view groups both `.run()` calls of one application under a single session with user `example-user`, confirming the `app_id` → session and `partition_key` → user mapping.
5. A forced failure (invalid OpenAI key) marks the step span and generation with ERROR status and the 401 message; the trace is flagged via child errors; the root span carries no error status, which is the documented core gap, not an integration limitation.

---

## Implementation Notes

### May 31 — Scoping
Commented on the issue proposing the OTel-based direction instead of the original dedicated-client approach (which depended on the closed #205), and scoped a focused first PR: one trace per run, one span per step, state observations, plus example and docs, with follow-ups for anything beyond. Maintainer (@elijahbenizzy) replied same day: "Can work both ways -- up to you," and confirmed we can get around #205.

### July 1–12 — Implementation
Surveyed the codebase (`OpenTelemetryBridge`, lifecycle hook signatures, the traceloop docs pattern, test conventions, `validate_examples.py` rules) and verified the current Langfuse SDK against the real package rather than docs — which is how I caught that the SDK is now v4.14 (docs and the issue thread still talk about v3) and that v4's default export filter would silently drop Burr spans. Implemented `LangfuseBridge`, the export filter, tests, docs, the example, and the pyproject extra. Testing error propagation surfaced the `post_run_execute_call` exception bug in core; prepared the fix in both wrappers with regression tests, then decided to keep it out of the feature PR for scope discipline and offer it to the maintainer separately. First test run also caught that Burr injects `__tracer` into action inputs, which was leaking framework internals into the serialized observation input — added dunder filtering.

### July 12–22 — Verification, polish, and submission
Set up the fork remotes, placed the changes on `feature/langfuse-integration`, and ran the full local gate (tests + pre-commit; the repo's hooks reformatted my files on first commit, a useful lesson in trusting the project's pinned tooling over my own formatter versions). Manual verification against Langfuse Cloud surfaced the complex-attribute drop (discovery #3): list-of-dict values logged via `@trace()` were rejected by OTel with a warning. Fixed with JSON serialization (`_convert_attribute`), extended the test to cover it, and re-verified a clean run end to end, including the trace tree, nested OpenAI generation with tokens/cost, session/user mapping, and error-status behavior. Opened [PR #844](https://github.com/apache/burr/pull/844) on July 22 using the project's PR template, with six captioned verification screenshots, and pinged the maintainer on #206 with two process questions (how to track the core fix, and whether to open the follow-ups issue the original issue requested) rather than filing unilaterally.

**Image 1: Langfuse trace list of one trace per execution call**


<img width="1512" height="868" alt="langfuse-trace-list" src="https://github.com/user-attachments/assets/3f2fbae0-1d8b-45ea-b40c-68021f6a06a4" />


Each `.run()` call produces one Langfuse trace rooted at a `run` span, with step and tracer spans nested beneath it.


**Image 2: Nested OpenAI generation view. Third-party OTel instrumentation composes automatically**


<img width="1512" height="869" alt="langfuse-openai-query-token-view" src="https://github.com/user-attachments/assets/19c971e7-a3bc-4e26-8ecc-bdae925d3927" />


With `opentelemetry-instrumentation-openai` installed, the LLM call appears as a Langfuse generation nested inside the `_query_openai` tracer span, with model, role-tagged prompt/completion messages, token usage (334 → 322, ∑ 656), and cost. No Langfuse-specific code in the action.


**Image 3: When a run is invalid, error status displays properly**


<img src="https://github.com/user-attachments/assets/cb43689f-a27a-412d-bf15-f82d5f24c5e6" style="max-width: 100%; height: auto;" />   



A run with an invalid OpenAI key: the `chat_response` step span and the generation are marked ERROR with the full 401 message, and Langfuse flags the trace via the child errors. The root `run` span itself carries no error status, and this is the pre-existing core gap described in section Notes below (`post_run_execute_call` receives `exception=None`), not a limitation of this integration; the test comment marks the assertion to strengthen once that's fixed. Metadata also shows `burr.sequence_id` and the `burr.integrations.langfuse` tracer scope.


**Image 4: Tracer spans + logged attributes as observation metadata**


<img src="https://github.com/user-attachments/assets/6540ca76-8c22-4a5f-8c07-2d8dbd2ba517" style="max-width: 100%; height: auto;" />   


The `prepare_history` span opened via `__tracer(...)` appears under its step, and `__tracer.log_attributes(history_length=...)` lands in the span's Langfuse metadata.


**Image 5: Step state captured as observation input/output**


<img width="1512" height="869" alt="langfuse-chat-response-input-output" src="https://github.com/user-attachments/assets/a6811c22-ff41-4dfc-baeb-6ad6a353d933" />



The `chat_response` step's read state (`prompt`, `chat_history`) shows as observation input and its written state as observation output (disable with `capture_state=False`).


**Image 6: Langfuse session view indicates that user mapping works correctly**


<img src="https://github.com/user-attachments/assets/b78cc186-bd40-4e67-980c-d84341596d3f" style="max-width: 100%; height: auto;" />   


Burr's `app_id` maps to the Langfuse session (both `.run()` calls of one application grouped together) and `partition_key` to the user (`example-user`), enabling Langfuse's session and user views.



### Code Changes (PR #844)
- **Files added:** `burr/integrations/langfuse.py`, `tests/integrations/test_burr_langfuse.py`, `docs/reference/integrations/langfuse.rst`, `examples/integrations/langfuse/{__init__.py,application.py,README.md}`
- **Files modified:** `pyproject.toml` (`langfuse` extra + tests extra), `docs/reference/integrations/index.rst` (toctree entry)
- **Prepared but withheld pending maintainer direction:** `burr/core/application.py` exception-propagation fix + 2 regression tests in `tests/core/test_application.py`
- **Key commits:** \[Commit links\](https://github.com/apache/burr/pull/844/commits)
- **Approach decisions:**
  - *Subclass `OpenTelemetryBridge` instead of a standalone client:* the SDK's OTel-native rewrite means Langfuse ingests standard OTel spans off the global provider, so a parallel client following `tracking/client.py` would duplicate the bridge's span-lifecycle machinery and lose free nesting of third-party LLM instrumentor spans.
  - *Inject `should_export_span` rather than spoof the Langfuse tracer scope:* naming our tracer `langfuse-sdk` would pass the filter but misrepresent the spans to Langfuse's mapping layer; extending the filter is the documented, public mechanism, and only relying on public API (I deliberately avoided the private `_otel_span` on Langfuse's span wrappers).
  - *`app_id` → session, `partition_key` → user:* Burr's own docs treat partition_key as a per-user key, and sessions grouping multiple runs of one app instance matches Langfuse's session semantics; both are constructor-overridable, and set on every span because Langfuse aggregates these per-observation.
  - *`capture_state` flag:* state can contain sensitive data; the integration shouldn't force users to ship it to a third party to get tracing.
  - *JSON-serialize complex logged attributes:* OTel attributes reject sequences of non-primitives, so passing them through drops the data with only a warning; serializing preserves it, and Langfuse renders JSON strings in metadata.
  - *Exclude the core fix from this PR:* it's 8 lines plus tests and required for root-span error status, but it touches `application.py`; keeping the feature PR single-scope makes review easier, and the maintainer chooses whether it lands as an issue, a separate PR, or folded in. The fix and tests are ready either way.

---

## Pull Request

**PR Link:** [apache/burr#844](https://github.com/apache/burr/pull/844)

**PR Summary:** `feat: Langfuse integration` — adds `LangfuseBridge` (subclassing `OpenTelemetryBridge`, attached via `with_hooks`) providing one trace per execution call, one span per step and per tracer span, state/inputs/results as observation input/output, logged attributes as observation metadata (complex values JSON-serialized), `app_id`/`partition_key` mapped to Langfuse session/user (overridable), `capture_state=False` opt-out, the v4 `should_export_span` filter (public as `burr_span_export_filter`), and client access via `bridge.langfuse_client`. Includes tests, docs, a runnable example, a `langfuse` extra, and six Langfuse Cloud verification screenshots. Uses the project's PR template with the acceptance checklist complete; `Closes #206`. The Notes section documents the core exception-propagation gap found during testing and asks the maintainer how to track it.

**Maintainer Feedback:**
- Jul 22, 2026: PR #844 submitted; pinged @elijahbenizzy on #206 with the PR link and two process questions (whether to file the core `exception=None` bug as an issue/separate PR, and whether to open the follow-ups tracking issue for approach (2) that the original issue requested). Awaiting response.
- [Date]: [Next feedback + response]

**Status:** Open, awaiting maintainer review.

---

## Learnings & Reflections

### Technical Skills Gained
- How OpenTelemetry context propagation actually works under the hood: tracer providers, span processors, export filters, and the token/attach/detach mechanics Burr's bridge uses to pair spans across non-context-manager hook boundaries.
- Reading a dependency's source instead of trusting its docs: the v4 export filter behavior and the exact `should_export_span` mechanism weren't findable from documentation alone, and pip-installing the SDK to inspect `_client/span_filter.py` and the `Langfuse` constructor signature resolved in minutes what could have been a mystifying "no traces appear" failure later.
- Langfuse's OTel attribute mapping (`langfuse.observation.input/output`, `langfuse.observation.metadata.*`, `langfuse.session.id`, `langfuse.user.id`), the per-observation attribute requirement for session/user aggregations, and OTel's attribute type constraints (primitives and flat primitive sequences only), which forced an explicit serialization strategy for complex values.
- Testing tracing integrations hermetically with `InMemorySpanExporter` and an isolated `TracerProvider`, asserting on the full span tree rather than just "it didn't crash" — and why hermetic tests still need a manual backend-ingestion pass (the complex-attribute drop only surfaced against the real pipeline).

### Challenges Overcome
- The design target had drifted since 2024: the original approach depended on a closed issue and predated Langfuse's OTel rewrite. Re-scoping on the issue thread before writing code, and getting maintainer sign-off on the new direction, meant the implementation didn't fight the current SDK.
- A failing exception test turned out to be a bug in Burr core, not my code. Tracing it from "root span shows OK" through the hook invocation path to the `try/finally` with no `except` was a good exercise in not assuming the framework is right — and in first-contribution scope discipline: prepare the fix, test it, keep the feature PR single-goal, document the gap in the PR, and ask the maintainer how they want it tracked rather than deciding for them.
- My own first version leaked Burr's internal `__tracer` object into the serialized observation input — caught by the very first test run, which reinforced writing the assertions on exact payload contents.
- Manual verification caught a bug the automated suite couldn't: complex logged attributes were silently dropped by the OTel SDK before export. The mocked-client tests exercised everything up to the exporter, but only a real run showed the warning; the fix (JSON serialization) then got its own automated assertion so the gap stays closed.

### What I'd Do Differently Next Time
Verify the current state of the external dependency *before* scoping the work in the issue comment, not after: my May 31 comment said "Langfuse now ingests OpenTelemetry natively" based on v3, and the v4 filter behavior would have been worth mentioning then. I'd start from the framework's existing tests earlier — reading `test_burr_opentelemetry.py` first would have saved some time reverse-engineering hook signatures from `lifecycle/base.py`. And I'd run the manual end-to-end verification earlier in the cycle rather than treating it as a final gate, since it surfaced a real bug (the attribute drop) that would have been cheaper to fix before the branch was polished.

---

## Resources Used
- [Burr hooks concept docs](https://burr.dagworks.io/concepts/hooks/)
- [Burr OpenTelemetry integration reference](https://burr.apache.org/reference/integrations/opentelemetry/) and `examples/opentelemetry/`
- [Langfuse OpenTelemetry integration guide](https://langfuse.com/integrations/native/opentelemetry) (attribute mapping)
- [Langfuse SDK overview](https://langfuse.com/docs/observability/sdk/overview) and [existing-OTel-setup FAQ](https://langfuse.com/faq/all/existing-otel-setup) (export filtering, global tracer provider)
- [Langfuse Python SDK v3 announcement](https://langfuse.com/changelog/2025-05-23-otel-based-python-sdk) — context for the OTel rewrite
- Langfuse SDK source (`langfuse/_client/span_filter.py`, client constructor) via local install — the export-filter discovery
- [OpenTelemetry attribute specification](https://opentelemetry.io/docs/specs/otel/common/#attribute) — attribute type constraints behind the serialization fix
- [Issue #206](https://github.com/apache/burr/issues/206), [#205 (closed)](https://github.com/apache/burr/issues/205), [#203](https://github.com/apache/burr/issues/203)
- Hamilton's Datadog plugin (`h_ddog.py`), referenced in the issue as inspiration for the original design
