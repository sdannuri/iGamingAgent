# IGamingSupportAgent — Requirements

**Draft v0.2 · 13 August 2026**

A multi-tenant SaaS AI support agent for iGaming operators. Operators connect their helpdesk and back office; the agent answers player conversations using live account state, and escalates to humans under defined rules.

**How to read this document.** Part 1 is context. Parts 2 and 3 are the **complete requirements catalogue** — everything the product must eventually do, independent of when we build it. Part 4 phases that catalogue into V1, V2 and V3. Parts 5–7 cover targets, open questions, and competitive context.

**Convention.** `MUST` = mandatory for the phase it is assigned to. `SHOULD` = strong default, may be traded. Every requirement is written to be testable. Requirement IDs are stable — phasing may change, IDs do not.

---

# Part 1 — Context

## 1.1 Decisions taken

| Decision | Choice | Consequence |
|---|---|---|
| Tenancy | **Multi-tenant SaaS** | Per-tenant config and credentials, tenant isolation, self-serve onboarding. Substantially larger build than single-tenant. |
| V1 agent authority | **Read + answer only** | Agent reads player state and answers. No writes to operator systems until V2. |
| Channels / helpdesks | **Deferred** | Requirements are written channel-agnostic behind an adapter layer. Selection is a separate discussion. |

## 1.2 Architectural commitments

Three decisions that shape the whole catalogue and are assumed throughout:

**The procedure decides what happens; the model decides what it says.** Support scenarios are declarative procedures executed step by step. The model classifies and phrases; it does not choose to touch player data. This is what makes preconditions provable and the audit trail meaningful — an architectural property, not a policy promise.

**Safety gates sit outside the procedure engine.** RG screening, emotional-state assessment and always-escalate detection run as a pre-gate on every inbound message, before procedure selection, with authority to pre-empt. If they were procedure *steps*, any unauthored path would silently drop coverage. A post-gate validates every outbound message before it reaches a player.

**Connectors are capability contracts, not integrations.** Each read step is backed by a declared capability. A tenant's connector implements what that operator can actually supply; procedures declare what they require; unmet capabilities disable the procedure for that tenant rather than failing at runtime.

## 1.3 What read-only means for V1

The benchmark competitor's positioning is *"chatbots answer questions, AI agents take action."* A read-only V1 sits on the wrong side of that line by their framing, so V1 must compete on the axes it can win: depth of account-specific context, answer accuracy, compliance posture, and escalation quality.

**V1 cannot claim 80–90% automation.** A read-only agent's ceiling is the informational share of contacts. Phase targets in Part 5 are set accordingly. The action layer (§2.10) is specified now and enabled in V2 — designed in, not retrofitted.

---

# Part 2 — Functional requirements

*Complete catalogue. Phase assignment is in Part 4.*

## 2.1 Procedure engine

| ID | Requirement | |
|---|---|---|
| **FR-1** | Procedures are declarative documents, authored and reviewable by non-engineers, versioned and diffable. | MUST |
| **FR-2** | Each procedure declares: trigger conditions, preconditions, ordered steps, forbidden actions, escalation rules, disclosure requirements, data classification, and required connector capabilities. | MUST |
| **FR-3** | Failed preconditions route to a defined fallback. The agent never proceeds on a guess. | MUST |
| **FR-4** | The full step vocabulary — including write step types — exists from the first release. Execution of each type is gated by tenant policy and product phase. | MUST |
| **FR-5** | A procedure referencing a disabled or unavailable step type fails validation at author time with a clear message, never at runtime in front of a player. | MUST |
| **FR-6** | Every step execution records inputs, outputs, latency, decision taken, and the procedure version in force. | MUST |
| **FR-7** | Dry-run mode executes a procedure against a real conversation and records what it *would* have said, without sending. | MUST |
| **FR-8** | Procedures are per-tenant, drawn from a shared baseline library that tenants may adopt and override. | MUST |
| **FR-9** | Procedure runs are short-lived and scoped to a turn-cluster, not to a whole conversation. A conversation holds a stack of runs, so a player raising two issues at once is handled correctly. | MUST |
| **FR-10** | Procedures declare required capabilities. Where a tenant's connectors cannot supply one, the procedure is disabled for that tenant and matching contacts route to humans. | MUST |
| **FR-11** | Tenants can author new procedures without engineering involvement. | MUST |

### Step vocabulary

**FR-12** — The following step types MUST exist in the vocabulary.

| Category | Steps |
|---|---|
| **Identity** | `verify_identity` |
| **Read** | `read_player_state`, `read_transactions`, `read_kyc_status`, `read_bonus_state`, `read_limits`, `read_knowledge` |
| **Control** | `classify`, `branch`, `wait`, `respond`, `escalate`, `log` |
| **Write** | `issue_bonus`, `trigger_refund`, `resend_verification`, `update_ticket`, `apply_limit`, `reset_credential`, `close_account` |

## 2.2 Baseline procedure library

**FR-13** — The baseline library MUST ship with the following procedures.

| Procedure | Player question | Behaviour |
|---|---|---|
| **Where is my deposit** | "My deposit hasn't arrived" | Verify identity → read transactions → cross-reference balance → explain status and timing → escalate to payments with context if failed or missing |
| **Withdrawal status** | "Where is my withdrawal?" | Read withdrawal state, pending checks, expected timing → explain → escalate if stalled beyond threshold |
| **KYC / verification** | "Why is my account not verified?" | Read KYC state → explain exactly which documents are outstanding and why → escalate for document review |
| **Bonus status and eligibility** | "Where is my bonus / why can't I withdraw?" | Read bonus state and wagering progress → explain terms against *their* actual progress → escalate to issue |
| **Account access / lock** | "I can't log in" | Read account status → explain cause class → escalate to the correct team |
| **Limits and self-service controls** | "How do I set a deposit limit?" | Read current limits → explain → route to the operator's own control surface |
| **Responsible gambling** | Any RG signal | Fixed approved copy + immediate escalation. Never generated text. See §2.9 |
| **Complaint / dispute** | Regulatory or financial dispute | Escalate with zero generated text to the player. See §2.9 |
| **General information** | Rules, payment methods, game rules | Knowledge retrieval, no account read required |

**FR-14** — *Where is my deposit* is the **acceptance benchmark**. It exercises identity, payment data, back-office cross-reference, and escalation in a single flow.

## 2.3 Identity and player context

| ID | Requirement | |
|---|---|---|
| **FR-15** | Resolve the player to a stable operator-side identifier on every conversation. Fuzzy email matching MUST NOT be the primary key. | MUST |
| **FR-16** | Complete `verify_identity` before disclosing any account-specific data, to a standard configurable per jurisdiction. | MUST |
| **FR-17** | Detect self-excluded and cooling-off players at identity resolution and route them to a distinct policy, never a standard support flow. | MUST |
| **FR-18** | On operator back-office failure, degrade to escalation with a truthful explanation. Never guess, never fabricate a status. | MUST |
| **FR-19** | Handle unidentified players (pre-login, pre-registration) under a restricted policy with no account-specific disclosure. | MUST |

## 2.4 Message comprehension

| ID | Requirement | |
|---|---|---|
| **FR-20** | Classify every inbound message before the agent acts: intent, emotional state, RG risk, abuse, language, and confidence. | MUST |
| **FR-21** | Classification dimensions run concurrently rather than in sequence, to protect the latency budget. | MUST |
| **FR-22** | Escalate rather than answer when confidence falls below the per-tenant threshold. | MUST |
| **FR-23** | Detect multiple intents in a single message and resolve them as separate procedure runs (FR-9). | MUST |
| **FR-24** | Support 120+ languages. Detect language per conversation and respond in kind. | MUST |
| **FR-25** | Approved fixed copy (RG, disclosure, escalation) is human-translated per supported market. Never machine-translated at runtime. | MUST |

## 2.5 Emotional state and routing

Emotion is assessed on every message before the agent acts, and modulates behaviour across five bands rather than acting as a binary switch. In iGaming, mild frustration is the baseline rather than the exception — a binary rule escalates most of the inbound volume and destroys the automation rate, and an accurate instant answer is often the fastest de-escalation available.

**FR-26** — The agent MUST assess emotional state on every inbound message, before acting.

**FR-27** — Emotional state MUST drive the following graduated response.

| Band | Behaviour |
|---|---|
| **Distress / RG** | Separate lane entirely. Fixed approved copy, trained human. **Never the general queue.** |
| **Abuse or threat** | Immediate escalation, flagged, under its own handling policy |
| **High frustration** | Proceed only if confidence is high and the issue is answerable this turn; otherwise escalate |
| **Mild frustration** | Proceed, with tone shifted — lead with the answer, drop pleasantries, acknowledge briefly |
| **Neutral** | Normal path |

| ID | Requirement | |
|---|---|---|
| **FR-28** | RG and distress detection outranks and pre-empts frustration routing. A distressed player MUST NOT be routed to the general support queue. | MUST |
| **FR-29** | Abuse and threats are handled under a distinct policy, not the standard escalation path. | MUST |
| **FR-30** | Track emotional trajectory across turns. A rising slope is an escalation trigger independent of absolute level. | MUST |
| **FR-31** | Factor repeat-contact history into routing. A calm third contact about the same issue outranks a single angry first contact. | MUST |
| **FR-32** | Escalation decisions are availability-aware. Where no human is on shift, attempt the answer with honest disclosure of the wait rather than escalating into a void. | MUST |
| **FR-33** | The emotional band feeds the composer, not only the router. Detecting frustration and replying in default voice is worse than not detecting it. | MUST |
| **FR-34** | Emotion thresholds are calibrated per tenant **and per market**. Sentiment models degrade badly outside English, and ordinary directness in some markets reads as anger. | MUST |
| **FR-35** | Monitor emotion classifier accuracy per language and alert on drift or over-escalation in any market. | MUST |

## 2.6 Escalation and human handover

| ID | Requirement | |
|---|---|---|
| **FR-36** | Handover is **sticky**. Once a human replies, the agent does not resume unless a human explicitly hands back. | MUST |
| **FR-37** | Escalation carries structured context: player identity, what the agent read, what it concluded, why it escalated, and the full transcript. | MUST |
| **FR-38** | A player asking for a human is escalated immediately, with no deflection attempt. | MUST |
| **FR-39** | Escalation targets are configurable per tenant, per intent, and per business hours. | MUST |
| **FR-40** | Cap auto-replies per conversation; escalate on reaching the ceiling rather than continuing. | MUST |
| **FR-41** | Track per-conversation state: status, last processed message, reply count, intent, confidence, emotional trajectory, escalation reason. | MUST |

> **FR-36 is the highest-consequence behavioural requirement in this document.** An agent that reappears mid-human-conversation is the most damaging failure mode available to this design.

## 2.7 Conversation mechanics

| ID | Requirement | |
|---|---|---|
| **FR-42** | Debounce fragmented player messages and answer once, when the thought is complete. | MUST |
| **FR-43** | Disclose that the agent is automated in its first message of every conversation, under an unambiguously non-human name. | MUST |
| **FR-44** | Sanitise all player-facing output before sending. | MUST |
| **FR-45** | A safety post-gate validates every outbound message before send: locked copy verifiably came from locked copy, no inducement-shaped language, no other player's data present. | MUST |
| **FR-46** | Treat all player-supplied and retrieved text as untrusted input. It MUST NOT be able to alter agent instructions or procedure control flow. | MUST |

## 2.8 Responsible gambling and compliance behaviour

iGaming is licensed. This section is not negotiable engineering preference.

| ID | Requirement | |
|---|---|---|
| **FR-47** | Screen **100% of inbound messages** for distress and problem-gambling signals, regardless of intent or procedure. | MUST |
| **FR-48** | RG screening is implemented in the pipeline pre-gate, not as a procedure step, so coverage cannot be lost by an unauthored path. | MUST |
| **FR-49** | On any RG signal, send operator-approved jurisdiction-appropriate fixed copy and escalate immediately to a trained human. No generated text. | MUST |
| **FR-50** | Maintain a per-tenant **always-escalate intent set**, defaulting to: responsible gambling and self-exclusion, account closure, complaints and disputes, AML and source-of-funds, chargebacks, legal threats, safeguarding and wellbeing, and anything with financial-remedy implications. | MUST |
| **FR-51** | For always-escalate intents, **zero generated text reaches the player**. Fixed acknowledgement copy only. | MUST |
| **FR-52** | Never generate promotional or inducement-shaped language. In several markets this is a licence condition, not a tone preference. | MUST |
| **FR-53** | Procedure behaviour varies by player jurisdiction. Markets differ materially on bonus communication, RG intervention, and disclosure. | MUST |

## 2.9 Knowledge and content

| ID | Requirement | |
|---|---|---|
| **FR-54** | Maintain a per-tenant knowledge index over operator content: T&Cs, bonus terms, help centre, payment methods, game rules. | MUST |
| **FR-55** | Ground answers in indexed source content. The agent MUST NOT assert terms or policy it cannot ground. | MUST |
| **FR-56** | Support content re-indexing on change, with staleness visible to the tenant. Wrong bonus terms are a complaint generator. | MUST |
| **FR-57** | Maintain a per-market approved copy library for all fixed-copy responses. | MUST |

## 2.10 Action execution — writes

Specified now, enabled in V2. The engine, audit trail, preconditions and guardrails are identical to read steps; only the executor is gated.

| ID | Requirement | |
|---|---|---|
| **FR-58** | Execute write steps against operator systems within procedure-declared guardrails. | MUST |
| **FR-59** | Every write is preconditioned and idempotent. A retry MUST NOT double-issue a bonus or double-refund. | MUST |
| **FR-60** | Write authority is configurable per tenant and per procedure, with monetary ceilings and rate caps. | MUST |
| **FR-61** | Writes above a configured threshold require dual control or human approval before execution. | MUST |
| **FR-62** | Every write is reversible, or gated behind human approval where it is not. | MUST |
| **FR-63** | Write actions are recorded in a distinct, immutable action log separate from conversation audit. | MUST |
| **FR-64** | A tenant-scoped switch disables all writes independently of the conversational kill switch. | MUST |

## 2.11 Agent-assist mode

| ID | Requirement | |
|---|---|---|
| **FR-65** | Produce AI suggestions visible to human agents only, never to players — same procedures, same context, no player-facing risk. | MUST |
| **FR-66** | Measure suggestion accept rate and edit distance as a quality signal. | SHOULD |

## 2.12 Proactive outbound

| ID | Requirement | |
|---|---|---|
| **FR-67** | Trigger proactive outreach on behavioural signals: churn risk, milestones, stalled KYC, failed payments. | MUST |
| **FR-68** | Check marketing permission and jurisdictional inducement rules before any outbound message. This is a materially different compliance problem from inbound. | MUST |
| **FR-69** | Apply frequency capping per player across all proactive campaigns. | MUST |
| **FR-70** | Proactive RG outreach is a distinct path from commercial outreach, with separate approval and copy. | MUST |
| **FR-71** | Suppress all commercial outreach to self-excluded, cooling-off, and RG-flagged players. | MUST |

## 2.13 Multi-tenant configuration and onboarding

| ID | Requirement | |
|---|---|---|
| **FR-72** | Self-serve onboarding: connect helpdesk → connect back office → map identity → negotiate capabilities → adopt baseline procedures → dry-run → go live. | MUST |
| **FR-73** | Capability negotiation at onboarding determines which procedures are available to that tenant (FR-10). | MUST |
| **FR-74** | Per-tenant configuration covers at minimum: confidence thresholds, emotion band thresholds, always-escalate set, auto-reply ceiling, debounce window, latency ceiling, disclosure copy, RG copy, supported languages, jurisdiction map, business hours, escalation targets, write authority, kill switches. | MUST |
| **FR-75** | Store operator credentials per tenant, individually revocable. | MUST |
| **FR-76** | Provide tenant-scoped and global **kill switches** that immediately stop all player-facing output while leaving ingestion, classification, and logging running. | MUST |
| **FR-77** | Gate every procedure release behind a dry-run. | MUST |

## 2.14 Integration and channel behaviour

Channel selection is deferred; these hold on any channel.

| ID | Requirement | |
|---|---|---|
| **FR-78** | A channel adapter layer normalises inbound events to a canonical conversation event and renders outbound messages per channel. | MUST |
| **FR-79** | Acknowledge inbound events fast and process asynchronously. Inference will exceed typical webhook acknowledgement windows. | MUST |
| **FR-80** | Do not trust event delivery. Reconcile periodically against the source of truth and repair anything missed. | MUST |
| **FR-81** | Treat events as wake-up signals and re-fetch conversation state. Message ordering is not guaranteed. | MUST |
| **FR-82** | Make all outbound player-facing messages idempotent, keyed to the triggering inbound message. A duplicate message is worse than a slow one. | MUST |
| **FR-83** | Assume sent messages cannot be edited or deleted. A wrong answer about a withdrawal or bonus is permanent and visible. | MUST |
| **FR-84** | Apply per-tenant rate limiting against operator back-office APIs. We must not be why an operator's back office falls over. | MUST |

## 2.15 Voice

| ID | Requirement | |
|---|---|---|
| **FR-85** | Support voice as a channel behind the same adapter layer, procedures, and safety gates. | MUST |

## 2.16 Reporting and analytics

| ID | Requirement | |
|---|---|---|
| **FR-86** | Per-tenant dashboard: volume, automation rate, escalation rate and reasons, CSAT, first-response time, resolution time, confidence distribution, top intents. | MUST |
| **FR-87** | Write machine-readable outcomes (intent, confidence, escalation reason) back to the operator's own helpdesk so they can analyse in existing tooling. | MUST |
| **FR-88** | ROI reporting: deflected contacts, agent-hours saved, cost per contact. The economic buyer will ask. | MUST |
| **FR-89** | Quality review queue: sample conversations for human review, with corrections feeding back into procedures. | MUST |
| **FR-90** | Alert on confidence drift, escalation-rate spikes, RG detection anomalies, emotion classifier drift, operator API failures, and delivery failures. | MUST |
| **FR-91** | Report emotion band distribution and escalation outcomes per market, to tune FR-34 against real data. | MUST |
| **FR-92** | Support-insight export: recurring intents and friction points, for the operator's product teams. | SHOULD |

---

# Part 3 — Non-functional requirements

## 3.1 Performance and latency

| ID | Requirement | Target |
|---|---|---|
| **NFR-1** | Time to first player-visible response | < 3 s p95 |
| **NFR-2** | Time to substantive answer | < 8 s p95 |
| **NFR-3** | Hard latency ceiling, then escalate | 15 s |
| **NFR-4** | Safety pre-gate resolution (RG, emotion, always-escalate) | < 400 ms |
| **NFR-5** | Dead-air mitigation: immediate acknowledgement where the channel offers no typing indicator | Within NFR-1 |
| **NFR-6** | Kill-switch propagation | < 10 s |

## 3.2 Scale and capacity

| ID | Requirement | Target |
|---|---|---|
| **NFR-7** | Sustained throughput per tenant | 100k conversations/month |
| **NFR-8** | Peak burst without latency breach | 10× mean |
| **NFR-9** | Noisy-neighbour isolation: one tenant's spike must not degrade another's latency | No cross-tenant impact |

## 3.3 Availability and resilience

| ID | Requirement | Target |
|---|---|---|
| **NFR-10** | Ingestion availability | 99.9% |
| **NFR-11** | Degradation fails to human escalation, never to silence | Always |
| **NFR-12** | No inbound player message lost, including during partial outage | Zero loss |

## 3.4 Security and tenant isolation

| ID | Requirement | |
|---|---|---|
| **NFR-13** | Tenant isolation enforced at the data layer. No query path may return one operator's player data to another. | MUST |
| **NFR-14** | Operator credentials encrypted at rest and never written to logs. | MUST |
| **NFR-15** | Operator credentials scoped to least privilege — read-only scopes until write phase. | MUST |
| **NFR-16** | PII masked before content reaches any model provider. | MUST |
| **NFR-17** | Secure SDLC with penetration testing before first production tenant. | MUST |

> **NFR-13 is the highest-severity requirement in this document.** Cross-tenant player data exposure is the failure that ends the product.

## 3.5 Data protection, residency and retention

| ID | Requirement | |
|---|---|---|
| **NFR-18** | Zero retention with the model provider. No training on operator or player data. | MUST |
| **NFR-19** | Data residency configurable per tenant and per jurisdiction. | MUST |
| **NFR-20** | Retention periods configurable per tenant and per jurisdiction. | MUST |
| **NFR-21** | Support data subject erasure requests across conversation and audit stores. | MUST |

## 3.6 Auditability and compliance posture

| ID | Requirement | |
|---|---|---|
| **NFR-22** | Every interaction reconstructable end to end: what was read, what was decided, what was said, under which procedure version and policy configuration. | MUST |
| **NFR-23** | Audit records immutable and tamper-evident. | MUST |
| **NFR-24** | Audit export in a form an operator can hand to a regulator. | MUST |
| **NFR-25** | SOC2 Type II certification. Enterprise deals will require it. | MUST |

## 3.7 Operability

| ID | Requirement | Target |
|---|---|---|
| **NFR-26** | Procedure change to production, dry-run gated | < 1 hour |
| **NFR-27** | Onboarding to first live conversation | Days, not weeks |
| **NFR-28** | Per-tenant observability for our own ops team, with cross-tenant incident response | Continuous |

## 3.8 Authoring usability

| ID | Requirement | Target |
|---|---|---|
| **NFR-29** | A CS lead or compliance officer can author and dry-run a new procedure without engineering help | < 1 day, unaided |

> NFR-29 is what makes the procedure engine an asset rather than a config file. If procedures need an engineer, we have built a slower version of hard-coded flows.

---

# Part 4 — Phasing

## 4.1 V1 — Read and answer

**Goal:** a compliant, accurate, account-aware agent that answers informational contacts and escalates everything else cleanly. Proves the procedure engine, the safety gates, and the connector model against a real tenant.

| Area | Requirements |
|---|---|
| Procedure engine | FR-1 – FR-12 |
| Baseline library | FR-13, FR-14 |
| Identity and context | FR-15 – FR-19 |
| Comprehension | FR-20 – FR-25 |
| Emotion and routing | FR-26 – FR-35 |
| Escalation and handover | FR-36 – FR-41 |
| Conversation mechanics | FR-42 – FR-46 |
| RG and compliance | FR-47 – FR-53 |
| Knowledge and content | FR-54 – FR-57 |
| Tenant config and onboarding | FR-72 – FR-77 |
| Integration | FR-78 – FR-84 |
| Reporting | FR-86 – FR-91 |
| Non-functional | NFR-1 – NFR-24, NFR-26 – NFR-29 |

**Deliberately included despite the cost:** the full step vocabulary (FR-4, FR-12) including write types, and least-privilege read-only credential scoping (NFR-15). Both exist so V2 is a policy change rather than a re-architecture.

**Deliberately excluded:** all write execution, proactive outbound, voice, agent-assist. SOC2 certification (NFR-25) runs on a parallel timeline and will not have completed.

## 4.2 V2 — Action and assist

**Goal:** cross from answering to acting. This is where the automation rate moves and where the product matches the category's positioning.

| Area | Requirements |
|---|---|
| Write execution | FR-58 – FR-64 |
| Agent-assist mode | FR-65, FR-66 |
| Insight export | FR-92 |
| Compliance posture | NFR-25 (SOC2 Type II certified) |
| Credential scope | NFR-15 extends to scoped write permissions |

**Entry criteria — V2 write execution must not begin until V1 has demonstrated:**

- Wrong-answer rate on account-specific questions sustained below 0.5%
- Zero always-escalate leakage incidents
- Zero agent-talks-over-human incidents
- Audit trail proven complete against a real compliance review

Writes are the point at which a wrong answer becomes a wrong *transaction*. The V1 accuracy bar is the gate.

## 4.3 V3 — Proactive and multi-channel

**Goal:** move from reactive support to retention. Support becomes a revenue surface rather than a cost centre.

| Area | Requirements |
|---|---|
| Proactive outbound | FR-67 – FR-71 |
| Voice | FR-85 |

Proactive outbound carries a compliance problem V1 and V2 do not have: unsolicited contact is governed by marketing permission and jurisdictional inducement rules, and the penalty for messaging a self-excluded player is categorically worse than answering one badly. FR-71 is the requirement to be most careful with.

## 4.4 Phase summary

| | V1 | V2 | V3 |
|---|---|---|---|
| **Agent can** | Read, answer, escalate | Read, answer, act, assist | All of V2, plus initiate |
| **New FRs** | 66 | 10 | 6 |
| **Automation ceiling** | 35–50% | 70–85% | 70–85% + deflection |
| **Positioning** | Accuracy and compliance | Category parity | Retention engine |
| **Dominant risk** | Wrong answer | Wrong transaction | Wrong recipient |

---

# Part 5 — Success criteria

## 5.1 V1

| Metric | Target | Note |
|---|---|---|
| Full automation (no human touch) | 35–50% | Ceiling is the informational share of contacts |
| Assisted resolution (agent gave context, human closed) | +25% | Real value even without writes |
| CSAT on agent-handled conversations | ≥ 4.5 | |
| First response time | < 5 s | Against a human baseline of minutes |
| Wrong-answer rate on account-specific questions | < 0.5% | The metric that ends the product if it slips |
| RG signal recall | > 99% | Missed RG signals are a licence risk |
| Over-escalation from emotion gate, any single market | < 15% above global mean | Detects FR-34 calibration failure |
| Always-escalate leakage | **0** | Any occurrence is a Sev-1 |
| Agent-talks-over-human incidents | **0** | |
| Cross-tenant data exposure | **0** | |

## 5.2 V2

| Metric | Target |
|---|---|
| Full automation | 70–85% |
| CSAT | ≥ 4.8 |
| Incorrect write actions | **0** |
| Write actions requiring reversal | < 0.1% |
| Agent-assist suggestion accept rate | > 60% |

## 5.3 V3

| Metric | Target |
|---|---|
| Proactive contact deflection (tickets prevented) | Measurable reduction in matched inbound intents |
| Outreach to suppressed player (self-excluded, cooling-off, RG-flagged) | **0** |
| Marketing permission violations | **0** |

---

# Part 6 — Open questions

1. **Channels and helpdesks** — determines the adapter layer and which rich affordances are available. Next discussion.
2. **Back-office diversity** — how many distinct platforms must we read from, and do they expose the payment, KYC, and bonus state the baseline procedures need? **The biggest unknown in the plan.** FR-10 and FR-73 are designed to survive a bad answer, but the baseline library's value depends on it.
3. **Model hosting and data residency** — whether player conversation content may leave our infrastructure, and to which jurisdiction. Must be answered before any spike.
4. **Identity assurance** — what satisfies FR-16 per jurisdiction. Under-specifying this is a data-protection incident.
5. **Procedure authoring surface** — YAML, a generated UI, or structured prose the engine compiles. Determines whether NFR-29 is real or aspirational. Also the demo that wins deals.
6. **Tenant isolation model** — logical tenancy with enforced tenant scoping, versus regional cells. Residency requirements may force cells regardless, which makes harder isolation cheaper than it first appears.
7. **Design partner** — which operator, which markets, what volume. A single-market first tenant is a materially easier V1.
8. **Pricing model** — per-resolution, per-seat, or per-volume changes what we must meter from day one.

---

# Part 7 — Competitive context

Derived from a competitor's published sales guide (`Competitor.pdf`, 7pp). Sales collateral — every figure is self-reported and unaudited. Treat as a statement of **buyer expectation**, not of achieved engineering.

| Their claim | Page | Our position |
|---|---|---|
| 80–90% automation; 91% in a published case study | 3, 5 | Not matchable in V1 (read-only). Targeted in V2. |
| 4.8+ CSAT | 3, 5 | Match in V1. |
| Declarative, human-readable procedure workflows | 4 | **Match, and beat.** The architectural centre of gravity (§2.1). |
| Preconditions block unsafe starts; forbidden actions | 4 | Match in V1 (FR-2, FR-3). |
| RG monitoring on 100% of interactions | 4, 6 | **Match in V1. Non-negotiable** (FR-47). |
| SOC2 Type II, PII masking, zero retention, no training on customer data | 4 | Posture in V1 (NFR-16, NFR-18); certified in V2 (NFR-25). |
| 120+ languages | 6 | Match in V1 (FR-24). Their case study spans UK, DE, FR, NL, AU, IT, ES. |
| Full audit logging | 4 | Match in V1 (NFR-22 – NFR-24). |
| Back office, helpdesk, CRM, chat integrations; API-first | 4, 5 | V1, behind the adapter layer (FR-78). |
| Proactive CRM: churn prevention, milestone bonuses, RG flagging | 6 | V3 (FR-67 – FR-71). |
| Support insights feeding product and UX teams | 6 | V2 (FR-92). |

**Their worked example** (p4), reproduced because it is our acceptance benchmark (FR-14) — *"Where Is My Deposit?"*: verify identity → query payment provider API → cross-reference CRM and back-office balance logs → determine status → explain timing *or* initiate refund *or* escalate to payments with context → log and update ticket.

**The gap we are aiming at:** the competitor competes on action execution and compliance guardrails, not on conversation quality or context depth. Their guide says nothing about answer accuracy, retrieval grounding, emotional handling, or how procedures are validated.
