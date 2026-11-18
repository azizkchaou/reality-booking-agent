# Reality AI Booking & Matchmaking Agent - Working Guide

Read this file fully before every task.

## Purpose and current state

This repository holds the design (and later the code) for Reality's **AI Booking & Matchmaking Agent**,
specified in `docs/source/Reality AI Agent 1.pdf` (PRD v1.1, 8 Nov 2026).

- The feature is **decided and will be built**. Do not argue whether to build it.
- Current stage: **design**. No production code until plan Phase 7 (spikes) is done.
- The active plan is `docs/plan/booking-agent-technical-plan.md`. Work only on the step you are asked
  for, then stop.

## Context

- Reality is a sports app: players find nearby facilities, book courts, organize or join games, and talk
  in club communities (chat).
- This feature is SEPARATE from Reality's existing "Matchmaking Agent" (a Python service that reads
  community-chat posts and turns them into match actions). Do not merge the two designs. Its guide and
  roadmap are in `docs/source/matchmaking-agent/` for conventions only.
- Scenarios: A = book a court only; B = organizer already has a court, find X players (B.a = auto-confirm
  from pre-consented standing intents; B.b = push invitations, confirm only after every invitee accepts
  and pays); C = find players AND book a court.
- Markets: Tunisia first (refundable deposit charged BEFORE any external AI call; matchmaking free);
  United States later (card authorization hold BEFORE any external AI call; joined players pay a
  technology convenience fee). Reality pays the facility and collects player balances.
- Core rule: the LLM never owns transaction finality. Bookings and payments become final only through
  verified events (payment-provider webhooks, partner API responses, verified facility evidence).
- Company conventions (from the Matchmaking Agent): Reality's backend owns all domain data and
  mutations; AI services never touch Reality's database directly and call approved Reality internal APIs;
  the model does bounded interpretation and deterministic code decides allowed actions.

## Rules

- Cite the PRD section (e.g. "PRD §4.3") for every requirement you state.
- Never invent Reality APIs, DTOs, database fields, vendor capabilities, secrets or product decisions.
  If something is needed but unknown, write "ASSUMPTION:" or add it to `docs/design/00-open-questions.md`.
- When the PRD is ambiguous or contradicts itself, say so and quote both places.
- Use tables and numbered IDs so later steps can reference your output.
- Markdown only; diagrams in Mermaid.
- Vendor facts (payments, telephony, voice, WhatsApp) come from official documentation with URL and date,
  never from memory.

## Source of truth (when sources disagree)

1. `docs/design/00-decisions.md` (recorded decisions, incl. founder answers).
2. Approved Reality backend contracts (when provided).
3. The PRD v1.1.
4. Conventions in `docs/source/matchmaking-agent/` (patterns only, never contracts for this feature).

## Workflow

- One plan step per task. Save the output to the path the step names under `docs/design/`.
- Check the step's "Done when" list and update the plan's Progress table.
- One commit per step: `design(step X.Y): <short title>`.
- Claude reviews each step's output (see `CLAUDE.md`). Fix review findings before the next step.

## Confidentiality

This repository is public. `docs/source/` is git-ignored and must never be committed or quoted at length
in committed files. Do not commit PRD text verbatim (e.g. `docs/design/01-prd.md`) while the repository
is public; keep it local or make the repository private first.
