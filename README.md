# Contribution #1: Langfuse integration

**Contribution Number:** 1

**Student:** Han Nguyen  

**Issue:** [\[GitHub issue link\]  ](https://github.com/apache/burr/issues/206#event-26164144705)

**Status:** Phase 1 - Complete. Phase 2 - In progress. Phase 3 - Have not started.

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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

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

