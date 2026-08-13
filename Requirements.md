# IGamingSupportAgent — Requirements

**Draft v0.1 · 7 August 2026**

---

## 1. Product definition

A **multi-tenant SaaS AI support agent for iGaming operators**. Operators connect their helpdesk and back office; the agent answers player conversations using live account state, escalating to humans under defined rules.

### 1.1 Decisions taken

| Decision | Choice | Consequence |
|---|---|---|
| Tenancy | **Multi-tenant SaaS** | Per-tenant config and credentials, tenant isolation, self-serve onboarding. Substantially larger build than single-tenant. |
| v1 agent authority | **Read + answer only** | Agent reads player state and answers. No writes to operator systems. |
| Channels / helpdesks | **Deferred** | Requirements below are written channel-agnostic. A channel abstraction is assumed; its shape is the next discussion. |

### 1.2 The positioning problem with read-only v1

Cevro's entire differentiation is *"Chatbots answer questions. AI agents take action."* A read-only v1 sits on the wrong side of that line by their framing. This is acceptable **only if** v1 competes on the axes it can win — depth of account-specific context, answer accuracy, compliance posture, escalation quality — and the action layer is architected in v1 and enabled in v2, not bolted on later.

Concretely: build the procedure engine (§4) in v1 with a step vocabulary that includes write actions, but ship with all write step types disabled by tenant policy. The engine, audit trail, precondition checks, and guardrails are identical; only the executor is gated.

**v1 cannot claim 80–90% automation.** A read-only agent's ceiling is the share of contacts that are genuinely informational. Target metrics in §11 are set accordingly.

---

## 2. Competitive baseline

What the competitor claims, and what it implies we must match or consciously decline.

| Cevro claim | Source | Our position |
|---|---|---|
| 80–90% automation; 91% in case study | p3, p5 | Not matchable in v1 (read-only). Required for v2. |
| 4.8+ CSAT | p3, p5 | Match in v1. Achievable without writes. |
| AI Procedures — declarative, human-readable workflows | p4 | **Match, and beat.** This is the architectural centre of gravity. |
| Preconditions block unsafe starts; forbidden actions | p4 | Match in v1. |
| RG monitoring on 100% of interactions | p4, p6 | **Match in v1. Non-negotiable.** |
| SOC2 Type II, PII masking, zero retention, no training on customer data | p4 | Match posture in v1; certification is a timeline item. |
| 120+ languages | p6 | Match. Cevro's case study spans UK, DE, FR, NL, AU, IT, ES. |
| Full audit logging of every action | p4 | Match in v1. |
| Back office + helpdesk + CRM + chat + Slack integrations, API-first | p4, p5 | Deferred to channel discussion. |
| Proactive CRM: churn prevention, milestone bonuses, RG flagging | p6 | Out of scope v1. Architect for it. |
| Support insights feeding product/UX teams | p6 | Falls out of §10 analytics. Cheap to offer. |

Cevro's guide is marketing, not a technical disclosure. Every number above is unaudited and self-reported. Treat as a statement of **buyer expectation**, not of achieved engineering.

---

## 3. Users and stakeholders

| Actor | Need |
|---|---|
| **Player** | Fast, accurate, empathetic answers about their own account. Immediate path to a human. Clear knowledge they are talking to software. |
| **CS agent** | Not to be talked over. Full context when a conversation escalates. Trust that the agent stayed inside its limits. |
| **CS / CX lead** (economic buyer) | Automation rate, CSAT, deflection, cost per contact. Control over what the agent is allowed to say. |
| **Compliance / RG officer** | Evidence of duty of care. 100% RG coverage. Complete audit trail. Jurisdictional data controls. |
| **Operator IT** | Low-effort integration. Credentials scoped and revocable. No unbounded load on their back office. |
| **Our ops team** | Per-tenant observability, kill switches, incident response across tenants. |

---

## 4. Functional requirements — the procedure engine

The core asset. Cevro calls theirs *AI Procedures (AIPs)*: declarative, human-readable workflow definitions the engine executes step by step, rather than scripted decision trees or fine-tuned models (p4).

**FR-1** — Procedures MUST be declarative documents, authored and reviewable by non-engineers (CS leads, compliance), versioned, and diffable.

**FR-2** — Each procedure MUST declare:
- **Trigger** — intents/conditions that select it
- **Preconditions** — must hold before any step runs; failure routes to a defined fallback, never a guess
- **Steps** — ordered, each with a typed action and expected outcomes
- **Forbidden actions** — explicitly disallowed within this procedure
- **Escalation rules** — conditions and target
- **Disclosure requirements** — what must be said to the player
- **Data classification** — what player data the procedure may touch

**FR-3** — Step vocabulary MUST include write action types from v1 (`issue_bonus`, `trigger_refund`, `resend_verification`, `update_ticket`, …) even though v1 disables their execution. A procedure referencing a disabled step type MUST fail validation at author time with a clear message, not at runtime in front of a player.

**FR-4** — Step types required and **enabled** in v1:
- `verify_identity` — confirm the player is who the session claims
- `read_player_state` — balance, account status, tier, limits, flags
- `read_transactions` — deposits, withdrawals, status, timestamps
- `read_kyc_status` — document state and outstanding requirements
- `read_bonus_state` — active bonuses, wagering progress, eligibility
- `read_knowledge` — retrieval over operator content (T&Cs, help centre)
- `classify` — intent, sentiment, RG risk, confidence
- `respond` — generate a player-facing message
- `escalate` — hand to a human with structured context
- `log` — write to our audit trail (not the operator's systems)

**FR-5** — Every step execution MUST be recorded: inputs, outputs, latency, decision taken, and which procedure version was in force.

**FR-6** — The engine MUST support a dry-run mode: execute a procedure against a real conversation and record what it *would* have said, without sending. Required for tenant onboarding and for validating procedure changes before release.

**FR-7** — Procedures MUST be per-tenant, with a shared library of iGaming baseline procedures a tenant can adopt and override.

### 4.1 Baseline procedure library — v1 scope

Derived from the contact types the competitor names as high-volume (p3, p4, p5). All read-only in v1.

| Procedure | Player question | v1 behaviour |
|---|---|---|
| **Where is my deposit** | "My deposit hasn't arrived" | Verify identity → read transactions → cross-reference balance → explain status and expected timing → escalate to payments with context if failed or missing |
| **Withdrawal status** | "Where is my withdrawal?" | Read withdrawal state, pending checks, expected timing → explain → escalate if stalled beyond threshold |
| **KYC / verification** | "Why is my account not verified?" | Read KYC state → explain exactly which documents are outstanding and why → escalate for document review |
| **Bonus status and eligibility** | "Where is my bonus / why can't I withdraw?" | Read bonus state and wagering progress → explain terms against *their* actual progress → escalate to issue |
| **Account access / lock** | "I can't log in" | Read account status → explain cause class → escalate to the correct team |
| **Responsible gambling** | Any RG signal | **Never generated free text.** Fixed approved copy + immediate escalation. See §6. |
| **Complaint / dispute** | Regulatory or financial dispute | Escalate with zero generated text to the player. See §6. |
| **General information** | Rules, limits, payment methods | Knowledge retrieval, no account read required |

**FR-8** — The "where is my deposit" procedure is the acceptance benchmark for v1, because it is the competitor's own worked example (p4) and exercises identity, payment data, back-office cross-reference, and escalation in one flow.

---

## 5. Functional requirements — conversation

**FR-9** — Resolve the player to a stable operator-side identifier on every conversation. No fuzzy email matching as a primary key.

**FR-10** — Classify every inbound message for: intent, RG risk, sentiment, and a confidence score. Classification runs before any response generation.

**FR-11** — Confidence below a per-tenant threshold MUST escalate rather than answer.

**FR-12** — Support 120+ languages, matching the competitor claim. Detect language per conversation and respond in kind. Approved fixed copy (RG, disclosure, escalation) MUST be human-translated per supported market, never machine-translated at runtime.

**FR-13** — Handover to a human MUST be sticky. Once a human replies, the agent does not resume unless a human explicitly hands back. *(An agent that reappears mid-human-conversation is the most damaging behavioural failure available to this design.)*

**FR-14** — Escalation MUST carry structured context to the human: player identity, what the agent read, what it concluded, why it escalated, and the full transcript.

**FR-15** — A player asking for a human MUST be escalated immediately, with no deflection attempt.

**FR-16** — Cap auto-replies per conversation; on hitting the ceiling, escalate rather than continue.

**FR-17** — Debounce fragmented player messages (players commonly send three short messages, not one long one) and answer once, when the thought is complete.

**FR-18** — Per-conversation state MUST be tracked: status, last processed message, reply count, intent, confidence, escalation reason.

---

## 6. Compliance and responsible gambling

The requirements most likely to be argued with and least likely to be regretted. iGaming is licensed; this section is not negotiable engineering preference.

**FR-19** — **RG monitoring on 100% of interactions**, matching the competitor's claim (p4). Every inbound message is screened for distress and problem-gambling signals regardless of intent.

**FR-20** — On RG signal: the agent MUST NOT generate free text. It sends operator-approved, jurisdiction-appropriate fixed copy and escalates immediately to a trained human.

**FR-21** — The **always-escalate intent set** MUST be per-tenant configurable and MUST default to: responsible gambling and self-exclusion, account closure, complaints and disputes, AML/source-of-funds, chargebacks, legal threats, safeguarding and wellbeing, and anything with financial-remedy implications.

**FR-22** — For always-escalate intents, **zero generated text reaches the player**. Fixed acknowledgement copy only.

**FR-23** — The agent MUST disclose it is automated in its first message of every conversation, and MUST be named unambiguously as software (not a human-sounding first name).

**FR-24** — Self-excluded and cooling-off players MUST be detected on identity resolution and handled under a distinct policy, never a standard support flow.

**FR-25** — Every interaction MUST be auditable end to end: what was read, what was decided, what was said, under which procedure version and which policy configuration. Retention period per tenant and per jurisdiction.

**FR-26** — Per-tenant jurisdiction configuration MUST gate behaviour: markets have materially different rules on bonus communication, RG intervention, and disclosure. A procedure MUST be able to vary by player jurisdiction.

**FR-27** — Player-facing output MUST be sanitised before sending. It is generated content entering a rendered surface.

**FR-28** — Regulatory constraint on generated content: the agent MUST NOT generate promotional or inducement-shaped language. In several markets this is a licensing matter, not a tone preference.

---

## 7. Multi-tenancy

**FR-29** — Tenant isolation MUST be enforced at the data layer. No query path may return one operator's player data to another. This is the single highest-severity failure available to us.

**FR-30** — Per-tenant credential storage for operator systems, encrypted at rest, individually revocable, never logged.

**FR-31** — Self-serve onboarding: connect helpdesk, connect back office, map identity, adopt baseline procedures, dry-run, go live.

**FR-32** — Per-tenant configuration surface covering, at minimum: confidence thresholds, always-escalate set, auto-reply ceiling, debounce window, latency ceiling, disclosure copy, RG copy, supported languages, jurisdiction map, business hours and escalation targets, and the kill switch.

**FR-33** — Per-tenant rate limiting against operator back-office APIs. We must not be the reason an operator's back office falls over.

**FR-34** — Tenant-scoped and global **kill switches** that immediately stop all player-facing output while leaving ingestion, classification, and logging running. Effective in seconds, not a redeploy.

**FR-35** — Noisy-neighbour isolation: one tenant's volume spike must not degrade another's latency.

---

## 8. Integration requirements (channel-agnostic)

Detailed channel and helpdesk selection is deferred. These hold regardless.

**FR-36** — Inbound events MUST be acknowledged fast and processed asynchronously. Model inference will exceed typical webhook acknowledgement windows.

**FR-37** — Do not trust event delivery. Reconcile periodically against the source of truth and repair anything missed.

**FR-38** — Treat events as wake-up signals; re-fetch conversation state rather than trusting the delivered payload. Ordering is not guaranteed on any platform we will integrate.

**FR-39** — All outbound player-facing messages MUST be idempotent, keyed to the triggering inbound message. A duplicate message to a player is worse than a slow one.

**FR-40** — Back-office reads MUST tolerate operator API failure: degrade to escalation with a truthful explanation, never to a guess or a fabricated status.

**FR-41** — Assume sent messages cannot be edited or deleted on most channels. A wrong answer about a withdrawal or bonus is permanent and visible.

---

## 9. Non-functional requirements

| ID | Requirement | Target |
|---|---|---|
| **NFR-1** | Time to first player-visible response | < 3 s p95 |
| **NFR-2** | Time to substantive answer | < 8 s p95 |
| **NFR-3** | Hard latency ceiling, then escalate | 15 s |
| **NFR-4** | Dead-air mitigation | Immediate acknowledgement message where the channel offers no typing indicator |
| **NFR-5** | Sustained throughput per tenant | 100k conversations/month (competitor case study: 80–100k) |
| **NFR-6** | Peak burst | 10× mean, without latency breach |
| **NFR-7** | Availability | 99.9% ingestion; degradation MUST fail to human escalation, never to silence |
| **NFR-8** | Procedure change to production | < 1 hour, with dry-run gate |
| **NFR-9** | Kill switch propagation | < 10 s |
| **NFR-10** | Onboarding to first live conversation | Days, not weeks (competitor claims impact "in weeks") |

---

## 10. Observability and reporting

**FR-42** — Per-tenant dashboard: volume, automation rate, escalation rate and reasons, CSAT, first-response time, resolution time, confidence distribution, top intents.

**FR-43** — Machine-readable outcome recorded on every conversation in the operator's own helpdesk (intent, confidence, escalation reason) so operators can analyse in their existing tooling.

**FR-44** — ROI reporting: deflected contacts, agent-hours saved, cost per contact. The competitor leads with an ROI calculator (p7); the economic buyer will ask.

**FR-45** — Quality review queue: sample conversations for human review, with correction feedback improving procedures.

**FR-46** — Alerting on: confidence drift, escalation rate spikes, RG detection anomalies, operator API failures, delivery failures.

**FR-47** — Support-insight export: recurring intents and friction points, for the operator's product teams. The competitor markets this as an upside benefit (p6) and it is nearly free once §10 exists.

---

## 11. Success criteria — v1

Set against read-only authority. These are not Cevro's numbers, and should not be presented as if they were.

| Metric | v1 target | Note |
|---|---|---|
| Full automation (no human touch) | 35–50% | Ceiling is the informational share of contacts |
| Assisted resolution (agent gave context, human closed) | +25% | Real value even without writes |
| CSAT on agent-handled conversations | ≥ 4.5 | Toward the competitor's 4.8 claim |
| First response time | < 5 s | Against human baseline of minutes |
| Wrong-answer rate on account-specific questions | < 0.5% | The metric that ends the product if it slips |
| RG signal recall | > 99% | Missed RG signals are a licence risk |
| Always-escalate leakage (generated text sent on a forbidden intent) | **0** | Any occurrence is a Sev-1 |
| Agent-talks-over-human incidents | **0** | |
| Cross-tenant data exposure | **0** | |

---

## 12. Out of scope for v1

- **Writes to operator systems.** Architected in v1 (FR-3), enabled in v2. This is the automation-rate unlock.
- **Proactive outbound / CRM.** Churn prevention, milestone bonuses, re-engagement (competitor p6). Requires marketing-permission and jurisdictional-inducement handling — a materially different compliance problem from inbound.
- **Voice.**
- **Agent-assist mode.** Deliberately deferred despite being low-risk; it is a different product surface and would split v1 focus.
- **SOC2 Type II certification.** Build to the posture in v1; certification is a parallel timeline item. The competitor leads with it (p4), so enterprise deals will require it.

---

## 13. Open questions

1. **Channels and helpdesks** — next discussion. Determines the integration abstraction and whether buttons/rich affordances are available.
2. **Back-office diversity** — how many distinct platforms must we read from, and do they expose the payment, KYC, and bonus state the baseline procedures need? *This is the biggest unknown in the plan.* The baseline library is worthless if operator systems can't answer FR-4's read steps.
3. **Model hosting and data residency** — whether player conversation content may leave our infrastructure, and to which jurisdiction. Must be answered before any spike. Cevro claims zero retention and no training on customer data; we will be held to that bar.
4. **Identity assurance** — what constitutes sufficient `verify_identity` for disclosing account-specific data over chat, per jurisdiction. Under-specifying this is a data-protection incident.
5. **Design partner** — which operator, which markets, which volume. Cevro's case study spans 7 markets; a single-market first tenant is a materially easier v1.
6. **Pricing model** — Cevro bills on outcomes. Per-resolution, per-seat, or per-volume changes what we must meter from day one.

---

## Appendix: source

`Competitor.pdf` — Cevro AI, *"AI Agents for Customer Support in iGaming: A Guide for Evaluating Agentic CX Automation."* 7 pages. Sales collateral; all figures self-reported and unaudited. Page references above are to that document.
