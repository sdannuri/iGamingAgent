# IGamingSupportAgent — Requirements

**Draft v0.3 · 13 August 2026**

---

## In one page

**What we're building.** Software that answers customer support questions for online gambling companies. An operator connects their help desk and their back office; our agent reads a player's actual account and answers their question. When it shouldn't answer, it hands the conversation to a human with everything that person needs to pick up where the agent left off.

**Who buys it.** Multiple gambling operators, each on their own isolated setup, sharing one platform. Not a one-off build for a single company.

**What makes it different from a chatbot.** A chatbot recites the bonus terms. This reads *your* wagering progress and tells you how much further you have to go. That requires a live connection to the operator's systems and a lot of care about who is allowed to see what.

**What the first release will not do.** It will not change anything. It reads and it answers. It cannot issue a bonus, trigger a refund, or unlock an account — those come in the second release, once the first has proved it gets answers right.

**The three rules everything else follows from:**

1. **The agent never decides on its own to look at someone's account.** A written procedure decides that. The AI only reads out the facts it was handed and phrases them well.
2. **Safety checks run before anything else, on every single message.** Problem-gambling signals, abusive messages, and legally sensitive topics are caught by a separate layer that can stop the agent before it does anything.
3. **If we can't do something safely, a human gets it.** Never a guess, never silence.

---

## How to read this

| If you are… | Read |
|---|---|
| **New to the project** | This page, then Part 1 |
| **A CS or CX lead** | Part 2 sections 2.2–2.7 (what the agent does in conversations) and Part 5 (what success looks like) |
| **Compliance or RG** | Part 2 section 2.8 (responsible gambling), Part 3 sections 3.4–3.6 (security, data, audit) |
| **An engineer** | All of Parts 2 and 3, then `Design.md` |
| **Deciding what to build first** | Part 4 (phasing) |
| **Unfamiliar with a term** | The glossary at the end |

**Conventions used here**

- **MUST** — the release it's assigned to doesn't ship without it, unless someone explicitly decides otherwise.
- **SHOULD** — a strong default, but tradeable.
- Every requirement has a permanent ID (`FR-12`, `NFR-3`). **IDs never change**, even if we move a requirement to a different release. `Design.md` refers back to these.
- **V1 / V2 / V3** are the three planned releases. Part 4 says which requirements land in each.

---

# Part 1 — Context

## 1.1 Decisions already made

| Question | Answer | What follows from it |
|---|---|---|
| One customer or many? | **Many — a shared platform** | Every operator's data must be walled off from every other operator's. Each needs their own settings, their own credentials, and a way to set themselves up. This is a much bigger build than a one-off. |
| Can the agent change things? | **No, not in V1. It reads and answers.** | Lower risk, faster to ship. But it also means V1 can't claim the automation rates competitors advertise. |
| Which chat channels? | **Not decided yet** | Requirements below are written so the answer can slot in later without rework. |

## 1.2 The three rules, explained

### Rule 1 — The procedure decides what happens; the AI decides what it says

Support scenarios are written down as step-by-step **procedures** — readable documents that a CS lead or compliance officer can write and review, not code. The procedure says which account details to look up and in what order. The AI's job is narrower than people expect: it works out what the player is asking, and it turns the facts the procedure fetched into a good sentence.

**Why this matters:** it means we can *prove* the agent checked someone's identity before telling them their balance, rather than promising that we asked it nicely. That proof is what a regulator wants to see.

### Rule 2 — Safety checks sit outside the procedures

Screening for problem gambling, distress, abuse, and legally sensitive topics happens on every incoming message, *before* the agent picks a procedure. It has the power to stop everything else.

**Why this matters:** if screening were just another step inside a procedure, then any procedure someone forgot to add it to would silently skip it. We're required to screen 100% of messages, and the only way that holds is if screening can't be skipped by accident. A second check runs on the way out, before any message reaches a player.

### Rule 3 — Connections to operator systems are a checklist, not a promise

Every operator's back office is different. Some can tell us a player's bonus wagering progress; some can't. So each type of lookup is defined as a **capability** — a specific question we can ask an operator's systems. During setup we check which capabilities that operator actually supports. Procedures that need a missing capability are switched off for that operator, and those questions go to humans instead.

**Why this matters:** we can start building without knowing what any given operator's systems can do, and a thin back office means a smaller agent rather than a broken one.

## 1.3 What "read-only" means for the first release

Our main competitor's pitch is *"chatbots answer questions, AI agents take action."* By that framing, a read-only V1 is on the wrong side of the line.

That's an acceptable trade **only if** V1 wins on the things it can win on: accuracy, depth of account knowledge, compliance posture, and clean handoffs to humans. Two consequences run through this whole document:

- **The action layer is designed now and switched on in V2.** It is not a retrofit. See `FR-3` and `FR-4`.
- **V1 cannot honestly claim 80–90% automation.** A read-only agent can only fully handle the questions that are purely informational. Part 5 sets realistic targets.

---

# Part 2 — What the system does

*The complete list, regardless of which release it lands in. Part 4 assigns releases.*

## 2.1 The procedure engine

This is the core of the product. A procedure is a written document describing how to handle one kind of question — readable and editable by the people who own the process today, not just engineers.

| ID | Requirement | |
|---|---|---|
| **FR-1** | Procedures are documents, not code. Non-engineers can write and review them. They are versioned, and changes can be compared side by side. | MUST |
| **FR-2** | Each procedure states: what triggers it, what must be true before it starts, its steps in order, what it is forbidden from doing, when to escalate, what must be disclosed to the player, what player data it may touch, and which operator-system capabilities it needs. | MUST |
| **FR-3** | If a starting condition isn't met, the procedure routes to a defined fallback. The agent never carries on with a guess. | MUST |
| **FR-4** | The full list of possible step types — including the action steps that change things — exists from the first release. Whether each type can actually run is controlled by settings. | MUST |
| **FR-5** | A procedure that uses a step type that's switched off fails when it's written, with a clear message. It never fails live in front of a player. | MUST |
| **FR-6** | Every step that runs is recorded: what went in, what came out, how long it took, what was decided, and which version of the procedure was in force. | MUST |
| **FR-7** | **Rehearsal mode.** A procedure can be run against real conversations and record what it *would* have said, without sending anything. | MUST |
| **FR-8** | Each operator has their own procedures, starting from a shared library they can adopt and adjust. | MUST |
| **FR-9** | A procedure handles one question, not one conversation. Players ask two things at once ("where's my deposit — and why can't I withdraw?"), so a conversation can have several procedures running. | MUST |
| **FR-10** | Procedures state which operator-system capabilities they need. If an operator can't supply one, that procedure is switched off for them and those questions go to humans. | MUST |
| **FR-11** | Operators can write new procedures without involving our engineers. | MUST |

**FR-12** — These step types MUST exist:

| Group | Steps | Plain meaning |
|---|---|---|
| **Identity** | `verify_identity` | Confirm this really is the account holder |
| **Look things up** | `read_player_state`, `read_transactions`, `read_kyc_status`, `read_bonus_state`, `read_limits`, `read_knowledge` | Read account status, payments, document checks, bonuses, limits, and the operator's own written content |
| **Decide and speak** | `classify`, `branch`, `wait`, `respond`, `escalate`, `log` | Work out what's being asked, take a different path depending on the answer, reply, hand to a human, keep a record |
| **Change things** (V2) | `issue_bonus`, `trigger_refund`, `resend_verification`, `update_ticket`, `apply_limit`, `reset_credential`, `close_account` | Actions that alter something. Defined now, switched off until V2. |

## 2.2 The starting set of procedures

**FR-9 continued — FR-13** — These procedures MUST ship in the shared library. All of them only read.

| Procedure | The player asks | What the agent does |
|---|---|---|
| **Where is my deposit** | "My deposit hasn't arrived" | Check identity → look up recent deposits → cross-check the balance → explain the status and when it'll land → if it failed or is missing, hand to the payments team with the full picture |
| **Withdrawal status** | "Where's my withdrawal?" | Look up the withdrawal, any pending checks, expected timing → explain → escalate if it's stuck |
| **Verification / KYC** | "Why isn't my account verified?" | Look up document status → explain exactly which documents are outstanding and why → hand over for document review |
| **Bonus status** | "Where's my bonus? Why can't I withdraw?" | Look up the bonus and wagering progress → explain the terms against *their* actual progress → escalate to issue anything |
| **Can't log in** | "I can't get into my account" | Look up account status → explain the type of problem → route to the right team |
| **Limits and controls** | "How do I set a deposit limit?" | Look up current limits → explain → point to the operator's own tools |
| **Responsible gambling** | Any sign of gambling harm | Pre-approved wording only, then straight to a trained human. Never a generated sentence. See §2.8 |
| **Complaint or dispute** | Anything formal or contested | Straight to a human. Nothing generated goes to the player. See §2.8 |
| **General questions** | Rules, payment methods, how a game works | Answer from the operator's own published content. No account lookup needed |

**FR-14** — *Where is my deposit* is the **benchmark we build against**. It touches identity, payment data, cross-checking two systems, and escalation — all in one flow. If that works properly, most of the engine works.

## 2.3 Knowing who the player is

| ID | Requirement | |
|---|---|---|
| **FR-15** | Identify the player using the operator's own stable customer ID. Matching on email address alone is not good enough. | MUST |
| **FR-16** | Confirm identity before revealing anything about an account. The standard for "confirmed" is set per country, since the rules differ. | MUST |
| **FR-17** | Detect self-excluded players and players in a cooling-off period the moment we identify them, and route them to a separate protective process — never a normal support flow. | MUST |
| **FR-18** | If the operator's systems are down, hand over to a human and say so honestly. **Never guess. Never invent a status.** | MUST |
| **FR-19** | Handle people who aren't logged in or don't have an account, without revealing anything account-specific. | MUST |

## 2.4 Understanding the message

| ID | Requirement | |
|---|---|---|
| **FR-20** | Every incoming message is assessed before the agent acts: what's being asked, how the person feels, any gambling-harm signals, any abuse, what language, and how confident we are. | MUST |
| **FR-21** | Those assessments run at the same time as each other, not one after another, to stay within the speed budget. | MUST |
| **FR-22** | If confidence is below the operator's threshold, hand to a human rather than answering. | MUST |
| **FR-23** | Spot when someone asks two things at once and handle each properly. | MUST |
| **FR-24** | Support 120+ languages. Work out which one the player is using and reply in it. | MUST |
| **FR-25** | Fixed, approved wording (gambling-harm messages, disclosures, handover messages) is translated by humans for each market. **Never machine-translated on the fly.** | MUST |

## 2.5 Emotion and when to hand over

Whether someone is upset changes how the agent should behave — but not in the obvious way.

**In gambling support, mild frustration is normal, not exceptional.** People contact support because money is missing. A simple rule of "if annoyed, get a human" would send most of the workload to humans and destroy the point of the product. It would also often make things worse: for many questions, the fastest way to calm someone down is an accurate answer in three seconds, not a twenty-minute queue for a person who will read the same screen and say the same thing.

**FR-26** — Assess emotional state on every incoming message, before acting.

**FR-27** — Respond across five bands, not a yes/no switch:

| What we detect | What the agent does |
|---|---|
| **Distress or gambling harm** | Completely separate path. Approved wording, trained human. **Never the ordinary support queue.** |
| **Abuse or threats** | Immediate handover, flagged, under its own policy |
| **Very frustrated** | Answer only if we're confident and can settle it in one reply. Otherwise hand over |
| **Mildly frustrated** | Answer, but change the tone — lead with the answer, drop the pleasantries, acknowledge briefly |
| **Calm** | Normal |

| ID | Requirement | |
|---|---|---|
| **FR-28** | Gambling-harm detection **overrides** frustration handling. A distressed player must never end up in the ordinary support queue. | MUST |
| **FR-29** | Abuse and threats have their own policy, separate from normal escalation. | MUST |
| **FR-30** | Track whether someone is getting *more* upset over the conversation. A rising trend is a reason to hand over, regardless of the starting point. | MUST |
| **FR-31** | Take account of repeat contacts. A calm third message about the same unresolved deposit matters more than one angry first message. | MUST |
| **FR-32** | Check whether a human is actually available. Handing over at 3am when nobody is on shift produces silence — attempt the answer instead, and be honest about the wait. | MUST |
| **FR-33** | The emotional reading also shapes *how* the agent writes, not just where the conversation goes. Detecting frustration and then replying cheerfully is worse than not detecting it. | MUST |
| **FR-34** | Tune the thresholds per operator **and per market**. Emotion detection is much weaker outside English, and ordinary directness in some countries reads as anger to a model trained on English politeness. | MUST |
| **FR-35** | Monitor how accurate emotion detection is in each language, and raise an alert if one market starts over-escalating. | MUST |

## 2.6 Handing over to a human

| ID | Requirement | |
|---|---|---|
| **FR-36** | **Once a human replies, the agent stops — permanently.** It only resumes if a human explicitly hands it back. | MUST |
| **FR-37** | Every handover carries the full picture: who the player is, what the agent looked up, what it concluded, why it handed over, and the whole conversation. | MUST |
| **FR-38** | If a player asks for a human, they get one immediately. No attempt to talk them out of it. | MUST |
| **FR-39** | Which team receives a handover is configurable per operator, per topic, and per time of day. | MUST |
| **FR-40** | Cap how many times the agent replies in one conversation. At the cap, hand over rather than keep going. | MUST |
| **FR-41** | Track the state of each conversation: status, last message handled, replies sent, topic, confidence, emotional trend, and reason for handover. | MUST |

> **FR-36 is the single most important behaviour in this document.** An agent that pipes up in the middle of a human's conversation is the most damaging thing this system could do.

## 2.7 Conversation mechanics

| ID | Requirement | |
|---|---|---|
| **FR-42** | Wait a moment for people who send three short messages instead of one long one, then answer once. | MUST |
| **FR-43** | Say it's an automated agent in the first message of every conversation, under a name that clearly isn't a person's. | MUST |
| **FR-44** | Clean and sanitise everything before it's sent. | MUST |
| **FR-45** | **Final check before sending.** Confirm approved wording really came from the approved list, that there's no promotional language, and that no other player's details have crept in. | MUST |
| **FR-46** | Treat everything a player writes, and everything we retrieve, as untrusted. It must never be able to change the agent's instructions or take over a procedure. | MUST |

## 2.8 Responsible gambling and compliance

Online gambling is licensed. This section isn't an engineering preference — it's the price of operating.

| ID | Requirement | |
|---|---|---|
| **FR-47** | Screen **every single incoming message** for distress and problem-gambling signals, whatever the message is about. | MUST |
| **FR-48** | That screening is part of the pipeline, not a step inside a procedure — so no procedure can accidentally skip it. | MUST |
| **FR-49** | On any gambling-harm signal: send the operator's pre-approved wording for that market and hand straight to a trained human. **Nothing generated.** | MUST |
| **FR-50** | Keep a per-operator list of **always hand over** topics. The default list: responsible gambling and self-exclusion, closing an account, complaints and disputes, money-laundering and source-of-funds questions, chargebacks, legal threats, safeguarding and welfare concerns, and anything involving money being paid back. | MUST |
| **FR-51** | On those topics, **not one generated word reaches the player.** Fixed acknowledgement wording only. | MUST |
| **FR-52** | Never write anything promotional or that encourages more play. In several markets this is a licence condition, not a matter of tone. | MUST |
| **FR-53** | Behaviour changes by the player's country. Markets differ substantially on bonus wording, gambling-harm intervention, and what must be disclosed. | MUST |

## 2.9 Operator content and knowledge

| ID | Requirement | |
|---|---|---|
| **FR-54** | Keep a searchable copy of each operator's own content: terms and conditions, bonus terms, help centre, payment methods, game rules. | MUST |
| **FR-55** | Answers must be based on that content. The agent must not state terms or policy it can't point to a source for. | MUST |
| **FR-56** | Re-read the content when it changes, and show the operator when it's gone stale. Quoting last month's bonus terms generates complaints. | MUST |
| **FR-57** | Keep a per-market library of approved fixed wording. | MUST |

## 2.10 Taking action (second release)

Written down now, switched on in V2. The engine, the record-keeping, the pre-checks, and the guardrails are identical to reading — only the switch differs.

| ID | Requirement | |
|---|---|---|
| **FR-58** | Carry out actions on operator systems, within the limits the procedure sets. | MUST |
| **FR-59** | Every action is checked beforehand and safe to retry. A retry must never issue a bonus twice or refund twice. | MUST |
| **FR-60** | Set per-operator, per-procedure limits on what the agent may do — money ceilings and rate caps. | MUST |
| **FR-61** | Above a set value, a human approves before anything happens. | MUST |
| **FR-62** | Every action is reversible, or it requires human approval. | MUST |
| **FR-63** | Actions are recorded in their own separate, unalterable log. | MUST |
| **FR-64** | A switch that stops all actions on its own, independently of the switch that stops the agent talking. | MUST |

## 2.11 Helping human agents (second release)

| ID | Requirement | |
|---|---|---|
| **FR-65** | Produce suggested answers that only human agents see — same procedures, same information, no risk to players. | MUST |
| **FR-66** | Measure how often agents accept a suggestion and how much they change it. | SHOULD |

## 2.12 Reaching out first (third release)

| ID | Requirement | |
|---|---|---|
| **FR-67** | Start conversations based on player behaviour: risk of leaving, milestones, stalled verification, failed payments. | MUST |
| **FR-68** | Check marketing permission and the country's rules on inducements before sending anything. This is a materially harder compliance problem than answering questions. | MUST |
| **FR-69** | Cap how often any one player is contacted, across all campaigns. | MUST |
| **FR-70** | Gambling-harm outreach is a completely separate path from commercial outreach, with its own approval and wording. | MUST |
| **FR-71** | **Never send commercial messages to self-excluded, cooling-off, or at-risk players.** | MUST |

## 2.13 Setting up and configuring an operator

| ID | Requirement | |
|---|---|---|
| **FR-72** | Operators set themselves up: connect help desk → connect back office → map player identity → check which capabilities they have → adopt procedures → rehearse → go live. | MUST |
| **FR-73** | The capability check during setup decides which procedures that operator gets (see FR-10). | MUST |
| **FR-74** | Configurable per operator, at minimum: confidence thresholds, emotion thresholds, always-hand-over topics, reply cap, waiting time, speed limit, disclosure wording, gambling-harm wording, languages, country mapping, opening hours, handover destinations, action permissions, and the stop switches. | MUST |
| **FR-75** | Store each operator's system credentials separately, and let them be revoked individually. | MUST |
| **FR-76** | **Stop switches** — one per operator and one global — that immediately stop everything the agent says to players, while still receiving messages, assessing them, and keeping records. | MUST |
| **FR-77** | Every procedure change is rehearsed before it goes live. | MUST |

## 2.14 Connecting to chat channels

Which channels we support is still open. These hold whatever we choose.

| ID | Requirement | |
|---|---|---|
| **FR-78** | A translation layer converts each channel's messages into one common format, and formats replies for each channel. | MUST |
| **FR-79** | Accept incoming messages quickly and do the thinking separately. The agent takes longer than most channels will wait. | MUST |
| **FR-80** | Don't assume messages always arrive. Check periodically against the source and pick up anything missed. | MUST |
| **FR-81** | Treat an incoming notification as "something happened, go look" rather than trusting its contents. Messages don't arrive in order. | MUST |
| **FR-82** | Never send the same reply twice. A duplicate message to a player is worse than a slow one. | MUST |
| **FR-83** | Assume a sent message can't be edited or deleted. A wrong answer about someone's withdrawal is permanent and public. | MUST |
| **FR-84** | Limit how hard we query each operator's systems. We must not be the reason an operator's back office falls over. | MUST |

## 2.15 Voice (third release)

| ID | Requirement | |
|---|---|---|
| **FR-85** | Support voice as another channel, using the same procedures and the same safety checks. | MUST |

## 2.16 Reporting

| ID | Requirement | |
|---|---|---|
| **FR-86** | A dashboard per operator: volume, how much was automated, how much was handed over and why, satisfaction scores, response and resolution times, confidence spread, most common topics. | MUST |
| **FR-87** | Write the outcome of each conversation back into the operator's own help desk, so they can analyse it in the tools they already use. | MUST |
| **FR-88** | Report the financial case: contacts handled, agent hours saved, cost per contact. The person signing the cheque will ask. | MUST |
| **FR-89** | A review queue where humans check a sample of conversations, and their corrections feed back into the procedures. | MUST |
| **FR-90** | Alert on: confidence drifting, handovers spiking, unusual gambling-harm patterns, emotion detection drifting, operator systems failing, messages failing to send. | MUST |
| **FR-91** | Report emotion readings and handover outcomes per market, so FR-34's thresholds can be tuned against real data. | MUST |
| **FR-92** | Export insights on recurring problems for the operator's product team. | SHOULD |

---

# Part 3 — How well it must do it

## 3.1 Speed

| ID | What | Target |
|---|---|---|
| **NFR-1** | Player sees *something* | Under 3 seconds, 95% of the time |
| **NFR-2** | Player has the actual answer | Under 8 seconds, 95% of the time |
| **NFR-3** | Absolute ceiling before giving up and handing over | 15 seconds |
| **NFR-4** | Safety checks complete | Under 0.4 seconds |
| **NFR-5** | On channels with no "typing…" indicator, send an immediate acknowledgement | Within NFR-1 |
| **NFR-6** | Stop switch takes effect | Under 10 seconds |

## 3.2 Volume

| ID | What | Target |
|---|---|---|
| **NFR-7** | Sustained load per operator | 100,000 conversations a month |
| **NFR-8** | Sudden spike handled without slowing down | 10× normal |
| **NFR-9** | One operator's spike must not slow another's | No knock-on effect |

## 3.3 Staying up

| ID | What | Target |
|---|---|---|
| **NFR-10** | Message intake available | 99.9% |
| **NFR-11** | **When something breaks, conversations go to humans — never to silence** | Always |
| **NFR-12** | No player message lost, even during a partial outage | Zero |

## 3.4 Security and keeping operators apart

| ID | Requirement | |
|---|---|---|
| **NFR-13** | Operators are separated at the database level. No possible query can return one operator's player data to another. | MUST |
| **NFR-14** | Operator credentials are encrypted and never written to logs. | MUST |
| **NFR-15** | We ask for the narrowest possible access — read-only until the action release. | MUST |
| **NFR-16** | Personal details are masked before anything reaches an AI provider. | MUST |
| **NFR-17** | Secure development process, with penetration testing before the first live operator. | MUST |

> **NFR-13 is the highest-severity requirement here.** One operator seeing another operator's players is the failure that ends the company.

## 3.5 Data protection and where data lives

| ID | Requirement | |
|---|---|---|
| **NFR-18** | The AI provider keeps nothing and trains on nothing. | MUST |
| **NFR-19** | Which country data is stored in is configurable per operator and per market. | MUST |
| **NFR-20** | How long data is kept is configurable per operator and per market. | MUST |
| **NFR-21** | Support "delete everything about me" requests, across conversations and records. | MUST |

## 3.6 Proving what happened

| ID | Requirement | |
|---|---|---|
| **NFR-22** | Any interaction can be reconstructed completely: what was looked up, what was decided, what was said, under which version of which procedure and which settings. | MUST |
| **NFR-23** | Records can't be altered, and tampering would be detectable. | MUST |
| **NFR-24** | Records can be exported in a form an operator can hand to a regulator. | MUST |
| **NFR-25** | SOC 2 Type II certification. Enterprise deals will require it. | MUST |

## 3.7 Running the service

| ID | What | Target |
|---|---|---|
| **NFR-26** | A procedure change reaches production (after rehearsal) | Under 1 hour |
| **NFR-27** | New operator from signing to first live conversation | Days, not weeks |
| **NFR-28** | Our own team can see what's happening per operator and respond to incidents | Continuously |

## 3.8 Ease of authoring

| ID | What | Target |
|---|---|---|
| **NFR-29** | A CS lead or compliance officer can write and rehearse a new procedure without an engineer | Under a day, unaided |

> **NFR-29 is what turns the procedure engine into an asset rather than a config file.** If writing a procedure needs an engineer, we've built a slower version of hard-coded rules.

---

# Part 4 — What ships when

```
V1  Read and answer      →  V2  Act and assist      →  V3  Reach out and talk
    66 new requirements       10 new requirements        6 new requirements
    35–50% automated          70–85% automated           + prevented contacts
    Wins on: accuracy         Wins on: parity            Wins on: retention
    Risk: wrong answer        Risk: wrong transaction    Risk: wrong recipient
```

## 4.1 V1 — Read and answer

**Goal:** a compliant, accurate agent that knows the player's actual account, answers what it safely can, and hands over everything else cleanly.

| Area | Requirements |
|---|---|
| Procedure engine | FR-1 – FR-12 |
| Starting procedures | FR-13, FR-14 |
| Identity | FR-15 – FR-19 |
| Understanding messages | FR-20 – FR-25 |
| Emotion and routing | FR-26 – FR-35 |
| Handover | FR-36 – FR-41 |
| Conversation mechanics | FR-42 – FR-46 |
| Responsible gambling | FR-47 – FR-53 |
| Operator content | FR-54 – FR-57 |
| Setup and configuration | FR-72 – FR-77 |
| Channels | FR-78 – FR-84 |
| Reporting | FR-86 – FR-91 |
| Everything in Part 3 | NFR-1 – NFR-24, NFR-26 – NFR-29 |

**Included even though nothing uses it yet:** the full step list including actions (FR-4, FR-12), and read-only credentials (NFR-15). Both exist so V2 is a settings change, not a rebuild.

**Deliberately left out:** all actions, proactive outreach, voice, agent suggestions. SOC 2 certification runs alongside and won't have finished.

## 4.2 V2 — Act and assist

**Goal:** cross from answering to doing. This is where the automation rate moves and where we match what the category claims.

| Area | Requirements |
|---|---|
| Taking action | FR-58 – FR-64 |
| Helping human agents | FR-65, FR-66 |
| Insight export | FR-92 |
| SOC 2 certified | NFR-25 |
| Credentials extended to include actions | NFR-15 |

**V2 doesn't start on a date. It starts when V1 has proved:**

- Wrong answers about someone's account stay below 0.5%
- Zero occurrences of generated text on a forbidden topic
- Zero occurrences of the agent talking over a human
- The audit trail has survived a real compliance review

Actions are the point where a wrong answer becomes a wrong transaction. The accuracy bar is the gate, not the calendar.

## 4.3 V3 — Reach out and talk

**Goal:** stop waiting for problems. Support becomes a way to keep players rather than a cost.

| Area | Requirements |
|---|---|
| Proactive outreach | FR-67 – FR-71 |
| Voice | FR-85 |

Reaching out first brings a compliance problem the first two releases don't have: uninvited contact is governed by marketing permission and each country's rules on encouraging play, and messaging a self-excluded player is categorically worse than answering someone badly. **FR-71 is the requirement to be most careful with in this document.**

---

# Part 5 — What success looks like

## 5.1 V1

Set against a read-only agent. **These are not the numbers competitors advertise, and must not be presented as if they were.**

| Measure | Target | Note |
|---|---|---|
| Handled with no human involvement | 35–50% | Limited by how many questions are purely informational |
| Human closed it, but the agent did the legwork | +25% | Real value even without actions |
| Player satisfaction on agent conversations | 4.5 or better out of 5 | |
| Time to first reply | Under 5 seconds | Against a human baseline measured in minutes |
| Wrong answers about someone's account | Under 0.5% | The number that ends the product if it slips |
| Gambling-harm signals correctly caught | Over 99% | A missed signal is a licence risk |
| Over-handover in any single market | No more than 15% above the average | Catches badly tuned emotion thresholds |
| Generated text on a forbidden topic | **Zero** | Any occurrence is a top-severity incident |
| Agent talking over a human | **Zero** | |
| One operator seeing another's data | **Zero** | |

## 5.2 V2

| Measure | Target |
|---|---|
| Handled with no human involvement | 70–85% |
| Player satisfaction | 4.8 or better |
| Incorrect actions taken | **Zero** |
| Actions needing to be reversed | Under 0.1% |
| Suggestions accepted by human agents | Over 60% |

## 5.3 V3

| Measure | Target |
|---|---|
| Contacts prevented by reaching out first | Measurable drop in matching incoming questions |
| Messages sent to a self-excluded, cooling-off, or at-risk player | **Zero** |
| Marketing permission breaches | **Zero** |

---

# Part 6 — Still to decide

1. **Which chat channels and help desks.** Determines the translation layer and what rich features (buttons, etc.) are available. Next discussion.
2. **How different operator back offices are.** How many platforms must we read from, and can they actually tell us about payments, verification, and bonuses? **This is the biggest unknown.** FR-10 and FR-73 are designed to survive a bad answer, but the value of the starting procedure library depends on it.
3. **Where the AI runs and where data lives.** Whether player conversations may leave our infrastructure, and to which country. Must be settled before any prototype.
4. **What counts as proving identity**, per country (FR-16). Getting this wrong is a data-protection incident.
5. **How procedures are written** — a text format, a point-and-click builder, or structured plain English. Decides whether NFR-29 is real or aspirational. It's also the demo that wins deals.
6. **How operators are separated** — logical separation within one system, or separate regional deployments. Data-residency rules may force the latter anyway, which makes stronger separation cheaper than it first looks.
7. **First customer** — which operator, which markets, what volume. A single-market first customer is a materially easier V1.
8. **Pricing** — per resolved conversation, per seat, or per volume. Changes what we must measure from day one.

---

# Part 7 — What the competition claims

From a competitor's published sales guide (`Competitor.pdf`, 7 pages). It's marketing, so every number is self-reported and unaudited. Treat it as **what buyers will expect**, not as proven engineering.

| Their claim | Page | Where we stand |
|---|---|---|
| 80–90% automated; 91% in a case study | 3, 5 | Not reachable in V1 with read-only. Targeted in V2. |
| Satisfaction 4.8+ | 3, 5 | Match in V1. |
| Written, human-readable procedures | 4 | **Match, and go further.** This is the heart of our design (§2.1). |
| Checks before starting; forbidden actions | 4 | Match in V1 (FR-2, FR-3). |
| Gambling-harm monitoring on 100% of conversations | 4, 6 | **Match in V1. Non-negotiable** (FR-47). |
| SOC 2, data masking, nothing retained or trained on | 4 | Posture in V1 (NFR-16, NFR-18); certified in V2 (NFR-25). |
| 120+ languages | 6 | Match in V1 (FR-24). Their case study covers UK, Germany, France, Netherlands, Australia, Italy, Spain. |
| Complete audit logging | 4 | Match in V1 (NFR-22 – NFR-24). |
| Connects to back offices, help desks, CRM, chat | 4, 5 | V1, behind the translation layer (FR-78). |
| Proactive outreach and retention | 6 | V3 (FR-67 – FR-71). |
| Support insights for product teams | 6 | V2 (FR-92). |

**Their worked example** (page 4) — reproduced because it's our benchmark (FR-14): *"Where Is My Deposit?"* — check identity → query the payment provider → cross-reference the CRM and back-office balance → work out the status → explain the timing, start a refund, or escalate to payments with the context → log it and update the ticket.

**The gap we're aiming at:** they compete on taking action and on compliance guardrails. Their guide says nothing about how accurate the answers are, how they're grounded in real content, how emotion is handled, or how a procedure is validated before it goes live.

---

# Glossary

| Term | Meaning |
|---|---|
| **Back office** | The operator's internal system holding player accounts, balances, payments, and bonuses |
| **Capability** | One specific question we can ask an operator's systems (e.g. "list recent deposits"). Operators support different sets |
| **Cooling-off** | A voluntary short break from gambling that a player has set |
| **Escalate / hand over** | Give the conversation to a human, with context |
| **KYC** | "Know Your Customer" — the identity and document checks gambling operators are legally required to run |
| **AML** | Anti-money-laundering checks, including questions about where a player's money came from |
| **Inducement** | Wording that encourages someone to gamble more. Regulated, and in some markets restricted or banned |
| **Operator** | A gambling company — our customer |
| **Player** | The operator's customer — the person in the conversation |
| **Procedure** | A written, versioned document describing how the agent handles one kind of question |
| **Rehearsal (dry run)** | Running a procedure against real conversations and recording what it *would* have said, without sending |
| **RG** | Responsible gambling — protecting players from gambling harm. A legal duty, not a courtesy |
| **Self-exclusion** | A player formally barring themselves from gambling, often across a whole market |
| **Wagering progress** | How much of a bonus's play-through requirement a player has completed before they can withdraw |
| **V1 / V2 / V3** | The first, second, and third planned releases |
| **MUST / SHOULD** | MUST = doesn't ship without it. SHOULD = strong default, tradeable |
| **FR-n / NFR-n** | Permanent requirement IDs. FR = what it does, NFR = how well. IDs never change |

---

**Source:** `Competitor.pdf` — a competitor's published guide, *"AI Agents for Customer Support in iGaming."* 7 pages. Sales material; all figures self-reported and unaudited. Page references above point to it.

**Companion:** `Design.md` — the technical design, which refers back to the requirement IDs used here.
