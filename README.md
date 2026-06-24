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

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]

