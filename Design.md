# IGamingSupportAgent — Design Document

**Draft v0.3 · 7 August 2026**

Source of requirements: `Competitor.pdf` — Cevro AI, *"AI Agents for Customer Support in iGaming"* (7 pp). This design targets feature parity with what that guide claims, treated as the statement of buyer expectation. Page references (p2–p7) point to that document. No other internal documents inform this revision.

Deployment model: **single-operator deployment** — one instance per iGaming operator, connected to that operator's back office, CRM, and payment stack. Multi-tenancy is out of scope.

Integration model: **synchronous request/response API**. The operator's chat front-end calls our Agent API with the player's message and receives the agent's response in the same call. No webhook ingestion, no event queue, no asynchronous delivery machinery — the operator owns the channel; we own the answer.

---

## 1. What the competitor claims, and what this design does about it

| Cevro claim | Source | Design response |
|---|---|---|
| "Chatbots answer questions. AI agents take action" — check eligibility, apply bonus, update CRM, confirm, in one conversation | p3 | Full action-taking agent: read **and write** steps enabled from day one, inside guardrails (§4, §5) |
| 80–90% automation, 91% in case study | p3, p5 | Target: ≥80% full automation at steady state (§10) |
| 4.8+ CSAT | p3, p5 | Target ≥4.8 on agent-handled conversations |
| AI Procedures (AIPs): declarative, human-readable workflows executed step by step — not decision trees, not fine-tuning | p4 | Procedure engine is the architectural core (§4) |
| Preconditions block unsafe starts; forbidden actions prevent errors | p4 | First-class fields in every procedure (§4.1) |
| RG monitoring on 100% of interactions | p4, p6 | Two-layer RG screen on every inbound message (§7) |
| SOC2 Type II, PII masking, zero data retention, no training on customer data, every action logged | p4 | Trust layer (§7); SOC2 certification runs as a parallel track |
| Connects to back offices, helpdesks, chat platforms, CRM, Slack; API-first | p4, p5 | MCP tool layer, one server per domain (§5); Slack reporting included |
| 120+ languages, responses tailored to player history | p6 | Language design in §6; personalisation from live account reads |
| Proactive CRM: churn prevention, milestone bonuses, RG flagging | p6 | Proactive engine (§8) |
| Support insights feeding product/UX teams | p6 | Falls out of the analytics store (§10) |
| Impact "in weeks"; ROI calculator | p7 | Onboarding path (§11); ROI metrics metered from day one (§10) |
| Case-study scale: 80–100k chats/month, 7 markets | p5 | Cost and capacity model built at 10k and 100k conv/month (§9) |

One deliberate divergence: Cevro markets autonomy. This design delivers the same outcomes with **bounded autonomy** — a deterministic procedure engine decides which tools run; the LLM never free-plans actions. In a licensed vertical that is the safer engineering posture and the stronger audit story, and it is invisible to the player.

---

## 2. System architecture

```
        ┌───────────────────────────────────────────────────────────┐
        │ Operator chat front-end (widget / app / helpdesk console)  │
        └───────────────────────┬───────────────────────────────────┘
             POST /messages     │        ▲  response: reply │ escalate
             (sync, one call)   ▼        │  + structured context
   ┌────────────────────────────────────────────────────────────────┐
   │ AGENT API  (auth · rate limit · request ID · kill switch)      │
   │        │                                                       │
   │        ▼                                                       │
   │   ORCHESTRATOR                                                 │
   │     guard sequence · conversation state (by conversation_id)   │
   │        │                                                       │
   │        ▼                                                       │
   │   PROCEDURE ENGINE (AIP equivalent)                            │
   │     select procedure → preconditions → typed steps →           │
   │     verify outcomes → log every step                           │
   │        │ model slots               │ tool slots                │
   │        ▼                           ▼                           │
   │   MODEL GATEWAY              MCP TOOL LAYER                    │
   │   tiered LLM routing,        player-account · payments · KYC · │
   │   caching, fallback          bonus · CRM · knowledge ·         │
   │                              compliance/RG · slack-reporting · │
   │                              audit                             │
   │        │                                                       │
   │        ▼                                                       │
   │   RESPONSE GATE (sanitise · forbidden-language screen ·        │
   │                  kill switch re-check) → return to caller      │
   └────────────────────────────────────────────────────────────────┘
              │                              │
              ▼                              ▼
      Model providers                Operator back office / PSPs /
      (Anthropic + OpenAI fallback)  CRM APIs

   Cross-cutting: audit log · config service · kill switch ·
   observability · quality review queue · PROACTIVE ENGINE (§8)
```

### 2.1 The Agent API

One primary endpoint, called by the operator's front-end whenever a player sends a message:

```
POST /v1/conversations/{conversation_id}/messages
{
  "player_external_id": "op-123456",
  "text": "where is my deposit?",
  "lang_hint": "de",            // optional
  "mode": "live" | "dry_run",   // dry_run returns would-have-said, sends nothing
  "request_id": "uuid"          // idempotency key, retries return the same response
}

200 →
{
  "type": "reply" | "escalate" | "fixed_copy" | "decline",
  "text": "…",                          // player-facing, already sanitised
  "actions_taken": [ { "action": "trigger_refund", "ref": "…", "verified": true } ],
  "escalation": { "target": "payments", "context_pack": { … } },   // when type=escalate
  "meta": { "intent": "…", "confidence": 0.93, "procedure": "where-is-my-deposit@1" }
}
```

Contract points: the response arrives in the same HTTP call (streaming via SSE optional for long generations); a hard **timeout ceiling of 15 s** — if the pipeline can't finish, the API returns `type: escalate` with whatever context was gathered, never an error the front-end has to invent copy for; `request_id` makes retries safe — a duplicate request returns the stored response and **never re-executes write actions**; escalation is a *return value*, not a side effect — the operator's front-end routes it to their human team however they choose (their helpdesk, their queue), which keeps their helpdesk out of our critical path.

Because the operator calls us, an entire class of machinery disappears: no webhook signature verification, no event queue, no delivery retries on our side, no reconciliation sweep, no out-of-order handling, no debounce (the front-end decides when to call). What remains is a clean request-scoped pipeline.

### 2.2 Components

| Component | Responsibility |
|---|---|
| **Agent API** | AuthN/authZ (operator API keys), per-key rate limiting, request-ID idempotency, kill-switch short-circuit, timeout enforcement |
| **Orchestrator** | Guard sequence, conversation state keyed by `conversation_id` |
| **Procedure engine** | Execute declarative procedures step by step; dry-run mode for safe rollout |
| **Model gateway** | Sole egress for LLM calls: routing table, prompt caching, timeouts, cross-vendor fallback, token metering |
| **MCP tool layer** | All reads of operator systems and all writes — account actions, CRM updates (§5) |
| **Response gate** | Sanitisation, forbidden-language screen, kill-switch re-check — the last thing between generated text and the caller |
| **Proactive engine** | Scheduled outreach: churn signals, milestones, RG flags (§8) — the one part of the system that initiates rather than responds; delivers via an operator-provided callback or CRM campaign trigger |

Conversation state machine (kept server-side, keyed by `conversation_id`): `active → answered → active…`, `active → handed_over` (the front-end tells us via a `handover` flag or a state endpoint when a human takes the conversation; while handed over, calls return `type: decline` so the agent never talks over a human), `active → suppressed` (kill switch or RG policy), `any → closed`.

### 2.3 Guard sequence, in detail

Every request passes through nine gates, in a fixed order, before any generated text can return. Two design rules govern the sequence. **Cheap before expensive:** gates that need no model call run first, so bad requests cost microseconds, not tokens. **Fail toward humans, never toward silence or a guess:** every gate's failure path returns a well-formed response (`decline`, `fixed_copy`, or `escalate` with context) — the front-end never has to handle an error by inventing copy.

| # | Gate | Checks | On trigger | Cost |
|---|---|---|---|---|
| 0 | **Kill switch** (API edge) | Global flag; write-freeze flag noted for later | Global on → `escalate` immediately, request still logged in shadow | ~0 |
| 1 | **Auth + rate limit** (API edge) | Operator API key valid; per-key rate budget | 401 / 429 — the only gates allowed to return HTTP errors, since no player is waiting on copy | ~0 |
| 2 | **Idempotency** | `request_id` seen before? | Return the stored response verbatim; no re-execution, ever — this is what makes front-end retries safe with write actions in play | 1 lookup |
| 3 | **Conversation state** | `closed`, `suppressed`, or `handed_over`? | `decline` (with reason for the front-end); handed-over stickiness lives here — only an explicit hand-back reopens the conversation to the agent | 1 lookup |
| 4 | **Identity + RG status** | `resolve_player` on the operator back office; self-excluded or cooling-off? | Self-excluded/cooling-off → conversation moves to `suppressed` under the RG policy: approved fixed copy + escalation to a trained human. Runs *before* classification so no procedure can ever trigger for these players. Identity resolution failure → `escalate` (never answer account questions for an unresolved player) | 1 tool call |
| 5 | **Classification** | Single Haiku call: intent, sentiment, **RG risk**, confidence — plus the deterministic RG lexicon, run in parallel, results OR-ed | RG signal from either layer → `fixed_copy` in the player's language + `escalate` to a trained human, zero generated text. This is the 100%-of-interactions RG guarantee (p4): it runs on every message, unconditionally, before any answering logic | 1 model call (~$0.0015) |
| 6 | **Always-escalate intents** | Classified intent in the fixed set: complaints/disputes, AML/source-of-funds, chargebacks, legal threats, self-exclusion requests | `fixed_copy` acknowledgement + `escalate`. Generated text on these intents is a Sev-1 by definition, so the gate is a hard allowlist check, not a model judgement | ~0 |
| 7 | **Reply ceiling** | Agent replies in this conversation ≥ configured max (default ~5) | `escalate` — a conversation the agent hasn't resolved in five turns is looping, not converging; more generation makes it worse | 1 lookup |
| 8 | **Confidence threshold** | Classification confidence ≥ configured floor (per-intent overridable — account-specific intents demand more confidence than general info) | `escalate` with context pack — a low-confidence answer about someone's withdrawal is worse than a handover | ~0 |

Only after gate 8 does the procedure engine run. Two guard-like checks live *outside* this sequence by design: **write-action guardrails** (precondition re-verify, ceilings, outcome verification, two-man rule — §4.2) execute inside the procedure at each write step, because they need step context; and the **response gate** (sanitisation, forbidden/inducement-language screen, kill-switch re-check) runs after generation, because it inspects output, not input. So generated text faces gates on both sides: nine before a model ever speaks, three more between the model and the player.

Ordering subtleties worth recording: gate 4 must precede gate 5 because self-excluded players must never reach even classification-driven flows — their handling is a matter of registry state, not model judgement. Gate 5's RG screen must precede gate 6's intent routing because RG risk is orthogonal to intent (a deposit question can carry a distress signal; the distress wins). And gate 2 must precede everything stateful, because a retried request must return the original response even if the conversation has since moved to `handed_over`.

### 2.4 Classification design (gate 5)

One Haiku call per inbound message returns a single structured object; the deterministic RG lexicon runs in parallel and its result is OR-ed in. The system prompt (taxonomy definitions + few-shot examples per intent + output schema) is stable and prompt-cached; only the conversation is fresh tokens.

**Output schema:**

```json
{
  "intent":        { "primary": "deposit_missing", "secondary": "bonus_eligibility" },
  "confidence":    0.93,
  "sentiment":     { "valence": "negative", "frustration": 0.7 },
  "rg_risk":       { "level": "none|low|medium|high", "signals": ["chasing_losses"] },
  "language":      "de",
  "entities":      { "amount": "200 EUR", "method": "Visa", "when": "yesterday" }
}
```

**Intent taxonomy.** Two levels: a stable category level that procedures and analytics hang off, and specific intents under each. The taxonomy ships as baseline and is operator-extensible (add intents; the always-escalate defaults cannot be removed).

| Category | Intents | Routing |
|---|---|---|
| **payments** | `deposit_missing`, `deposit_delayed`, `deposit_failed`, `withdrawal_status`, `withdrawal_delayed`, `withdrawal_rejected`, `payment_method_question` | Procedures with account reads; the core volume drivers (p3, p5) |
| **account** | `login_issue`, `account_locked`, `password_reset`, `details_update`, `account_closure_request` | Procedures; `account_closure_request` gets special handling (see §2.5) |
| **kyc** | `verification_status`, `document_rejected`, `document_help`, `source_of_funds_query` | Procedures; `source_of_funds_query` → always-escalate (AML) |
| **bonus** | `bonus_missing`, `bonus_eligibility`, `wagering_progress`, `bonus_terms_question` | Procedures incl. `issue_bonus` flows |
| **gameplay** | `game_crashed`, `bet_settlement_question`, `free_spins_missing` | Procedures; settlement *disputes* reclassify to complaints |
| **responsible_gambling** | `self_exclusion_request`, `limit_setting_request`, plus the orthogonal `rg_risk` signal | Always-escalate / RG policy (limit-setting may be a permitted procedure per market policy) |
| **complaints_legal** | `formal_complaint`, `financial_dispute`, `chargeback`, `legal_threat`, `regulator_mention` | Always-escalate |
| **safeguarding** | `underage_mention`, `third_party_account`, `harm_or_crisis` | Always-escalate, highest priority |
| **general** | `promotions_info`, `rules_question`, `payment_methods_info`, `smalltalk_or_other` | Knowledge-only procedure; no account reads |

**Semantics that matter:**

- **Primary intent selects the procedure; secondary intent** is carried into the respond step so the answer can acknowledge it ("your deposit is X — and on the bonus, you need Y more wagering"), and is logged for analytics. If the secondary intent is an always-escalate intent, it wins over the primary.
- **Confidence is per-decision, calibrated, and thresholded per intent.** Account-specific intents (payments, bonus) need ≥0.85 to proceed; general info can proceed at ≥0.7; anything below its floor escalates. Thresholds are config, tuned against the QA sample where predicted confidence is compared with human-judged correctness.
- **`smalltalk_or_other` is a real class**, not a dumping ground: it routes to knowledge retrieval, and if retrieval returns nothing usable, the reply is an honest "let me get you to a colleague" escalation — the classifier is never forced to pick a business intent for an unclassifiable message.
- **Entities are extracted in the same call** (amounts, methods, dates, bonus codes) so procedures start with their parameters filled — one model call, not two.
- **`rg_risk` is orthogonal to intent** and evaluated on every message regardless of what the message is "about". Signal vocabulary: explicit distress ("can't stop", "gambling problem"), chasing losses, borrowed/rent money mentions, spend-or-time remorse, self-harm language. Levels: `high` (or any lexicon hit) → RG policy immediately (fixed copy + trained-human escalation); `medium` → Sonnet secondary confirm (§6.1), then RG policy if confirmed; `low` → logged, softens reply tone, never marketing-adjacent content in the response. Self-harm language skips all secondary checks and goes straight to crisis copy with market-appropriate helplines.
- **Language** is detected deterministically (fastText) pre-call; the classifier output field is a cross-check. Mismatch lowers effective confidence.

### 2.5 Always-escalate intents (gate 6)

The defining property: **on these intents, zero generated text ever reaches the player.** The agent sends operator-approved, human-translated fixed copy (from the Compliance MCP, keyed by intent × market × language) and returns `escalate` with a context pack. This is enforced structurally — the respond step is unreachable once gate 6 fires — not by prompting.

| Intent | Why generated text is prohibited | Fixed copy does | Escalation target · SLA |
|---|---|---|---|
| `self_exclusion_request` | Statutory duty of care; a wrong or delayed response is a licence event. Ambiguity ("close my account, I gamble too much") must resolve toward exclusion | Confirms the request is understood and protection is being applied; no retention language of any kind | RG-trained team · minutes. Operators may enable a protective auto-action: immediate temporary cool-off applied on classification, before the human confirms the full exclusion — delay is the risk, so the safe action is taken first |
| `limit_setting_request` | Market-dependent: some regulators require limits be applied immediately and without discouragement | Acknowledges; where market policy permits, a guided limit-setting procedure (write-enabled) may run instead — the per-market policy from the Compliance MCP decides | RG team · same day |
| `formal_complaint`, `financial_dispute` | Complaint handling is a regulated procedure (ADR paths, response deadlines); anything generated becomes part of the record | Acknowledges receipt, states the complaint process and timeline verbatim from approved copy | Complaints handler · per regulatory clock |
| `chargeback` | Active financial/legal position; an off-hand sentence can concede liability | Neutral acknowledgement only | Payments/risk team · hours |
| `legal_threat`, `regulator_mention` | Litigation/regulatory posture; statements are discoverable | Neutral acknowledgement | Legal · same day |
| `source_of_funds_query` (AML) | **Tipping-off risk**: discussing why funds are being reviewed can itself be an offence in most licensed markets | States only that a review is in progress and documents required, from approved copy | AML/compliance officer · per case |
| `account_closure_request` | Frequently disguised self-exclusion; retention-flavoured generated text at this moment is both an RG failure and in some markets an inducement breach | Confirms closure path; explicitly offers the self-exclusion alternative in approved wording | Human agent · same day |
| `underage_mention`, `third_party_account` | Potential criminal/licence matter; evidence handling | Neutral acknowledgement, no accusatory language | Compliance · immediate, with proposed account freeze queued for human approval |
| `harm_or_crisis` | Wellbeing; only vetted crisis copy is acceptable | Market-appropriate crisis resources and helplines, human follow-up | RG-trained team · immediate |

**Mechanics and safeguards:**

- **Two independent triggers.** The gate fires on the classified intent *or* on a deterministic keyword/pattern layer (per language, per market) that runs regardless of the classifier — "I will sue" reaches gate 6 even if the classifier read the message as a withdrawal question. Either trigger alone is sufficient; they are never AND-ed.
- **Bidirectional precedence.** A secondary always-escalate intent beats a primary business intent (§2.4); an always-escalate trigger also beats a high classifier confidence on something else.
- **Context pack still runs.** Escalation carries the transcript, classification, any account state already read, and the fired trigger — the human starts informed, not cold.
- **The set is a floor, not a default.** Operators can add intents to the set (e.g. VIP conversations) but cannot remove the baseline entries; that floor is part of our compliance posture, not operator preference.
- **Leakage is a Sev-1** and is tested for: a red-team corpus of paraphrased, misspelled, code-switched, and multi-intent phrasings for every always-escalate intent runs in CI against every classifier-prompt or lexicon change, with a zero-leak pass bar before release.
- **Fixed copy is versioned** in the Compliance MCP with market-by-market legal review dates; a market without reviewed copy for an intent falls back to escalate-with-no-message plus an internal alert, never to another market's copy or machine translation.

---

## 3. Integration surfaces

Cevro's pitch is "connects directly to your existing iGaming stack" (p4). In the sync model the operator's front-end is the caller, not an integration we ingest from — our integrations are the systems the agent reads and acts on:

| Surface | Direction | Notes |
|---|---|---|
| **Operator chat front-end** | Inbound caller | Calls the Agent API (§2.1); owns rendering, human handover routing, and channel choice. Not our code |
| **Back office / gaming platform** (SOFTSWISS, EveryMatrix, White Hat class — p5 logos) | Read + write | Player state, transactions, KYC, bonuses; write actions gated by guardrails |
| **CRM** | Read + write | Player history/preferences in; interaction outcomes and campaign triggers out (p3 "updates the CRM") |
| **Payment providers** | Read | Transaction status cross-reference (p4 deposit example) |
| **Helpdesk / ticketing** (Zoho in the case study, p5) | Write (optional) | `update_ticket` and escalation context can be pushed to the operator's helpdesk if they want it there rather than handled by their front-end |
| **Slack** | Write | Operational reporting: escalations, daily digests, anomaly alerts (p5) |

All read/write surfaces are wrapped as MCP servers (§5), so the procedure engine sees one uniform, schema-checked tool interface, and swapping a back office or CRM is a driver change, not an engine change.

---

## 4. The procedure engine (our AIP)

### 4.1 Format

Declarative YAML, versioned, diffable, authored and reviewed by CS leads and compliance — not engineers (matches p4: "describe processes in human-readable form"). Cevro's own worked example, as our procedure:

```yaml
procedure: where-is-my-deposit          # p4 example, full parity
version: 1
trigger:
  intents: [deposit_missing, deposit_delayed]
preconditions:                          # "preconditions block unsafe starts"
  - identity_verified
  - account_status is active
steps:
  - read_transactions:   { window: 14d, types: [deposit] }
  - get_psp_status:      { for: latest_unmatched }
  - read_player_state:   { fields: [balance, currency] }   # cross-reference
  - classify:            { task: reconcile_deposit, output: [status, txn] }
  - branch:
      completed: respond  { hint: confirm_timing_and_balance }
      pending:   respond  { hint: explain_pending_timing }
      failed:
        - trigger_refund: { txn: matched, ceiling: txn.amount }   # WRITE
        - respond:        { hint: refund_initiated }
      not_found: escalate { target: payments, include: [claim, window] }
  - update_ticket:       { status: resolved_or_escalated }        # WRITE
  - log:                 { outcome: full }
forbidden:                              # "forbidden actions prevent errors"
  - issue_goodwill_bonus
  - state_settlement_time_without_psp_status
escalation:
  on_confidence_below: threshold
  on_action_failure:   payments_team_with_context
```

### 4.2 Write actions and guardrails

The differentiating claim is action-taking (p3). Write steps are enabled from v1 with four mandatory guardrails, enforced by the engine, not by procedure authors:

1. **Precondition gate** — a write step never runs unless its declared preconditions re-verify against live state at execution time (not selection time).
2. **Ceilings** — every monetary action carries a per-action and per-player-per-day ceiling; exceeding routes to human approval instead of failing silently.
3. **Outcome verification** — after every write, the engine re-reads the affected state and confirms the expected transition; mismatch → freeze the procedure run, escalate with context.
4. **Two-man rule tier** — actions above configured thresholds (refund > X, bonus > Y) execute as *proposed actions*: the agent prepares it, a human clicks approve, the agent completes and confirms to the player. Still "one conversation" from the player's view.

### 4.3 Baseline procedure library

Derived from the contact types Cevro names (p3, p4, p5): where-is-my-deposit (acceptance benchmark — it is their own worked example), withdrawal status, account unlock, document verification, bonus request / bonus issue (eligibility check → `issue_bonus` within ceilings), wagering-progress explanation, general information (knowledge only), plus the fixed-copy RG and complaint flows (§7). Release path for any procedure change: edit → schema + tool validation → dry-run against replayed real conversations → approval → live.

---

## 5. MCP tool layer

Every integration domain is an MCP server. Rationale: one protocol between engine and world; tool schemas the procedure validator checks at author time; credential injection at the server boundary (the engine and models never see credentials); per-tool rate budgets so the agent can't hammer the operator's back office; uniform audit hooks on every call; growing vendor ecosystem of ready-made MCP servers for helpdesks and CRMs we can adopt rather than build.

### 5.1 Server catalog

**Player Account MCP** (back-office driver behind a fixed surface)
- `resolve_player`, `get_player_state`, `get_account_status`, `get_limits`
- writes: `unlock_account` (cause-class-gated), `resend_verification`

**Payments MCP**
- `get_transactions`, `get_withdrawal_status`, `get_psp_status`
- writes: `trigger_refund` (ceiling-gated, outcome-verified)

**KYC MCP**
- `get_kyc_status`, `get_verification_requirements`
- writes: `request_document_reupload`, `submit_for_review`

**Bonus MCP**
- `get_bonus_state`, `get_bonus_terms`, `check_eligibility`
- writes: `issue_bonus` (eligibility must pass in same run; ceiling-gated), `apply_bonus`

**CRM MCP** (p3 "updates the CRM"; p6 retention)
- `get_player_history`, `get_preferences`, `get_engagement_signals`
- writes: `record_interaction`, `update_player_notes`, `trigger_campaign`

**Knowledge MCP**
- `search_knowledge(query, market, lang)`, `get_article` — retrieval over T&Cs, help centre, bonus terms; market- and language-filtered so answers cite the copy that actually applies to this player

**Compliance & RG MCP** (deterministic, never model-mediated)
- `check_rg_status` (self-exclusion / cooling-off), `get_approved_copy(key, market, lang)` (human-translated fixed texts), `get_market_policy`, `evaluate_forbidden(text)` (inducement/promotional-language screen before send)

**Ticketing MCP** (optional — only if the operator wants outcomes in their helpdesk)
- `update_ticket`, `add_note`, `tag`, `set_attributes` — player-facing text always returns via the Agent API, never through this server

**Slack Reporting MCP** (p5)
- `post_escalation`, `post_daily_digest`, `post_alert`

**Audit MCP**
- `log_step`, `record_outcome`, `record_dry_run` — append-only

### 5.2 Data minimisation

Each procedure declares what data classes it may touch; the tool layer redacts everything else from results before they enter a prompt. That is the concrete mechanism behind the "PII masking" claim (p4): masking happens at the tool boundary, so the model never receives fields the procedure didn't declare.

---

## 6. LLM strategy

### 6.1 Tiered routing

Models fill exactly two step types — `classify` and `respond`. Everything else is deterministic. Small model on every message; mid model only when a procedure reaches a respond step.

| Call | Model | Price (per Mtok, Aug 2026) | Why |
|---|---|---|---|
| Intent + sentiment + RG + confidence (every inbound message) | **Claude Haiku 4.5** | $1 / $5 | High-volume structured classification; strong multilingual behaviour for the 120+ language target; fast enough for instant-reply UX |
| Player-facing response generation | **Claude Sonnet 5** | $3 / $15 list ($2/$10 promo to 31 Aug 2026) | Where empathy + accuracy (4.8 CSAT, p3 "empathetic conversations") is bought; grounded on tool outputs and template hints |
| Escalation context pack | Claude Haiku 4.5 | — | Human-facing summary, not player-facing |
| RG secondary confirm (flagged only) | Claude Sonnet 5 | — | Precision pass; recall guaranteed upstream (§7) |
| QA sampling / dry-run judging (async) | Claude Sonnet 5, Batch API | −50% | Offline quality loop |
| Language detection | fastText/CLD3 (non-LLM) | free | Deterministic, microseconds |
| Cross-vendor fallback | GPT-5.6 Terra ($2/$12) respond · GPT-5.6 Luna ($0.20/$1.20) classify | — | Availability incidents; prompts kept provider-portable; model-in-force is part of the audit record |

Rejected for now: self-hosted open weights — multilingual quality and RG-recall validation burden outweigh the residency benefit for a first operator; regional hosted inference (EU processing from either vendor) is the answer if the operator's regulator requires it.

**Zero-retention posture** (p4 parity): provider agreements with no training on customer data and no retention beyond abuse windows, contractual before the first live conversation.

### 6.2 Languages (120+, p6)

Generation and classification are natively multilingual via the chosen models. The critical carve-out: **regulated fixed copy — RG interventions, disclosure, escalation acknowledgements — is human-translated per market and served from the Compliance MCP, never machine-generated at runtime.** Languages are tiered: launch markets get human-reviewed template hints and QA sampling quotas; long-tail languages run generation with tighter confidence thresholds and higher escalation propensity.

### 6.3 Latency budget

The whole pipeline runs inside one HTTP request, so the budget is the caller's patience:

```
auth + guards + state load ............. < 100 ms
classify (Haiku, cached system) ........ 0.5–1.5 s
tool reads (parallelised) .............. 0.5–2 s
respond (Sonnet, ~350 tok) ............. 2–4 s
write action + outcome verify .......... 1–3 s   (when procedure writes)
response gate (sanitise + screens) ..... < 300 ms
─────────────────────────────────────────────────
informational answer ................... ~3–7 s p95
action-confirming answer ............... ~5–11 s p95
hard timeout → return escalate ......... 15 s
```

For dead-air UX the operator's front-end shows its own typing indicator while the call is in flight — a real advantage of the sync model: the front-end always knows the agent is working. SSE streaming of the `respond` tokens is available so first words appear in ~2–3 s. Prompt caching (system prompts, procedures, copy packs at 10% of input price) is load-bearing for both cost and latency.

---

## 7. Trust layer

Parity with p4's "Trust Layer", as engineering rather than assertion:

- **RG monitoring on 100% of interactions.** Two-layer screen on every inbound message: a deterministic multilingual lexicon/pattern layer ORed with the Haiku classification. Either layer triggering → the conversation leaves normal procedure flow: approved fixed copy in the player's language, immediate escalation to a trained human, **zero generated text**. Self-excluded/cooling-off status is checked at identity resolution, before any procedure triggers.
- **Always-escalate intents** (complaints/disputes, AML/source-of-funds, chargebacks, legal threats, self-exclusion requests): fixed acknowledgement copy only; generated text on these intents is treated as a Sev-1.
- **PII masking**: at the tool boundary per declared data classes (§5.2).
- **Zero retention / no training**: provider-contractual; conversation content encrypted at rest on our side with operator-scoped keys and a defined retention policy per market.
- **Full audit**: every step execution recorded — inputs/outputs post-redaction, model + version + cache state for model calls, procedure version, guardrail decisions, action outcomes with before/after state reads. Append-only. This is what "every action is logged for full audits" (p4) has to mean to survive a regulator.
- **Disclosure**: the agent identifies as automated in its first message and is software-named; a human path is always available and honoured immediately, with no deflection attempt.
- **SOC2 Type II**: build to the control set from day one (access control, change management, audit trails already fall out of the design); certification is a parallel calendar track, not a design item.
- **Kill switch**: config-service flag checked at the API edge and re-checked at the response gate; when on, every call returns `type: escalate` immediately (the front-end routes to humans) while classification and audit keep running in shadow. A second, separate flag freezes write actions only — degrading the agent to read-and-answer without silencing it.

---

## 8. Proactive engine (AI support agent as CRM, p6)

Separate from the reactive loop, sharing the tool layer and trust layer:

| Play | Trigger | Flow |
|---|---|---|
| **Churn prevention** | Engagement-signal drop from CRM MCP | Eligibility + RG check → personalised re-engagement message or `trigger_campaign`; never a bonus without `check_eligibility` passing |
| **Milestone bonuses** | First deposit, VIP tier reached | `issue_bonus` within ceilings → congratulation message |
| **Proactive RG flagging** | Behaviour-pattern signals | RG policy flow — fixed copy, human escalation; proactive RG outreach is always human-approved before send |
| **Proactive status updates** | e.g. withdrawal cleared, documents approved | Short confirmation message; suppresses a future inbound ticket |

Two hard gates on everything proactive: **marketing-permission check** (a player who opted out of promotional contact gets service messages only, and bonus-bearing outreach is promotional) and **market-policy check** (inducement rules differ by market; the Compliance MCP's `get_market_policy` gates each play per player market). Every proactive send is rate-capped per player and logged like any other action.

---

## 9. Cost per conversation

### 9.1 Assumptions

Average reactive conversation: 3 inbound player messages (debounced), 2 substantive generated replies, 0.7 write actions, ~15% escalation rate (consistent with 80–90% automation), 5% RG secondary checks. Steady-state prompt caching (~55% of respond input cached). List prices, not promo.

### 9.2 Model cost

| Call | Model | Calls/conv | Cost/call | Cost/conv |
|---|---|---|---|---|
| Classify + RG screen (1.2k cached + 0.6k fresh in, 150 out) | Haiku 4.5 | 3 | $0.0015 | $0.0045 |
| Respond (3.5k cached + 4.5k fresh in — incl. tool/action context, 350 out) | Sonnet 5 | 2 | $0.0198 | $0.0396 |
| Action outcome summarisation (2k in, 150 out) | Haiku 4.5 | 0.7 | $0.0028 | $0.0020 |
| Escalation context pack (3k in, 300 out) | Haiku 4.5 | 0.15 | $0.0045 | $0.0007 |
| RG secondary confirm | Sonnet 5 | 0.05 | $0.0075 | $0.0004 |
| Embeddings + retrieval | — | — | — | $0.0005 |
| **Subtotal** | | | | **$0.0477** |
| Cache writes, retries, dry-runs, QA sampling (+20%) | | | | $0.0095 |
| **Model cost per conversation** | | | | **≈ $0.057 — call it $0.06** |

Worked respond call: 3,500 cached × $0.30/M = $0.00105 + 4,500 fresh × $3/M = $0.0135 + 350 out × $15/M = $0.00525 → $0.0198.

### 9.3 All-in, both scales

| | 10k conv/mo | 100k conv/mo (case-study scale, p5) |
|---|---|---|
| Model spend | ~$570 | ~$5,700 |
| Infrastructure (API compute, Postgres, cache, vector store, secrets, observability) — sync single-operator stack: no queue, no webhook/reconciler machinery | ~$1,200 | ~$3,800 |
| **Total** | **~$1,770/mo** | **~$9,500/mo** |
| **Cost per conversation** | **≈ $0.18** | **≈ $0.095** |

Proactive outreach is additive and cheap: one Haiku eligibility/personalisation call + one Sonnet message ≈ $0.012 per outreach at similar token shapes.

### 9.4 Sensitivity

| Scenario | Model cost/conv |
|---|---|
| Base (Haiku + Sonnet, list) | $0.057 |
| Sonnet promo ($2/$10, to 31 Aug 2026) | $0.041 |
| Budget floor (Gemini 3.5 Flash-Lite classify + Gemini 3.6 Flash respond) | ~$0.02 — not recommended until wrong-answer rate on account-specific replies is proven at target |
| Premium respond (Opus 5, $5/$25) | ~$0.10 — reserve for QA judging |

**The economics against the claim:** a human contact costs an operator roughly $3–5. At ≈$0.10 all-in and 85% automation on 100k conversations, ~85k contacts are deflected for under $10k/mo — the arithmetic behind Cevro's ROI-calculator numbers (p7, $381k/yr savings at 80%), reproducible in our dashboard from metered actuals rather than a marketing slider. Token usage and resolution outcomes are metered per conversation from day one.

---

## 10. Observability, ROI, and insights

- **Dashboard**: volume, automation rate (full vs assisted), escalation reasons, CSAT, first-response/resolution times, confidence distribution, top intents, action counts by type, proactive-play performance.
- **Targets** (parity with p3/p5 claims): ≥80% full automation at steady state, ≥4.8 CSAT, instant first response, wrong-answer rate on account-specific replies <0.5%, RG recall >99%, zero generated text on always-escalate intents, zero agent-talks-over-human incidents.
- **ROI reporting** (p7): deflected contacts × operator's loaded cost per contact, agent-hours saved, cost per contact before/after — the buyer-facing numbers, from metered data.
- **Slack reporting** (p5): escalations with context in real time; daily digest; anomaly alerts (confidence drift, escalation spikes, RG anomalies, integration failures).
- **Support insights** (p6): recurring intents, friction clusters, and product-gap signals exported for the operator's product/UX teams — nearly free once outcomes are structured.
- **Quality loop**: stratified sampling (intent × confidence × language) into human review; corrections become procedure edits, released through dry-run.

---

## 11. Rollout ("impact in weeks", p7)

1. **Week 1–2 — Connect and observe.** Back office/CRM connected read-only; the operator's front-end calls the API with `mode: dry_run` alongside their existing support flow — every response is recorded, nothing shown to players. Zero player exposure.
2. **Week 3 — Informational go-live.** Knowledge-only and read-only procedures live (general info, status explanations). RG screen live on 100% of messages from day one.
3. **Week 4–5 — Action go-live, tiered.** Write actions enabled lowest-risk first (update_ticket → resend_verification → unlock_account → issue_bonus/trigger_refund under the two-man rule), each after its dry-run history is reviewed.
4. **Week 6+ — Ceiling raises and proactive plays.** Two-man thresholds relaxed as verified-outcome history accumulates; proactive engine enabled market by market.

Dry-run mode is what makes "weeks" honest: every stage is rehearsed on real traffic before it touches a player.

---

## 12. Risks

| Risk | Mitigation |
|---|---|
| A write action misfires (wrong refund, ineligible bonus) | Live-state precondition re-check, ceilings, outcome verification, two-man rule above thresholds, write-only kill switch |
| Front-end retry after timeout re-executes a write action | `request_id` idempotency: retries return the stored response; write steps never re-execute for a seen request_id |
| Long tool chains blow the 15 s sync ceiling | Parallel tool reads, per-step timeouts, streaming first tokens early; past the ceiling the API returns a clean `escalate` with gathered context, never a raw timeout |
| Operator front-end forgets to signal human handover | `decline` responses while `handed_over`; handover state also settable by their helpdesk automation; periodic state audit against ticket assignments where Ticketing MCP is connected |
| RG miss in a low-resource language | Deterministic lexicon ∪ model screen; per-language recall measurement; lexicon-only languages get tighter escalation thresholds |
| Back-office API lacks needed reads/writes | Capability probe in week 1; procedures degrade per capability (missing wagering API ⇒ bonus procedure downgrades to explain + escalate) |
| Bad generated answer is permanent on most channels | Confidence gating, forbidden-language screen, template hints, kill switch, QA sampling; no free text on regulated intents |
| Provider price/behaviour changes | Gateway abstraction, provider-portable prompts, monthly cost re-derivation from metered actuals |
| Proactive outreach breaches inducement rules in a market | Market-policy gate + marketing-permission gate on every play; proactive RG always human-approved |

---

## Appendix — Pricing sources (August 2026)

- Anthropic: Haiku 4.5 $1/$5 · Sonnet 5 $3/$15 list ($2/$10 promo to 31 Aug 2026) · Opus 5 $5/$25; cache hits 10% of input; batch −50% (benchlm.ai/anthropic/api-pricing)
- OpenAI: GPT-5.6 Terra $2/$12 · Luna $0.20/$1.20; cached input 10%; batch −50% (benchlm.ai/openai/api-pricing)
- Google: Gemini 3.6 Flash $1.50/$7.50 · 3.5 Flash-Lite $0.30/$2.50 · 3.1 Pro $2/$12 (benchlm.ai/google/api-pricing)
