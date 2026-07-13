# Contribution #1: Langfuse Integration with apache/burr

**Contribution Number:** 1

**Student:** Han Nguyen  

**Issue:** [\[Langfuse Integration\]  ](https://github.com/apache/burr/issues/206#event-26164144705)

**Status:** Phase 1 - Complete. Phase 2 - Complete. Phase 3 - In progress.

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
- **Fork:** [Link to commit in your fork]
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
 
Two non-obvious discoveries came out of reading the actual SDK source rather than just the docs:
 
1. **Langfuse v4 silently drops unknown spans.** The v4 SDK applies a default export filter (`is_default_export_span`) that only keeps spans created by the Langfuse SDK tracer, spans carrying `gen_ai.*` attributes, or spans from a hardcoded allowlist of known LLM instrumentors. Raw Burr spans match none of these, so a naive bridge would produce zero traces in the UI with no error anywhere. The fix is to inject a custom `should_export_span` filter into the Langfuse client that extends the default with Burr's tracer scope (with a graceful fallback on v3, where everything exports by default).
2. **A pre-existing bug in Burr core.** While testing error propagation I found that `_call_execute_method_pre_post` in `burr/core/application.py` initializes `exc = None` inside a `try/finally` with no `except` clause, so the `post_run_execute_call` hook *always* receives `exception=None`. This means even the existing `OpenTelemetryBridge` never marks a root span as errored when a run fails. The fix is four lines in each of the sync and async wrappers (`except BaseException as e: exc = e; raise`), and it directly matters for this integration since "errors recorded on the trace" is part of the expected behavior.

### Proposed Solution
A `LangfuseBridge` class in `burr/integrations/langfuse.py` that subclasses `OpenTelemetryBridge` and is attached via `.with_hooks(LangfuseBridge())`. It delivers everything the issue asked for:
 
- **One trace per application execution call** (`run`/`step`/`iterate`/`stream_result`/...), rooted at the `pre_run_execute_call` hook.
- **One span per step**, with the action's inputs and read state captured as the Langfuse observation input, and its result and written state as the observation output (using Burr's `serde` for serialization, and filtering out internal dunder inputs like `__tracer`).
- **One span per tracer span** opened through Burr's `__tracer` API, inherited from the parent bridge.
- **Observations at will**: attributes logged via `__tracer.log_attributes()` land as Langfuse observation metadata (with `langfuse.*` and `gen_ai.*` keys passed through unprefixed so users can deliberately set mapped attributes).
- **Access to the Langfuse client** via `bridge.langfuse_client`, e.g. for `flush()` in short-lived scripts or for scoring traces.
Two Burr-to-Langfuse mappings fell out naturally: `app_id` maps to the Langfuse session (so multiple execution calls of one application group together) and `partition_key` maps to the Langfuse user, both overridable via constructor args. These attributes are set on every span, not just the root, because Langfuse's session/user aggregations operate per-observation. A `capture_state=False` flag turns off state capture for applications with sensitive state, and `burr_span_export_filter` is exported publicly for users who construct their own `Langfuse` client. Because everything rides on the global OTel provider, any third-party LLM instrumentor (e.g. `opentelemetry-instrumentation-openai`) automatically appears as generations nested inside the correct Burr step spans, with prompts, completions, and token usage.
 
The core `application.py` exception-propagation fix is included with its own regression tests, clearly flagged for the maintainer since it touches core and he may prefer it split into a separate small PR.
 
### Implementation Plan
Using UMPIRE framework (adapted):
**Understand:** Burr needs a hooks-based integration that maps its execution model onto Langfuse's trace model: execution call → trace, step → span, tracer span → nested span, state I/O → observation input/output, logged attributes → metadata, with the Langfuse client accessible and errors recorded.
**Match:** `OpenTelemetryBridge` in `burr/integrations/opentelemetry.py` already solves the hard span-lifecycle problems (context token stack, enter/exit pairing across hooks, error status). The traceloop integration shows the docs pattern for OTel-vendor pages; `require_plugin` in `burr/integrations/base.py` is the standard optional-dependency guard; `tests/integrations/test_burr_opentelemetry.py` sets the test style; `examples/integrations/` is the home for integration examples (exempt from the notebook/statemachine.png requirements in `validate_examples.py`).
**Plan:**
1. Add `burr/integrations/langfuse.py` with `LangfuseBridge(OpenTelemetryBridge)`, `burr_span_export_filter`, and serialization helpers.
2. Override `pre/post_run_execute_call`, `pre/post_run_step`, `pre_start_span`, and `do_log_attributes` to layer Langfuse attributes (`langfuse.observation.input/output`, `langfuse.observation.metadata.*`, `langfuse.session.id`, `langfuse.user.id`) onto the parent's span handling.
3. Fix the `exc = None` bug in both wrappers of `_call_execute_method_pre_post` in `burr/core/application.py`.
4. Add `tests/integrations/test_burr_langfuse.py` plus two regression tests in `tests/core/test_application.py`.
5. Add the `langfuse` extra to `pyproject.toml` (and to the `tests` extra), `docs/reference/integrations/langfuse.rst` wired into the integrations index, and a runnable example under `examples/integrations/langfuse/`.
**Implement:** Complete locally; patch prepared against main. [Link to branch/commits once pushed to fork.]
**Review:** Formatted and linted with the repo's pre-commit tooling (black 23.11 at line length 100, isort with the black profile, flake8) — all clean. ASF license headers on every new file. Docs page follows the traceloop/opentelemetry RST structure. Example placed where `validate_examples.py` permits a README + application.py without a notebook.
**Evaluate:** Automated tests assert the full span tree and attributes against an in-memory OTel exporter (no credentials needed); manual verification runs the example against a free Langfuse Cloud project to confirm the trace renders correctly in their UI, since the automated tests mock the Langfuse client itself.
---

## Testing Strategy

### Unit Tests
- [x] `test_burr_span_export_filter`: Burr-scoped spans pass the export filter, and spans matching Langfuse's default criteria (e.g. `gen_ai.*`) still pass.
- [x] `test_langfuse_bridge_rejects_client_and_kwargs`: passing both a pre-constructed client and constructor kwargs raises `ValueError`.
- [x] Core regression (sync + async): `post_run_execute_call` receives the actual exception when a step raises (`test_app_step_broken_calls_post_run_execute_call_with_exception` and the `astep` variant in `tests/core/test_application.py`).
### Integration Tests
All run a real two-action Burr app (with a tracer span and logged attributes) against an in-memory OTel `TracerProvider`/`InMemorySpanExporter` with a mocked Langfuse client, so no network or credentials are needed in CI:
- [x] `test_langfuse_bridge_span_structure_and_attributes`: one root span per `run` call, step spans parented to the root, tracer span parented to its step, all in one trace; session/user attributes on every span; step observation input contains action inputs and read state (with `__tracer` filtered out); step output contains result and written state; trace-level output reflects final application state; `__tracer.log_attributes` lands in observation metadata.
- [x] `test_langfuse_bridge_capture_state_false`: no observation input/output attributes anywhere when state capture is disabled.
- [x] `test_langfuse_bridge_session_and_user_overrides`: explicit `session_id`/`user_id` override the `app_id`/`partition_key` defaults.
- [x] `test_langfuse_bridge_records_exceptions`: a raising action marks both the step span and the root span as errored (this is what surfaced the core bug).
Results: 6/6 new integration tests pass; 400/400 across `tests/core`, `tests/visibility`, and the OTel/pydantic integration tests (including 139 in `tests/core/test_application.py` with the two new regression tests).
### Manual Testing
In progress. The automated tests exercise the bridge against a real OTel pipeline but mock the Langfuse client, so they can't confirm how Langfuse's backend ingests and renders the spans. I'm running `examples/integrations/langfuse/application.py` against a free Langfuse Cloud project (with `opentelemetry-instrumentation-openai` installed) to verify: (1) one trace appears per `.run()` call, (2) the trace tree shows steps → tracer spans → OpenAI generations nested correctly, (3) observation input/output and metadata render as expected, (4) the session and user views group traces by `app_id`/`partition_key`, and (5) a forced error shows up on the trace. Screenshots of the Langfuse trace view will go here for the PR description.
---

## Implementation Notes

### May 31 — Scoping
Commented on the issue proposing the OTel-based direction instead of the original dedicated-client approach (which depended on the closed #205), and scoped a focused first PR: one trace per run, one span per step, state observations, plus example and docs, with follow-ups for anything beyond. Maintainer (@elijahbenizzy) replied same day: "Can work both ways -- up to you," and confirmed we can get around #205.

### July 1-12 — Implementation
Surveyed the codebase (`OpenTelemetryBridge`, lifecycle hook signatures, the traceloop docs pattern, test conventions, `validate_examples.py` rules) and verified the current Langfuse SDK against the real package rather than docs — which is how I caught that the SDK is now v4.14 (docs and the issue thread still talk about v3) and that v4's default export filter would silently drop Burr spans. Implemented `LangfuseBridge`, the export filter, tests, docs, the example, and the pyproject extra. Testing error propagation surfaced the `post_run_execute_call` exception bug in core; fixed it in both wrappers with regression tests. First test run also caught that Burr injects `__tracer` into action inputs, which was leaking framework internals into the serialized observation input — added dunder filtering. All formatting/linting clean per the repo's pre-commit config.

### Next
Manual verification against Langfuse Cloud (in progress), then push the branch, open the PR with the core fix clearly flagged (offering to split it out if preferred), and update the issue thread noting the v4 SDK landscape.

### Code Changes
- **Files added:** `burr/integrations/langfuse.py`, `tests/integrations/test_burr_langfuse.py`, `docs/reference/integrations/langfuse.rst`, `examples/integrations/langfuse/{__init__.py,application.py,README.md}`
- **Files modified:** `burr/core/application.py` (exception propagation fix), `tests/core/test_application.py` (2 regression tests), `pyproject.toml` (`langfuse` extra + tests extra), `docs/reference/integrations/index.rst` (toctree entry)
- **Key commits:** [Links once pushed to fork]
- **Approach decisions:**
  - *Subclass `OpenTelemetryBridge` instead of a standalone client:* the SDK's OTel-native rewrite means Langfuse ingests standard OTel spans off the global provider, so a parallel client following `tracking/client.py` would duplicate the bridge's span-lifecycle machinery and lose free nesting of third-party LLM instrumentor spans.
  - *Inject `should_export_span` rather than spoof the Langfuse tracer scope:* naming our tracer `langfuse-sdk` would pass the filter but misrepresent the spans to Langfuse's mapping layer; extending the filter is the documented, public mechanism, and only relying on public API (I deliberately avoided the private `_otel_span` on Langfuse's span wrappers).
  - *`app_id` → session, `partition_key` → user:* Burr's own docs treat partition_key as a per-user key, and sessions grouping multiple runs of one app instance matches Langfuse's session semantics; both are constructor-overridable, and set on every span because Langfuse aggregates these per-observation.
  - *`capture_state` flag:* state can contain sensitive data; the integration shouldn't force users to ship it to a third party to get tracing.
  - *Include the core fix in this PR but flag it:* it's 8 lines plus tests and is required for correct error status on traces, but it touches `application.py`, so the maintainer gets an explicit offer to split it into its own PR.
---
## Pull Request
**PR Link:** [GitHub PR URL when submitted]
**PR Description (draft):**
> Implements #206: a Langfuse integration built on the OpenTelemetry-native Langfuse Python SDK, as discussed in the issue thread. Adds `LangfuseBridge` (subclassing `OpenTelemetryBridge`, attached via `with_hooks`) providing one trace per execution call, one span per step, one span per tracer span, state/inputs/results as observation input/output, logged attributes as observation metadata, and public access to the Langfuse client. `app_id`/`partition_key` map to Langfuse session/user (overridable); `capture_state=False` disables state capture. Includes tests, docs, an example, and a `langfuse` extra.
>
> Two things reviewers should know: (1) Langfuse SDK v4 filters span export to LLM-relevant spans by default, so the bridge injects a `should_export_span` filter (exported as `burr_span_export_filter` for users constructing their own client), with a fallback for v3. (2) This PR includes a small core fix: `_call_execute_method_pre_post` never captured the raised exception, so `post_run_execute_call` hooks always received `exception=None` — meaning the existing OTel bridge never marked failed runs as errored. Fixed in both sync/async wrappers with regression tests. Happy to split this into a separate PR if preferred.
 
**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]
**Status:** Preparing PR (manual verification in progress)
---
## Learnings & Reflections
### Technical Skills Gained
- How OpenTelemetry context propagation actually works under the hood: tracer providers, span processors, export filters, and the token/attach/detach mechanics Burr's bridge uses to pair spans across non-context-manager hook boundaries.
- Reading a dependency's source instead of trusting its docs: the v4 export filter behavior and the exact `should_export_span` mechanism weren't findable from documentation alone, and pip-installing the SDK to inspect `_client/span_filter.py` and the `Langfuse` constructor signature resolved in minutes what could have been a mystifying "no traces appear" failure later.
- Langfuse's OTel attribute mapping (`langfuse.observation.input/output`, `langfuse.observation.metadata.*`, `langfuse.session.id`, `langfuse.user.id`) and the per-observation attribute requirement for session/user aggregations.
- Testing tracing integrations hermetically with `InMemorySpanExporter` and an isolated `TracerProvider`, asserting on the full span tree rather than just "it didn't crash."
### Challenges Overcome
- The design target had drifted since 2024: the original approach depended on a closed issue and predated Langfuse's OTel rewrite. Re-scoping on the issue thread before writing code, and getting maintainer sign-off on the new direction, meant the implementation didn't fight the current SDK.
- A failing exception test turned out to be a bug in Burr core, not my code. Tracing it from "root span shows OK" through the hook invocation path to the `try/finally` with no `except` was a good exercise in not assuming the framework is right, and in deciding how a first-time contributor should handle a core fix inside a feature PR (include it, test it, flag it, offer to split).
- My own first version leaked Burr's internal `__tracer` object into the serialized observation input — caught by the very first test run, which reinforced writing the assertions on exact payload contents.
### What I'd Do Differently Next Time
Verify the current state of the external dependency *before* scoping the work in the issue comment, not after: my May 31 comment said "Langfuse now ingests OpenTelemetry natively" based on v3, and the v4 filter behavior would have been worth mentioning then. Also, I'd start from the framework's existing tests earlier — reading `test_burr_opentelemetry.py` first would have saved some time reverse-engineering hook signatures from `lifecycle/base.py`.
---

## Resources Used
- [Burr hooks concept docs](https://burr.dagworks.io/concepts/hooks/)
- [Burr OpenTelemetry integration reference](https://burr.apache.org/reference/integrations/opentelemetry/) and `examples/opentelemetry/`
- [Langfuse OpenTelemetry integration guide](https://langfuse.com/integrations/native/opentelemetry) (attribute mapping)
- [Langfuse SDK overview](https://langfuse.com/docs/observability/sdk/overview) and [existing-OTel-setup FAQ](https://langfuse.com/faq/all/existing-otel-setup) (export filtering, global tracer provider)
- [Langfuse Python SDK v3 announcement](https://langfuse.com/changelog/2025-05-23-otel-based-python-sdk) — context for the OTel rewrite
- Langfuse SDK source (`langfuse/_client/span_filter.py`, client constructor) via local install — the export-filter discovery
- [Issue #206](https://github.com/apache/burr/issues/206), [#205 (closed)](https://github.com/apache/burr/issues/205), [#203](https://github.com/apache/burr/issues/203)
- Hamilton's Datadog plugin (`h_ddog.py`), referenced in the issue as inspiration for the original design











