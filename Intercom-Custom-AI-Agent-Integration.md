# Running a Custom AI Agent Inside Intercom

**A technical feasibility and design document**

Version 1.0 · Research date: August 2026 · Intercom REST API v2.14 era

---

## 1. Purpose and scope

This document evaluates one specific integration pattern: **operating a self-hosted AI agent (our own model, our own prompt, our own data) as a conversational participant inside Intercom's Messenger and Inbox.**

It answers three questions:

1. What is genuinely possible with Intercom's public API surface.
2. What is impossible, or possible only via undocumented behaviour.
3. What a production-grade implementation actually has to handle.

It is written to be independent of any particular host platform. It assumes only that we operate an internet-reachable backend service and have administrative access to an Intercom workspace.

### Out of scope

- Using Intercom's own AI agent (Fin) as the answering brain.
- Embedding Fin into a product we own via the Fin Agent API or Custom Channels API.
- Detailed model selection, prompt engineering, or retrieval design.

Those alternatives are summarised in Section 3 for context, because the decision to build a custom agent should be made with knowledge of what we are giving up.

---

## 2. Executive summary

**It works, but it is unsupported territory.**

Intercom does not offer a third-party bot integration point. Their support position, stated publicly, is that they do not natively support third-party bots, but that a custom solution using their API and webhooks is possible. Everything in this design is therefore built on general-purpose primitives (webhooks in, admin-authored replies out) that were designed for human agents and integrations, not for bots.

The practical consequences:

| Dimension | Assessment |
|---|---|
| Core message loop (receive, answer, reply) | **Fully supported.** Stable, documented, low risk. |
| Conversation control (assign, close, snooze, tag, note) | **Fully supported.** Everything a human agent can do. |
| Identity and context enrichment | **Fully supported.** External ID mapping gives us a clean key into our own data. |
| Bot presentation (avatar, "Operator" styling, typing indicator) | **Materially limited.** Our agent appears as a teammate. No thinking indicator. |
| Interactive UI (buttons, forms) | **Uncertain.** Documented but reported broken. Must be validated first. |
| Delivery reliability | **Weak by default.** We must build the durability layer ourselves. |
| Long-term stability | **Medium risk.** No contract, no deprecation guarantees, no support path. |

**Recommended posture:** build it, but stage it. Ship agent-assist (private notes to human agents) before customer-facing replies, and treat the button/UI question as a gating spike, because it determines whether the product is a free-text assistant or a guided flow.

---

## 3. Alternatives considered

Four integration patterns exist. This document covers pattern D.

**A. Fin plus Data Connectors.** Intercom's Fin remains the agent. We expose our systems as HTTP tools ("Data connectors", formerly "Custom Actions") that Fin calls mid-conversation, with API-key or OAuth client-credentials auth. Cheapest path to a working assistant. We control data and escalation logic but not the model, the prompt, or the reasoning. Billed per resolved outcome. Execution logs retained only 14 days.

**B. Fin Agent API.** Endpoints `/fin/start` and `/fin/reply` let us run Fin inside our own product surface, with events delivered by webhook (HMAC-SHA256) or SSE, including streaming reply chunks. Under managed availability (access by request). Outcome-based billing.

**C. Custom Channels API.** We own the channel (our own app UI, a messaging platform we operate) and Intercom treats it as a first-class channel, with workflows and Fin applying to it. Inverts the direction of integration: Intercom comes to us.

**D. Custom agent inside Intercom (this document).** Our model answers in Intercom's own Messenger. Maximum control, maximum build cost, no vendor AI billing, no official support.

The honest trade: A and B are faster and supported but rent the intelligence. D owns the intelligence and pays for it in engineering and fragility.

---

## 4. Architecture

### 4.1 The loop

```
   Customer types in the Intercom Messenger
                    │
                    ▼
   ┌────────────────────────────────────────┐
   │ Intercom webhook delivery              │
   │   conversation.user.created            │
   │   conversation.user.replied            │
   └────────────────┬───────────────────────┘
                    │  HTTPS POST, HMAC-SHA1 signed
                    ▼
   ┌────────────────────────────────────────┐
   │ Ingress service                        │
   │   verify signature                     │
   │   enqueue                              │
   │   return 200 immediately (< 5s budget) │
   └────────────────┬───────────────────────┘
                    │  durable queue
                    ▼
   ┌────────────────────────────────────────┐
   │ Orchestrator                           │
   │   dedupe + order + debounce            │
   │   guard: is a human handling this?     │
   │   resolve customer identity            │
   │   fetch context from our systems       │
   │   call model                           │
   │   decide: answer | escalate | stay out │
   └────────────────┬───────────────────────┘
                    │
                    ▼
   ┌────────────────────────────────────────┐
   │ Intercom writer                        │
   │   POST /conversations/{id}/reply       │
   │   or /conversations/{id}/parts         │
   │   retry, rate-limit aware, idempotent  │
   └────────────────┬───────────────────────┘
                    │
                    ▼
        Reply appears in the Messenger
```

### 4.2 Why the ingress and orchestrator must be separate

Intercom treats a webhook as failed if it does not receive a `200` within **5 seconds**. Model inference will routinely exceed that. If the ingress does inference inline, every slow response is a lost customer message and a step toward subscription suspension (Section 7.2).

The ingress therefore does three things only: verify, persist, acknowledge. Everything else is asynchronous.

---

## 5. What is possible

### 5.1 Receiving messages

Subscribe to the conversation webhook topics. The relevant set:

| Topic | Meaning | Why we need it |
|---|---|---|
| `conversation.user.created` | Customer started a conversation | Primary trigger |
| `conversation.user.replied` | Customer follow-up | Primary trigger |
| `conversation.admin.replied` | A human teammate replied | Stand down, human has taken over |
| `conversation.admin.assigned` | Assignment changed | Stand down or resume |
| `conversation.operator.replied` | Fin or a workflow replied | Detect and avoid collision |
| `conversation.admin.closed` | Conversation closed | End session state |
| `conversation.admin.snoozed` | Conversation snoozed | Pause session state |
| `conversation.rating.added` | CSAT submitted | Quality signal per conversation |
| `conversation.contact.attached` | Contact linked to conversation | Identity resolution |

Payload envelope:

```json
{
  "type": "notification_event",
  "id": "notif_abc123",
  "topic": "conversation.user.replied",
  "app_id": "workspace_id",
  "created_at": 1754300000,
  "delivery_status": "pending",
  "delivery_attempts": 1,
  "data": {
    "item": {
      "type": "conversation",
      "id": "1234567890",
      "...": "full conversation object"
    }
  }
}
```

`data.item` carries the conversation object, not merely an ID, so contact references and conversation state arrive with the event. Deletion topics are the exception and send minimal identifying payloads only.

**Signature verification is mandatory.** Header `X-Hub-Signature`, format `sha1=<hex>`, computed as HMAC-SHA1 over the **raw** request body using the app's `client_secret`. Compute against raw bytes before any JSON parsing or re-serialisation, and compare in constant time.

### 5.2 Sending replies

```
POST https://api.intercom.io/conversations/{conversation_id}/reply
Authorization: Bearer <access_token>
Intercom-Version: <pinned version>
Content-Type: application/json

{
  "message_type": "comment",
  "type": "admin",
  "admin_id": "493881",
  "body": "<p>Your refund was issued on 2 August and should land within 3 working days.</p>"
}
```

Notes on the fields:

- `body` accepts HTML. Keep the subset conservative (`p`, `b`, `i`, `ul`, `li`, `a`) and sanitise model output before sending, since we are injecting generated content into a rendered surface.
- `message_type` accepts `comment` (customer-visible) and `note` (internal only).
- `attachment_urls` accepts an array of file/image URLs. Documented limits differ by API version (5 on older versions, 10 on current), so verify against the pinned version.
- Notes support HTML `@mention` of a teammate, which is how the agent can page a specific human.

**The internal-note mode deserves emphasis.** Posting `message_type: "note"` puts AI output in front of the human agent and never in front of the customer. It is the same integration, the same webhooks, the same identity resolution, with essentially none of the customer-facing risk. It is the correct first release.

### 5.3 Controlling the conversation

All via `POST /conversations/{id}/parts`:

```jsonc
// Assign to a teammate or team
{ "message_type": "assignment", "type": "admin",
  "admin_id": "<acting admin>", "assignee_id": "<target admin or team>",
  "body": "optional note" }

// Close
{ "message_type": "close", "type": "admin", "admin_id": "<acting admin>" }

// Reopen
{ "message_type": "open", "type": "admin", "admin_id": "<acting admin>" }

// Snooze until a unix timestamp
{ "message_type": "snoozed", "type": "admin",
  "admin_id": "<acting admin>", "snoozed_until": 1754400000 }
```

Plus `POST /conversations/{id}/run_assignment_rules` to hand the conversation back to the workspace's normal routing rules, which is the cleanest escalation primitive: the agent does not need to know the org chart.

Conversation tags and conversation custom attributes are also writable, which is how we record machine-readable outcomes (intent classification, confidence, escalation reason) for later analysis.

### 5.4 Identity and context

Set Intercom's `external_id` (the `user_id` field on a contact) to our own customer identifier when booting the Messenger. Every subsequent webhook then contains a direct key into our systems, with no fuzzy email matching.

This is the entire competitive argument for building a custom agent rather than renting one: the agent can answer account-specific questions from live system state, with our own retrieval and our own authorisation rules, rather than from a knowledge base of articles.

### 5.5 Coexisting with Intercom's own automation

If the workspace also runs Fin or customer-facing workflows, three mechanisms keep them from talking over each other:

1. **Bot inbox.** Conversations under bot handling are held in a separate inbox so human teams are not double-handling them. Conversations exit the bot inbox when the customer completes the path, asks for a human, or after 15 minutes of inactivity when no auto-close rule applies. Note that assignments from background workflows are deferred until the conversation leaves the bot inbox.
2. **Workflow conditions.** Gate any Fin step on "conversation is not already assigned to a teammate or team".
3. **Trigger discipline.** Intercom explicitly warns against using the "Customer sends any message" trigger on a path containing a Fin step, because it fires on every customer message and causes Fin to interject into live human conversations indefinitely. The same hazard applies to our agent: our stand-down logic must be event-driven, not blanket.

A human (or our agent) sending a message into a conversation stops Fin from continuing to reply in that conversation.

---

## 6. What is not possible

| Desired capability | Status | Detail |
|---|---|---|
| Reply attributed to "Operator" or a bot identity | **Not officially possible** | No documented API. Replies are authored by a real admin teammate. |
| Typing or "thinking" indicator during inference | **Not possible** | No API. Intercom's own typing indicator only appears once that teammate has already replied in the conversation, and cannot be triggered programmatically. |
| Token streaming into the Messenger | **Not possible** | Each reply is a discrete POST. No progressive rendering of a single message. |
| Editing or deleting a sent reply | **Not possible** | A wrong or hallucinated customer-facing reply is permanent and visible. |
| Rich custom UI in the message stream | **Not possible** | Message bodies are HTML plus (possibly) quick-reply buttons. Anything richer requires a Canvas Kit Messenger app, which is a separate surface, not a message. |
| Guaranteed webhook ordering | **Explicitly not guaranteed** | Intercom states no ordering guarantee. Rapid consecutive customer messages can arrive out of order. |
| Guaranteed webhook delivery | **Not guaranteed** | 5s ack window, then one retry after 60s, then the notification is marked failed and dropped permanently. |
| A free "bot" seat | **No** | The authoring teammate is a real teammate. `has_inbox_seat: false` causes assignment and away-mode operations to fail, so a paid Inbox seat is effectively required. |

### 6.1 The "post as Operator" workaround, and why to avoid it

Community reports indicate that passing the workspace's Operator admin ID as `admin_id` on a reply produces Operator-styled output. This is undocumented, unconfirmed by Intercom staff, and carries no deprecation protection. It could break silently at any release.

Do not build on it. Create a purpose-named teammate instead (for example "Assistant" with a distinct avatar). For most use cases this is the better choice regardless, because it makes the bot's nature legible to the customer, which is increasingly a regulatory expectation rather than a style preference.

### 6.2 Presentation consequences, taken seriously

The combination of "no typing indicator" plus "no streaming" plus "authored by a teammate" produces a specific and bad UX failure mode: the customer sends a message, sees a human name attached to the conversation, and then sees nothing at all for several seconds while inference runs.

Mitigations, in order of preference:

1. **Optimistic acknowledgement.** Post a short immediate reply ("Let me check that for you") within a few hundred milliseconds, then post the substantive answer. Costs an extra message in the transcript. Effective.
2. **Latency budget enforcement.** Cap end-to-end inference and hard-fall-back to escalation past a threshold, so the dead-air period is bounded.
3. **Response shaping.** Prefer short answers that can be generated fast over comprehensive ones that cannot.

---

## 7. The grey zone and the operational envelope

### 7.1 Quick-reply buttons: unverified

The current API reference documents `reply_options` and a `quick_reply` message type on admin replies, described as mapping to buttons in the Messenger UI. However a developer report from October 2025 states that the documented endpoint returns errors in practice, with no Intercom response on the thread.

**This is the single highest-leverage unknown in the design.** Free-text-only agents and button-driven guided flows are different products with different failure modes and different quality ceilings. Validate this before any UX work begins. Treat "buttons work" as an assumption to be proved, not a feature to be planned around.

### 7.2 Webhook reliability

| Behaviour | Value |
|---|---|
| Acknowledgement window | 5,000 ms |
| Retry policy | One retry, 60 seconds after the first failure |
| After second failure | Marked failed, not retried again |
| Consecutive error throttle | More than 1,000 consecutive errors in a 15-minute window pauses delivery for 15 minutes |
| Sustained failure | Errors for more than 7 days suspends the subscription entirely |
| `410 Gone` response | Subscription disabled immediately |
| Ordering | No guarantee |

Design implications, all of which are non-optional:

- **Acknowledge unconditionally.** Return 200 as soon as the payload is verified and persisted. Never let downstream failure surface as a non-2xx to Intercom.
- **Never return 410.** A stray 410 from a proxy or CDN silently kills the integration.
- **Reconcile, do not trust.** Because a message can be dropped after two failed attempts, run a periodic reconciliation sweep that lists recently updated conversations via the API and repairs anything the webhook stream missed. This is the difference between a demo and a product.
- **Order by content, not arrival.** Sort by the conversation part `created_at`, and treat the webhook purely as a wake-up signal rather than as the source of truth. Re-fetching the conversation on wake is more robust than parsing the delivered payload.

### 7.3 Rate limits

- 10,000 API calls per minute per app.
- 25,000 API calls per minute per workspace.
- Enforced in 10-second slices, so the practical ceiling is roughly one sixth of the per-minute figure in any burst.
- Exceeding returns `429`. Current allowance is exposed in response headers.

Not a binding constraint at ordinary support volumes, but the writer component should still read the headers, back off on `429`, and never retry in a tight loop.

### 7.4 Versioning

Pin `Intercom-Version` explicitly on every request. Several details in this document (attachment count limits, quick-reply support, reply model fields) differ between versions. Unpinned requests inherit the workspace default, which can move without our involvement.

---

## 8. Reference implementation design

### 8.1 Components

| Component | Responsibility | Latency budget |
|---|---|---|
| **Ingress** | Verify HMAC, persist raw event, enqueue, return 200 | < 200 ms |
| **Orchestrator** | Dedupe, order, debounce, guard, enrich, infer, decide | Seconds |
| **Writer** | Post to Intercom with retry, backoff, idempotency | < 1 s per call |
| **Reconciler** | Periodic sweep for missed events | Minutes |
| **Session store** | Per-conversation agent state | n/a |

Splitting the writer out matters: it is the only component that mutates the customer-visible world, so it is where idempotency, rate limiting, and the kill switch belong.

### 8.2 Per-conversation state

Keyed by conversation ID:

```
status              active | handed_over | closed | suppressed
last_seen_part_id   last conversation part we have processed
reply_count         auto-replies sent in this conversation
last_reply_at       for debounce and rate shaping
intent              classified intent, for routing and analytics
confidence          last inference confidence
escalation_reason   set when handing over
```

The `handed_over` state must be sticky. Once a human replies, the agent does not resume unless a human explicitly reassigns back. Bots that reappear mid-human-conversation are the most damaging failure mode in this entire design, and Intercom's own documentation calls it out for Fin.

### 8.3 Guard sequence, evaluated before every reply

1. Is this event a duplicate of a part we already processed? Drop.
2. Is the conversation in `handed_over`, `closed`, or `suppressed`? Drop.
3. Is `admin_assignee_id` set to a human teammate? Transition to `handed_over`, drop.
4. Has the customer sent another message since this one? Debounce, process only the latest.
5. Has `reply_count` hit the configured ceiling? Escalate instead of answering.
6. Does the classified intent fall in the always-escalate set? Escalate without generating customer-facing text.
7. Is model confidence below threshold? Escalate.

Only after all seven does the agent speak.

### 8.4 Debouncing

Customers commonly send three short messages in a row rather than one long one. Without debouncing, the agent answers the first fragment before the thought is complete, and does it three times. Hold for a short window (2 to 4 seconds is a reasonable starting point) after each inbound message, and process only once the customer stops.

Combined with the absent ordering guarantee, this also incidentally repairs most out-of-order delivery.

### 8.5 Idempotency

Because retries exist at both the webhook layer and our own writer layer, every outbound reply needs a deterministic key derived from the triggering conversation part ID. Record the key before posting and check it after failures, since a timeout does not tell us whether the reply landed. Duplicate customer-facing messages are worse than a delayed one.

### 8.6 Configuration, not constants

Everything below should be runtime configuration, because all of them will need tuning against real traffic:

- inference timeout and total end-to-end latency ceiling
- debounce window
- maximum auto-replies before forced escalation
- confidence threshold for answering versus escalating
- always-escalate intent list
- authoring teammate admin ID
- pinned API version
- global kill switch (see 8.7)

### 8.7 The kill switch

A single flag that immediately stops all customer-facing replies while leaving ingestion and internal notes running. This is not optional. When a model regression starts producing bad answers to real customers, the response time that matters is measured in seconds, and a redeploy is too slow.

---

## 9. Security

- **Verify every webhook signature** against the raw body. Reject unsigned or mismatched payloads.
- **Access tokens are server-side only.** They carry broad workspace scope. Never expose to a browser or embed in a client bundle.
- **Scope minimally.** Request only the permissions the integration needs, and be aware that some webhook topics map to multiple permissions.
- **Treat conversation content as sensitive personal data.** It routinely contains identifiers, contact details, and payment references. Retention on our side needs a defined and defensible policy, especially if transcripts are sent to a third-party model provider.
- **Model provider data flow is a compliance question, not a technical one.** Whether customer conversation content may leave our infrastructure, and to which jurisdiction, should be answered before the first spike, not after.
- **Sanitise model output before posting.** We are injecting generated content into an HTML-rendering surface seen by customers.

---

## 10. Disclosure and customer experience

Because replies are authored by a teammate account, the customer has no inherent signal that they are talking to software. Several jurisdictions and sector regulators now expect explicit disclosure of automated agents.

Recommended baseline:

- Name the authoring teammate unambiguously (not a human-sounding first name).
- Disclose in the agent's first message in each conversation.
- Always offer a visible path to a human, and honour a request for one immediately, without an intervening deflection attempt.
- Never let the agent generate free text on topics with regulatory, financial, safety, or wellbeing implications. Route those to a human with zero generated text sent to the customer.

The last point is the one most likely to be argued with and least likely to be regretted.

---

## 11. Cost model

| Item | Note |
|---|---|
| Intercom seat for the authoring teammate | Paid Inbox seat effectively required, since `has_inbox_seat: false` breaks assignment and away-mode operations |
| Intercom AI charges | None. We are not using Fin, so there is no per-resolution outcome billing |
| Model inference | Per conversation turn, our cost, our control |
| Infrastructure | Ingress, orchestrator, writer, session store, reconciler |
| Engineering | The dominant cost. This is a real service with real reliability requirements, not a script |

The comparison against Fin plus Data Connectors is therefore: Fin trades a per-resolution fee for near-zero engineering; the custom agent trades sustained engineering investment for control over the model, the data, and the marginal cost curve. The crossover depends on volume and on how much of the value comes from proprietary context that Fin cannot reach.

---

## 12. Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Undocumented behaviour breaks without notice | Medium | High | Avoid all workarounds. Pin API version. Contract tests against a sandbox workspace, run on a schedule |
| Quick replies unavailable | Medium | Medium | Validate in spike 2 before committing to a button-driven UX |
| Agent talks over a human | Medium | High | Sticky `handed_over` state, assignment guard, bot inbox, kill switch |
| Dropped webhooks lose customer messages | Medium | High | Reconciliation sweep, do not rely on webhook delivery alone |
| Bad generated answer reaches a customer | Medium | High | Note-only first release, confidence thresholds, always-escalate intents, kill switch. Note that replies cannot be deleted |
| Dead-air latency during inference | High | Medium | Optimistic acknowledgement message, latency ceiling with fallback |
| Subscription suspended by sustained errors | Low | Critical | Unconditional 200 from ingress, never return 410, alert on delivery failures |
| Compliance exposure from undisclosed automation | Medium | High | Explicit disclosure, unambiguous bot naming, human path always available |

---

## 13. Validation plan

Four spikes, strictly ordered. Each has a binary exit criterion.

**Spike 1: prove the round trip.** Sandbox workspace, webhook received and verified, admin reply posted and visible in the Messenger. Establishes that the whole premise holds. Estimated half a day.

**Spike 2: prove or disprove quick-reply buttons.** Attempt `reply_options` against the pinned version. Record the exact error if it fails. Gates the entire UX direction.

**Spike 3: prove human-takeover detection.** Simulate a human replying mid-agent-conversation, including the case where webhooks arrive out of order. The agent must go quiet and stay quiet.

**Spike 4: prove note-only mode.** AI-generated private note attached to a live conversation, useful to a human agent, invisible to the customer. This is the shippable fallback if the customer-facing version is blocked on compliance or quality.

Only after all four should reliability engineering (reconciler, idempotency, rate limiting) begin, since the earlier spikes may change the shape of what needs to be reliable.

---

## 14. Open decisions

1. **Customer-facing replies or agent-assist first?** Agent-assist carries a fraction of the risk and much of the internal value, and uses the same integration. Recommended starting point.
2. **Single workspace or multi-tenant?** Operating inside our own workspace allows a private app with a workspace access token. Integrating into workspaces we do not own requires a public OAuth app with the full install, token refresh, and per-tenant configuration burden. This is a substantially larger build and should be decided before any code is written.
3. **Model hosting and data residency.** Determines what conversation content may leave our infrastructure.
4. **Coexistence policy.** Does the target workspace also run Fin or customer-facing workflows? If so, the stand-down and bot-inbox configuration is part of the design rather than an afterthought.
5. **Channel scope.** Conversations also arrive by email and other channels. Buttons and Messenger-specific affordances do not render there. Decide early whether the agent handles all channels or only the Messenger.

---

## Appendix A: API quick reference

**Base URL:** `https://api.intercom.io`
**Auth:** `Authorization: Bearer <access_token>`
**Version:** `Intercom-Version: <pinned>`

| Operation | Method and path |
|---|---|
| Reply to conversation | `POST /conversations/{id}/reply` |
| Add part (assign, close, open, snooze) | `POST /conversations/{id}/parts` |
| Run assignment rules | `POST /conversations/{id}/run_assignment_rules` |
| Retrieve conversation | `GET /conversations/{id}` |
| Search conversations | `POST /conversations/search` |
| List admins | `GET /admins` |
| Retrieve contact | `GET /contacts/{id}` |

**Reply body, customer-visible:**
```json
{ "message_type": "comment", "type": "admin",
  "admin_id": "493881", "body": "<p>...</p>" }
```

**Reply body, internal note:**
```json
{ "message_type": "note", "type": "admin",
  "admin_id": "493881", "body": "<p>Suggested answer: ...</p>" }
```

**Webhook verification:**
```
X-Hub-Signature: sha1=<hex(HMAC-SHA1(raw_body, client_secret))>
```

## Appendix B: Sources

- Intercom Developer Platform: https://developers.intercom.com/
- Webhook topics: https://developers.intercom.com/docs/references/webhooks/webhook-models
- Webhook notifications, retries and failure handling: https://developers.intercom.com/docs/webhooks/webhook-notifications
- Reply to a conversation: https://developers.intercom.com/docs/references/rest-api/api.intercom.io/conversations/replyconversation
- Admin Reply model: https://developers.intercom.com/docs/references/rest-api/api.intercom.io/models/admin_reply_conversation_request
- Quick Reply Option model: https://developers.intercom.com/docs/references/rest-api/api.intercom.io/models/quick_reply_option
- Rate limiting: https://developers.intercom.com/docs/references/rest-api/errors/rate-limiting
- Fin Agent API: https://developers.intercom.com/docs/guides/fin-agent-api
- Custom Channel Events: https://developers.intercom.com/docs/references/rest-api/api.intercom.io/custom-channel-events
- Data connectors for automation: https://www.intercom.com/help/en/articles/6298285-using-data-connectors-for-automation
- Bot inbox: https://www.intercom.com/help/en/articles/3722087-keep-bot-conversations-in-a-separate-inbox
- Fin in Workflows: https://www.intercom.com/help/en/articles/10032299-use-fin-ai-agent-in-workflows
- Seats: https://www.intercom.com/help/en/articles/8205716-seats
- Real-time messaging and typing indicators: https://www.intercom.com/help/en/articles/258-real-time-messaging-explained
- Community, third-party chatbot integration: https://community.intercom.com/conversations-9/integrating-a-third-party-chatbot-into-intercom-7392
- Community, posting as Operator: https://community.intercom.com/api-webhooks-23/how-can-i-post-a-message-as-operator-1272
- Community, reply_options on quick_reply: https://community.intercom.com/ideas/1511

---

*Findings reflect Intercom's documented behaviour as of August 2026. Items marked unverified require validation against a live workspace before being treated as design constraints.*
