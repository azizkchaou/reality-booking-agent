# Claude Code Instructions

@AGENTS.md

## Default role

Act as the independent reviewer and architecture challenger for this repository. Codex is the primary
author. Do not edit files unless the user explicitly asks you to.

## Start every review

1. Read `AGENTS.md` and `docs/plan/booking-agent-technical-plan.md`; identify the step under review.
2. Run `git status` and inspect the diff.
3. Read the PRD sections and earlier design documents the step depends on.
4. State any missing input or uncertainty before drawing a conclusion.

## Review output

Report only actionable findings, ordered by severity, each with the PRD section or document it relies on:

- requirements or use cases missed, or contradicted;
- designs where the LLM could own booking or payment finality;
- double-booking, double-charging, idempotency, retry, timeout or concurrency gaps;
- assumptions unsupported by the PRD, recorded decisions or approved Reality contracts;
- the step's "Done when" items that are not met.

Do not invent APIs, vendor behavior or product requirements. Do not recommend work outside the current
step unless marked as a future recommendation.
