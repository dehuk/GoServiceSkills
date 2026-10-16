---
description: Restate requirements, assess risks, and produce a step-by-step implementation plan for a Go service change using the planner agent. Waits for user confirmation before any code is written.
argument-hint: "[feature description | path/to/requirements.md]"
---

# Plan

Produce an implementation plan for: $ARGUMENTS

## Steps

1. If `$ARGUMENTS` is empty, ask the user what should be planned and stop.
2. If `$ARGUMENTS` is a path to a file, read it and use it as the requirements source. Otherwise treat it as a free-form feature description.
3. Delegate to the **planner** agent with the requirements. If the planner agent is unavailable, plan inline following the same structure.
   The plan must:
   - Restate the requirements and list assumptions and open questions
   - Ground itself in the codebase: cite existing handlers, use cases, repositories, migrations, NATS publishers/consumers, and tests as `path:line` patterns to mirror; say explicitly when no precedent exists
   - Break the work into phases with file-level steps, dependencies, and risk (Low / Medium / High)
   - Cover transaction boundaries, event publishing, idempotency, and error-to-status mapping where relevant
   - Include a testing strategy and the verification commands (`go build ./...`, `go test -race ./...`, `golangci-lint run`)
   - Estimate overall complexity (Low / Medium / High)
4. If the change involves a major structural decision (new service, data ownership, sync vs async integration), say so and suggest consulting the **architect** agent before implementation.
5. Present the plan and end with:

   **Waiting for confirmation**: proceed with this plan? (yes / no / modify)

## Confirmation Gate

Do not create, edit, or delete any source files until the user explicitly approves the plan (for example "yes", "proceed", "go ahead"). Planning is cheap to change; code written against a misunderstood requirement is not.

If the user asks for changes ("modify: ...", "different approach: ...", "skip phase 2"), revise the plan and wait for confirmation again.

After approval, implement phase by phase, keeping the build green and running `go test -race ./...` after each phase. Suggest `/go-review` when implementation is complete.
