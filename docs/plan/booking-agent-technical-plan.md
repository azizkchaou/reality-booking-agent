# Technical plan — Reality AI Booking & Matchmaking Agent
Source spec: *Reality AI Booking & Matchmaking Agent — PRD & Feasibility Assessment v1.1 (8 Nov 2026)* (`Reality AI Agent 1.pdf`)
Owner: Aziz (technical lead) · Product decisions: Founder/CEO · Status: **DRAFT — awaiting approval**
Audience: Aziz and any LLM (Claude, GPT/Codex, Gemini…) working on this feature. Every prompt below is self-contained.

---

## How to use this plan (for humans and LLMs)

1. Work **one step at a time**, in order. A step may start only when everything in its `Blocked by` line is done.
2. For each step: paste the **Shared context block** (below), then the step's **Prompt**, then attach the listed **Inputs**.
3. Save the output to the listed **Output** path under `docs/design/` in the feature repo (Decision D4). Commit each step separately. Outputs are Markdown; diagrams are Mermaid inside Markdown so any LLM can read and edit them.
4. Review: give the output to the reviewing LLM (Decision D6) with: *"Review this against the PRD and the step's Done-when list. Report only concrete problems, ordered by severity, each with the PRD section."* Fix, then check **Done when** and update the Progress table.
5. Any question or contradiction an LLM finds goes into `00-open-questions.md` — never silently resolved.

### Shared context block (paste at the top of every prompt)

```
CONTEXT
- Company: Reality — a sports app where players find nearby facilities, book courts, organize or join
  games, and talk in club communities (chat).
- Feature: "AI Booking & Matchmaking Agent", specified in the attached PRD v1.1. The feature is DECIDED
  and will be built. Do not argue whether to build it.
- It is SEPARATE from Reality's existing "Matchmaking Agent" (a Python service that reads community-chat
  posts and turns them into match actions). Do not merge the two designs.
- Scenarios: A = book a court only; B = organizer already has a court, find X players
  (B.a = auto-confirm from pre-consented standing intents; B.b = push invitations, confirm only after
  every invitee accepts and pays); C = find players AND book a court.
- Markets: Tunisia first (refundable deposit charged BEFORE any external AI call; matchmaking free),
  United States later (card authorization hold BEFORE any external AI call; joined players pay a
  technology convenience fee). Reality pays the facility and collects player balances.
- Core rule: the LLM never owns transaction finality. Bookings and payments become final only through
  verified events (payment-provider webhooks, partner API responses, verified facility evidence).
- Company architecture conventions (from the existing Matchmaking Agent): Reality's backend owns all
  domain data and mutations; AI services never touch Reality's database directly and call approved
  Reality internal APIs; the model does bounded interpretation and deterministic code decides allowed
  actions.
- My role: technical lead. I must understand every use case, design the best implementation, cover
  the needed use cases, and identify risks with concrete solutions.

RULES FOR YOU
- Cite the PRD section (e.g. "PRD §4.3") for every requirement you state.
- Never invent Reality APIs, database fields, vendor capabilities or product decisions. If something is
  needed but unknown, write it as "ASSUMPTION:" or add it to an "Open questions" list.
- When the PRD is ambiguous or contradicts itself, say so explicitly and quote both places.
- Prefer tables and numbered IDs so later steps can reference your output.
- Output Markdown only. Diagrams in Mermaid.
```

### Tools and skills — what to use where
| Need | Claude (skills / MCP) | Any other LLM |
| --- | --- | --- |
| Read the PRD PDF | Read tool / `anthropic-skills:pdf` | Use `01-prd.md` from step 1.1 (never the PDF) |
| Requirements, glossary | `zizi-skills:idea-refine` | Prompt as written |
| Architecture | `engineering:system-design`, `engineering:architecture`, `10x-Team:principal-architect`, `zizi-skills:service-architecture` | Ask for two alternative designs and a comparison table |
| Vendor / API facts | `zizi-skills:source-driven-development`, `zizi-skills:integrations`, WebSearch/WebFetch | Give it the official doc URLs; forbid answers from memory |
| Data model, locks | `zizi-skills:database-optimization`, `10x-Team:dba` | Prompt as written |
| Security | `zizi-skills:security-and-hardening`, `10x-Team:security-engineer` | Ask for a STRIDE threat model |
| Tests | `engineering:testing-strategy`, `10x-Team:qa-engineer`, `zizi-skills:tdd-skill` | Ask for Given/When/Then cases |
| Attack the design | `zizi-skills:adversarial-review` | Run the same review prompt on 2 different LLMs and diff the findings |
| Build plan, tickets | `zizi-skills:planning`, `zizi-skills:project-manager` | Prompt as written |
| Diagrams | Mermaid (render with `mmdc`) or a Whiteboard artifact | Mermaid |

**Cross-LLM trick:** for the design steps (Phase 4) and the risk review (5.3), run the prompt on two different models, then give both answers to a third run with: *"Compare these two answers. List where they agree, where they differ, and which is better-supported by the PRD. Produce a merged answer."*

---

## Phases

### Phase 0 — Set-up and founder answers
Verified when: the gate questions are answered and the design workspace exists with the PRD in Markdown.

#### Step 0.1: Get founder answers (Size: XS)
- Goal: remove blockers that no LLM can resolve.
- Ask the founder (record answers in `00-decisions.md`):
  1. MVP scope and order — recommended: follow PRD §11 (partner booking A → B → phone booking A → C).
  2. Tech stack and ownership — PRD §8 proposes Flutter → Spring Boot BFF → Java booking orchestrator. Is that confirmed? Which parts does Aziz build, which does the backend team build?
  3. Which Reality APIs exist today (facilities, partner availability/holds, bookings, matches, chat, notifications, payments), and who provides their contracts?
  4. Payment provider in Tunisia today; voice vendor (Twilio / Retell / other) chosen or open?
  5. Expected deliverable and deadline (design doc, prototype, production slice).
  6. How this feature and the existing Matchmaking Agent interact when both recruit the same player.
  7. LLM boundary: PRD §8.3 exposes ~22 tools to the LLM, while the company convention is bounded interpretation + deterministic decisions. Which model is intended?
- Done when: every question has an answer or an owner + date.
- Confirm-it-happened step: answers are written down and re-read to the founder (or sent in writing).
- Blocked by: none

#### Step 0.2: Create the design workspace and convert the PRD (Size: S)
- Goal: one shared place every LLM reads from; PRD in text form.
- Prompt:
  ```
  [SHARED CONTEXT]
  Convert the attached PRD into clean Markdown. Keep every section number, table and list exactly.
  Do not summarize or rephrase. At the end add a "Section index" table: section number, title,
  one-line description. Mark any text you could not read as [UNREADABLE].
  ```
- Inputs: `Reality AI Agent 1.pdf`
- Output: `01-prd.md`, plus empty `00-decisions.md`, `00-open-questions.md`, `00-glossary.md`
- Skills: Claude `anthropic-skills:pdf`; others: any PDF-to-text, then the prompt.
- Done when: a spot-check of 5 random tables matches the PDF; the appendix ("Prompt to explain the logic") is included.
- Blocked by: none

---

### Phase 1 — Understand the feature completely
Verified when: every PRD requirement and edge case maps to a use-case ID, and every gap is a written question.

#### Step 1.1: Glossary (Size: S)
- Prompt:
  ```
  [SHARED CONTEXT]
  From the PRD, build a glossary of every domain term (e.g. intent, standing intent, partner facility,
  external facility, authorization hold, refundable commitment deposit, facility settlement, participant
  court balance, technology convenience fee, CONFIRMED, PENDING_VERIFICATION, recruiting proposal,
  auto-join consent, cutoff). For each: definition in one sentence, PRD section, market differences
  (TN vs US), and terms it is commonly confused with. Then list terms the PRD uses inconsistently.
  ```
- Inputs: `01-prd.md`
- Output: `00-glossary.md`
- Skills: `zizi-skills:idea-refine`
- Done when: ≥ 25 terms; every term has a PRD reference; inconsistencies listed.
- Blocked by: 0.2

#### Step 1.2: Actors and systems inventory (Size: S)
- Prompt:
  ```
  [SHARED CONTEXT]
  List every actor (human and system) in the PRD: organizer, joining player, invited player, partner
  facility, external facility, facility receptionist, Reality backend, booking orchestrator, payment
  provider, voice gateway, notification service, admin/ops reviewer, LLM. For each: what it can do,
  what it must never do, which scenarios it appears in, and what data it owns. Output a table and a
  Mermaid context diagram.
  ```
- Inputs: `01-prd.md`, `00-glossary.md`
- Output: `02-actors-and-systems.md`
- Skills: `engineering:system-design`
- Done when: each actor has "can / must never / owns"; diagram renders.
- Blocked by: 1.1

#### Step 1.3: Use-case catalog (Size: M)
- Prompt:
  ```
  [SHARED CONTEXT]
  Build a complete use-case catalog. Use IDs: UC-A-a1, UC-A-a2, UC-A-b1, UC-A-b2, UC-B-a, UC-B-b,
  UC-C, plus cross-cutting ones (UC-X-consent, UC-X-refund, UC-X-chat, UC-X-cancel, UC-X-admin, ...).
  For each use case give: trigger, preconditions, main flow (numbered), alternative flows, failure
  flows, payment behaviour per market (TN / US), resulting states (PRD §7.1 names), notifications,
  chat behaviour, and PRD references. Include partner vs external facility variants where they differ.
  Use the PRD appendix ("Prompt to explain the logic") as the founder's original intent and point out
  where the formal PRD sections differ from it.
  ```
- Inputs: `01-prd.md`, `00-glossary.md`, `02-actors-and-systems.md`
- Output: `03-use-cases.md`
- Skills: `zizi-skills:idea-refine`, `10x-Team:product-manager`
- Done when: every scenario branch in PRD §4.2, §5, §6 has a use case; each has ≥ 1 failure flow.
- Blocked by: 1.2

#### Step 1.4: Coverage matrix and gap list (Size: S)
- Prompt:
  ```
  [SHARED CONTEXT]
  Build a coverage matrix. Rows: every requirement, rule, edge case and acceptance case in the PRD
  (§3 rules, §4.3, §4.4, §5.2, §5.3, §6.2–6.5, §7, §9, §13). Columns: requirement text, PRD section,
  use-case ID(s) from 03-use-cases.md that cover it, status (covered / partly / missing).
  Then list: (a) requirements not covered, (b) use-case steps with no PRD basis, (c) contradictions
  inside the PRD, (d) questions only the founder can answer. Add (c) and (d) to the open-questions file.
  ```
- Inputs: `01-prd.md`, `03-use-cases.md`
- Output: `04-coverage-matrix.md`; append to `00-open-questions.md`
- Skills: `engineering:testing-strategy`
- Done when: no row is blank; every "missing" row has a follow-up.
- Note: known example to check — PRD §11 Stage 1 is titled "Partner booking A" but its scope text describes the two B pathways.
- Blocked by: 1.3

#### Step 1.5: Walkthrough with a human (Size: XS)
- Goal: Aziz (and ideally the founder) reads `03-use-cases.md` and `00-open-questions.md` and answers what they can.
- Done when: open questions are answered, assigned, or marked "assume X" with sign-off.
- Blocked by: 1.4

---

### Phase 2 — Ground truth on what already exists
Verified when: every external dependency the design relies on has a fact sheet with sources.

#### Step 2.1: Reality capability inventory (Size: S)
- Goal: know which Reality APIs/data the agent can rely on.
- Prompt:
  ```
  [SHARED CONTEXT]
  Here are the Reality backend documents/contracts I have [attach]. List every capability the new
  agent needs (from 03-use-cases.md) and mark each: EXISTS (cite the doc/endpoint), PARTIAL, MISSING,
  or UNKNOWN. For MISSING/UNKNOWN, write the exact question to send to the backend team.
  Do not invent endpoints.
  ```
- Inputs: Reality API docs/contracts from the backend team; `03-use-cases.md`
- Output: `05-reality-capabilities.md`
- Skills: `zizi-skills:source-driven-development`
- Done when: each capability has a status and a source or a question.
- Confirm-it-happened step: questions are actually sent to the backend team and answers recorded.
- Blocked by: 0.1, 1.3

#### Step 2.2: Vendor fact sheets (Size: M — one sub-step per vendor)
- Sub-steps: 2.2a Tunisian payment provider (charge + refund, refund delay, webhooks, fees); 2.2b Stripe authorize/capture for US holds (hold duration, re-authorization, void); 2.2c Telephony in Tunisia (Twilio guidelines, caller ID, local numbers); 2.2d Voice-agent platform (Retell or alternative: latency, languages incl. French/Arabic, pricing, webhooks, transcripts); 2.2e WhatsApp Business API as an alternative channel to facilities.
- Prompt (one per vendor):
  ```
  [SHARED CONTEXT]
  Using ONLY the official documentation pages I provide (or that you fetch from the vendor's own site),
  write a fact sheet for [VENDOR] covering what our design needs: [list from use cases]. For every
  fact give the source URL and the date checked. List limits and costs. End with "Impact on our
  design" and "Unknowns to confirm with the vendor". Do not answer from memory.
  ```
- Inputs: PRD §14 links; vendor docs
- Output: `06-vendors/<vendor>.md`
- Skills: `zizi-skills:integrations`, `zizi-skills:source-driven-development`, WebSearch/WebFetch
- Done when: every fact has a URL; unknowns listed.
- Blocked by: 0.1 (to know which vendors are in scope)

#### Step 2.3: Patterns to reuse from the existing Matchmaking Agent (Size: XS)
- Goal: reuse *conventions*, not code — event transport, bounded interpretation, provider fallback, idempotency, audit.
- Prompt:
  ```
  [SHARED CONTEXT]
  Here is the architecture guide of Reality's existing Matchmaking Agent [attach AGENTS.md and roadmap].
  List the conventions the new feature should follow (boundaries, events, LLM provider strategy,
  idempotency, audit, testing) and any that do not fit this feature, with reasons. Do not propose
  merging the two services.
  ```
- Output: `07-conventions.md`
- Blocked by: none

---

### Phase 3 — Model the domain
Verified when: every use case can be traced through events, state transitions and a sequence diagram.

#### Step 3.1: Event storming (Size: S)
- Prompt:
  ```
  [SHARED CONTEXT]
  List every domain event in time order for each use case (e.g. IntentCaptured, ConsentGiven,
  DepositCharged, HoldAuthorized, PartnerSlotHeld, CallStarted, FacilityConfirmed, EvidenceVerified,
  BookingConfirmed, InvitationSent, InvitationAccepted, FeeCollected, CutoffReached, HoldVoided,
  DepositRefunded, FacilityPaid...). For each event: producer, consumers, payload (fields), what happens
  if it arrives twice, arrives late, arrives out of order, or never arrives. Flag every event whose
  "never arrives" case needs a timeout job.
  ```
- Inputs: `03-use-cases.md`, `00-glossary.md`
- Output: `08-events.md`
- Skills: `engineering:system-design`
- Done when: every event has the four failure answers.
- Blocked by: 1.4

#### Step 3.2: State machines (Size: M)
- Prompt:
  ```
  [SHARED CONTEXT]
  For each aggregate in PRD §7.1 (BookingIntent, CourtReservation, Match, PlayerInvitation, Payment
  incl. TN deposit, US hold, PlayerBalance, FacilityPayable), produce a Mermaid stateDiagram and a
  transition table: from, to, triggering event, guard condition, side effects, who may trigger it
  (LLM never triggers a final state), and timeouts. List illegal transitions explicitly. Show how the
  machines interact in Scenario C (which state of one gates another).
  ```
- Inputs: `08-events.md`, PRD §7
- Output: `09-state-machines.md`
- Skills: `engineering:architecture`
- Done when: every event in 08 appears as a trigger; every state has an exit (including timeouts).
- Blocked by: 3.1

#### Step 3.3: Sequence diagrams (Size: M)
- Prompt:
  ```
  [SHARED CONTEXT]
  Draw Mermaid sequence diagrams for: (1) A with partner facility, (2) A with external facility via
  AI call, (3) B.a auto-confirm, (4) B.b push-to-fill, (5) C players-first with external facility,
  (6) C court-first with partner hold, (7) refund/void after failure. Participants: Player app, LLM
  agent, Orchestrator, Reality backend, Payment provider, Voice gateway, Facility, Notifications.
  Mark where a payment commitment must exist before a call, and where idempotency keys are used.
  ```
- Inputs: `03-use-cases.md`, `09-state-machines.md`
- Output: `10-sequences.md`
- Done when: 7 diagrams render; each step matches a state transition in 09.
- Blocked by: 3.2

#### Step 3.4: Data model (Size: S)
- Prompt:
  ```
  [SHARED CONTEXT]
  Starting from PRD §8.2, design the data model for the new feature. For each entity: fields, types,
  keys, indexes, uniqueness constraints that prevent double booking / double charge, and which fields
  reference Reality-owned entities (by ID only — Reality owns those). Money in minor units + ISO
  currency; times in UTC + facility IANA timezone (PRD §3). Mark what must be added on Reality's side
  vs owned by the new service.
  ```
- Inputs: `09-state-machines.md`, `05-reality-capabilities.md`
- Output: `11-data-model.md`
- Skills: `zizi-skills:database-optimization`, `10x-Team:dba`
- Done when: every state machine maps to a table/state column; every idempotency rule maps to a constraint.
- Blocked by: 3.2, 2.1

---

### Phase 4 — Architecture and component design
Verified when: a chosen architecture document exists with alternatives compared and decisions logged.

#### Step 4.1: Overall architecture — designed twice (Size: M)
- Prompt (run on two different LLMs, then merge with the cross-LLM trick):
  ```
  [SHARED CONTEXT]
  Propose TWO architectures for the booking orchestrator and its components (conversation agent,
  orchestrator, matching engine, payment adapter, voice/WhatsApp gateway, notifications, admin console):
  Option 1: durable workflow engine (long-running workflows with timers and retries).
  Option 2: database state machines + transactional outbox + scheduled jobs.
  For each: component diagram (Mermaid), how timeouts/retries/compensation work, how double booking
  and double charging are prevented, failure recovery after a crash, operational cost, team skill fit
  with the confirmed stack. Finish with a comparison table and a recommendation.
  ```
- Inputs: `09`, `10`, `11`, `07-conventions.md`, `00-decisions.md`
- Output: `12-architecture.md` (+ decision in `00-decisions.md`)
- Skills: `engineering:system-design`, `10x-Team:principal-architect`, `zizi-skills:service-architecture`
- Done when: both options cover all 7 sequences; a decision is recorded with reasons.
- Blocked by: 3.3, 3.4, 0.1

#### Step 4.2: Conversation agent / LLM boundary (Size: M)
- Prompt:
  ```
  [SHARED CONTEXT]
  Design the player-facing conversation agent. Cover: slot filling for intent normalization (PRD §3),
  multilingual input (French, Tunisian Arabic, English, mixes), how consent and preference toggles are
  captured and stored (PRD §3), which tools the LLM may call (start from PRD §8.3) and which must be
  server-side only, how every side-effect tool is validated server-side, prompt-injection risks (incl.
  from facility speech), and the fallback to a human. Compare "LLM with tools" vs "LLM extracts a
  structured request, deterministic code acts". Recommend one, with tool JSON schemas.
  ```
- Output: `13-conversation-agent.md`
- Skills: `zizi-skills:security-and-hardening`, `engineering:architecture`
- Done when: every tool has a JSON schema, an authorization rule and an idempotency rule.
- Blocked by: 4.1

#### Step 4.3: Payments and ledger design (Size: M)
- Prompt:
  ```
  [SHARED CONTEXT]
  Design the payment adapter and ledger for both markets (PRD §2, §6 "Pre-call commitment vs facility
  settlement", §7.2): TN refundable deposit, US authorization hold (capture/void/expiry/re-auth),
  participant court balances, technology convenience fee, facility payable/settlement, points/credits.
  Include ledger entries per use case, webhook handling (signature, replay, ordering), idempotency,
  reconciliation jobs, refund failures, chargebacks. Use only facts from 06-vendors/.
  ```
- Output: `14-payments.md`
- Skills: `zizi-skills:integrations`, `zizi-skills:security-and-hardening`
- Done when: each use case's money flow is a ledger table that balances; each webhook has a replay test.
- Blocked by: 4.1, 2.2

#### Step 4.4: Facility channel design — voice and WhatsApp (Size: M)
- Prompt:
  ```
  [SHARED CONTEXT]
  Design the facility-contact gateway (PRD §4.3, §4.4, §9). Cover: call/message script, AI disclosure,
  opt-in and do-not-call rules, calling hours, what counts as confirmation evidence (PRD §9 last
  bullet), confidence thresholds, PENDING_VERIFICATION handling, human review queue, dropped calls,
  voicemail, facility proposing different time/price, two parallel calls finding slots. Compare voice
  vs WhatsApp for Tunisia. Include a Darija benchmark plan (dataset, metrics, pass threshold).
  ```
- Output: `15-facility-channel.md`
- Done when: every PRD §4.4 edge case has a handling rule.
- Blocked by: 4.1, 2.2

#### Step 4.5: Matching engine design (Size: M)
- Prompt:
  ```
  [SHARED CONTEXT]
  Design the matching engine for B.a and B.b and its use in C (PRD §5.2, §5.3, §6.2, §6.3).
  Cover: standing-intent storage and expiry, eligibility filters, scoring, atomic claim of intents
  (no double commitment), invitation scheduling with rate limits and quiet hours, accept-by deadlines,
  cutoff → EXPIRED/INCOMPLETE, sport-specific capacity (never hardcode 4), competing matches for the
  same player. Give the concurrency strategy (locks/transactions) with pseudo-code.
  ```
- Output: `16-matching.md`
- Skills: `zizi-skills:database-optimization`
- Done when: race conditions listed with the exact mechanism that prevents each.
- Blocked by: 4.1, 3.4

#### Step 4.6: API and event contracts (Size: S)
- Prompt:
  ```
  [SHARED CONTEXT]
  From 12–16, write the contracts: OpenAPI for the new service's endpoints, JSON schemas for events,
  and the list of Reality APIs we call (from 05, marked EXISTS/NEEDED). Every mutating call has an
  idempotency key. Mark every contract that needs backend-team approval.
  ```
- Output: `17-contracts.md`
- Done when: every sequence step in 10 maps to a contract.
- Blocked by: 4.2, 4.3, 4.4, 4.5

---

### Phase 5 — Risks and hard problems
Verified when: every high risk has a mitigation, an owner and a test.

#### Step 5.1: Risk register (Size: S)
- Prompt:
  ```
  [SHARED CONTEXT]
  Build a risk register for this design: technical (concurrency, idempotency, workflow durability,
  voice accuracy in Darija, vendor limits), payments (refund delays, hold expiry, chargebacks,
  unrecovered balances), operational (human review load, facility opt-out), legal/compliance
  (Tunisia Law 2004-63, call recording, US TCPA — flag for counsel, do not give legal conclusions),
  product (low player liquidity). For each: likelihood, impact, early warning signal, mitigation in
  our design (cite the doc), residual risk, owner.
  ```
- Output: `18-risks.md`
- Blocked by: 4.6

#### Step 5.2: Threat model (Size: S)
- Prompt:
  ```
  [SHARED CONTEXT]
  Do a STRIDE threat model of the design: card data never reaching the LLM, webhook spoofing, prompt
  injection from users and from facility speech, abuse of outbound calls, privacy of players before
  acceptance (PRD §2 chat rule), audit trail integrity. Give a mitigation per threat.
  ```
- Output: `19-threat-model.md`
- Skills: `zizi-skills:security-and-hardening`, `10x-Team:security-engineer`
- Blocked by: 4.6

#### Step 5.3: Adversarial design review (Size: M)
- Prompt (run on ≥ 2 different LLMs; merge findings):
  ```
  [SHARED CONTEXT]
  You are a hostile senior reviewer. Try to break this design [attach 09–17]. Find scenarios where:
  a player is charged without a booking, a facility is booked twice or paid twice, a match is
  CONFIRMED without a real court or enough committed players, a hold expires mid-flow, the system
  calls a facility without a commitment, or a crash leaves money or a court in limbo. For each: exact
  sequence of events, severity, and the fix.
  ```
- Output: `20-design-review.md`; fixes applied to 09–17
- Skills: `zizi-skills:adversarial-review`
- Done when: every critical finding is fixed or accepted in `00-decisions.md`.
- Blocked by: 5.1, 5.2

---

### Phase 6 — Test strategy
Verified when: every use case and every PRD §13 case has an automated test definition.

#### Step 6.1: Acceptance test catalog (Size: M)
- Prompt:
  ```
  [SHARED CONTEXT]
  Write acceptance tests in Given/When/Then for: every PRD §13 case, every failure flow in 03, every
  risk in 18 and finding in 20. Tag each with use-case ID, market (TN/US), and level (unit /
  integration / end-to-end).
  ```
- Output: `21-acceptance-tests.md`
- Skills: `engineering:testing-strategy`, `10x-Team:qa-engineer`
- Blocked by: 5.3

#### Step 6.2: Test harness design (Size: S)
- Prompt:
  ```
  [SHARED CONTEXT]
  Design the test harness: payment-provider simulator (webhook delays, duplicates, failures), fake
  facility (scripted voice/WhatsApp responses incl. Darija recordings), controllable clock for
  timeouts/cutoffs, concurrency tests for double booking. Say which tests run in CI and which in
  staging.
  ```
- Output: `22-test-harness.md`
- Skills: `zizi-skills:tdd-skill`
- Blocked by: 6.1

---

### Phase 7 — Build plan
Verified when: tickets exist for the first MVP slice with blockers linked.

#### Step 7.1: Risk-first spikes (Size: S each)
- Spikes before production code: (a) Darija voice benchmark on the chosen platform; (b) TN payment charge + refund end-to-end in sandbox; (c) double-booking concurrency test on the chosen data model; (d) workflow durability (crash mid-flow, resume).
- Each spike ends with go / no-go / change-design, recorded in `00-decisions.md`.
- Blocked by: 4.3 (b), 4.4 (a), 4.5 (c), 4.1 (d)

#### Step 7.2: Vertical slices and tickets (Size: S)
- Prompt:
  ```
  [SHARED CONTEXT]
  Split the MVP (scope from 00-decisions.md) into vertical slices, each one complete path end to end,
  highest risk first, in PRD §11 order. Each slice: goal, use-case IDs, acceptance tests from 21,
  components touched, size (XS–M; split anything larger), and "Blocked by". Then write one ticket per
  slice.
  ```
- Output: `23-build-plan.md` + tickets
- Skills: `zizi-skills:planning`, `zizi-skills:project-manager`
- Blocked by: 6.2, 7.1

### Phase 8 — Implementation (fog)
Each slice gets its own approved implementation plan at pickup (stack, repo, CI, environments are decided in 0.1/7.2). Not planned here.

---

## Decision log
| ID | Decision | Chosen option | Why |
| --- | --- | --- | --- |
| D1 | Is the feature being built? | Yes — decided | Founder decision; this plan is about how |
| D2 | Relation to existing Matchmaking Agent | Separate feature and design | Confirmed by Aziz, 18 Nov 2026 |
| D3 | Format for all artifacts | Markdown + Mermaid | Readable and editable by every LLM |
| D4 | Where artifacts live | New git repo (e.g. `reality-booking-agent`), all design docs under `docs/design/` | Versioned; every LLM (Codex, Claude, others) reads the same files. Confirmed by Aziz, 18 Nov 2026 |
| D6 | LLM roles | Codex/GPT produces the documents and code; Claude reviews and challenges each output | Same split as the existing Matchmaking Agent repo. Confirmed by Aziz, 18 Nov 2026 |
| D5 | Order of work | Understand → ground truth → model → design → risks → tests → build plan | Design built on verified facts, risk-first spikes before code |

## Not yet specified
- Phase 0: all seven founder questions in step 0.1.
- Phase 2: which Reality API docs exist for facilities, availability, bookings, payments, notifications.
- Phase 4: stack-dependent choices (workflow engine product, language) wait on 0.1.
- Phase 8: fog.

## Always / Never
- Always: cite PRD sections; record assumptions and open questions; Mermaid diagrams; Tunisia first with US designed in; verify vendor facts from official docs.
- Never: invent Reality APIs or vendor capabilities; let the LLM own booking or payment finality; give card data to the LLM; modify the existing Matchmaking Agent repository for this feature; write production code before Phase 7 spikes.

## Progress
| Phase | Status |
| --- | --- |
| 0 Set-up & founder answers | pending |
| 1 Understand | pending |
| 2 Ground truth | pending |
| 3 Domain model | pending |
| 4 Architecture | pending |
| 5 Risks | pending |
| 6 Tests | pending |
| 7 Build plan | pending |
| 8 Implementation | fog |
