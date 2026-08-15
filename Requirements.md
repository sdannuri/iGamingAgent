# Autonomous AI Player Support — Functional Requirements

**Version** 1.0 · **Status** Draft for review · **Owner** Product

**Sources.** This document is built from two inputs: `IGamingPRD.pdf` (product requirements — vision, goals, personas, the AI Procedures engine, trust layer, integrations, NFRs) and `Queries.pdf` (the player query taxonomy — six operational categories with real player phrasings and their underlying causes). Where the two disagree, the taxonomy wins on *what players ask* and the PRD wins on *what we build*. Open conflicts are listed in §17 rather than silently resolved.

**How to read this.** Every requirement has a stable ID. Use the ID in tickets, test cases, and change requests — the numbering never gets reused, even if a requirement is dropped. Requirements are grouped by what the player is trying to do, not by which system we call.

Each requirement carries an **action type**, because the type determines how much safety machinery sits behind it:

| Type | Meaning | Risk posture |
|---|---|---|
| **Read** | Looks something up and explains it. Changes nothing. | Low — worst case is a wrong answer, which we can correct |
| **Change** | Alters the player's account, balance, or entitlements | High — needs identity verification, limits, and an audit trail |
| **Protective** | Restricts the account only: limits down, exclusions on, access revoked | Low — there is no wrong direction. Always allowed |
| **Explain** | The answer is a policy or a rule, often a "no". No system call needed | Medium — the risk is saying it badly, not doing it wrong |
| **Escalate** | Hands to a human with context assembled | Low — but the handover must be clean |

---

## 1. What we are building

Today's support chatbots retrieve information. A player asks where their money is, and the bot links them to a help article about withdrawal times. The player already read that article. That is why they are in the chat.

We are building something that acts instead of points. When a player asks where their $1,200 withdrawal went, the agent looks at the actual withdrawal, finds the actual reason it is held, and either fixes it or explains precisely what the player must do next. It works through the operator's existing back office, payment gateway, helpdesk, and CRM — no infrastructure replacement, no data migration.

The distinction matters commercially. Deflection tools reduce the number of conversations a human sees. This reduces the number of conversations that need a human at all, which is a different and much larger saving.

Three things constrain everything that follows:

**This is regulated money.** Every action either touches real funds or touches a compliance obligation. An agent that is right 95% of the time and wrong 5% of the time is not 95% good — the 5% is a regulatory finding.

**Player safety outranks everything.** When a player shows signs of gambling harm, commercial objectives stop applying. No retention offer, no upsell, no attempt to keep them playing. This is not a policy we can configure away for a particular operator.

**Operators are all different.** Different back office, different payment providers, different licences, different rules about what a support agent may do. The product has to absorb that variation without a code change per operator.

---

## 2. Goals and how we measure them

| Goal | Measure | Target |
|---|---|---|
| Resolve without a human | Share of inbound conversations closed end to end, no human touch | 80–90% (see §17.1) |
| Players are satisfied | CSAT across all resolved conversations | 4.8 / 5.0 |
| Reduce manual workload | Reduction in support hours spent on tier-1 work | 40% (see §17.1) |
| No waiting | Time to first substantive reply, including at peak | Instant — no queue |
| Support becomes a growth channel | Retention lift from proactive engagement | Directional, per operator |

**On the 80–90% number.** This is the share of *conversations*, not the share of *categories*. Financial and account questions are high volume and highly automatable. Bet settlement disputes and source-of-funds reviews are lower volume and often need a human. The blended number can be high while several categories stay mostly manual. We should report the blend and the per-category breakdown together, or the number will be read as a promise we did not make.

**On CSAT.** Measuring satisfaction only on conversations the agent resolved flatters the number, because the hard ones get escalated out of the sample. We report CSAT on every conversation the agent touched, including escalations.

---

## 3. Who this is for

| Persona | What they want | What goes wrong today | What we give them |
|---|---|---|---|
| **Player** | An answer now — where the money is, why the account is locked, why the bonus is missing | Queue waits, chatbots that link to articles, generic non-answers | 24/7 action, not information. Real account state, in plain language |
| **VIP & Retention Manager** | Keep high-value players engaged; act before they leave | VIPs sit behind tier-1 traffic; disengagement is noticed after churn, not before | Behavioural signals, automatic milestone rewards, proactive re-engagement |
| **Compliance & RG Officer** | Provable adherence; duty of care actually discharged | Model hallucinations, missed distress cues, non-compliant promo claims, unauditable bot actions | Screening on 100% of messages, hard stops, immediate exclusion routing, complete audit trail |
| **Support Operations Lead** | Low response times, controlled cost, insight into what is breaking | Seasonal churn in the team, traffic spikes on big fixtures, disconnected tools | Fast integration, procedures they can author themselves, feedback on recurring friction |

The Player persona covers VIP players too — a VIP asking about a withdrawal is asking the same question as anyone else. The VIP & Retention Manager is the operator-side stakeholder who manages those players, not the player.

---

## 4. What players actually ask

Six categories, in descending order of what they cost us to get wrong.

| # | Category | Share of volume | Dominant emotion | Priority |
|---|---|---|---|---|
| 1 | Financial & transactional | **35–40%** | Frustration, urgency | High |
| 2 | Account verification (KYC / AML) | — | Impatience, friction | Medium-high |
| 3 | Bonuses, promotions & VIP | — | Confusion | Medium |
| 4 | Gameplay, settlement & technical | — | Anger | Medium |
| 5 | Responsible gaming & safety | — | Vulnerability, distress | **Critical** |
| 6 | Account access & security | — | Anxiety | High |

Only category 1 has a volume figure in our sources. We should instrument the rest during the first operator integration rather than guess.

Two observations that shape the build:

**Emotion is a routing signal, not decoration.** The taxonomy records a dominant emotion per category because the same factual answer lands differently depending on the player's state. "Your withdrawal is held pending KYC" is fine for a calm player and inflammatory for one who has been waiting three days and already been told this twice.

**Category 5 is not a category.** It is a condition that can appear inside any of the other five. A player can reveal gambling distress in the middle of a bonus question. Screening therefore runs on every message regardless of what the message appears to be about — see §10.

---

## 5. How the agent works: AI Procedures

### 5.1 The idea

A support scenario is written down as a procedure: what must be true before we start, the steps in order, what we are never allowed to do, and what gets logged at the end. The agent executes the procedure. It does not improvise the sequence.

This is the central design decision of the product, and it is worth being explicit about why.

The obvious alternative is to give a model a set of tools and let it work out what to do. That approach is more flexible and it is how most agent demos are built. It is also unshippable here, because you cannot tell a regulator what the agent will do — only what it did. Every audit becomes an argument about sampling.

With procedures, the control flow is fixed and inspectable before anything runs. The model still does the hard language work: understanding what the player meant, and writing the reply. It does not decide which systems to touch or in what order. That decision was made by a named human when the procedure was approved.

The practical consequence is that we can prove things. "No path through this procedure discloses a balance before identity is verified" is a property we can check by examining the procedure, not by testing a thousand conversations and hoping.

### 5.2 What every procedure contains

| ID | Requirement | Detail | Type |
|---|---|---|---|
| AIP-01 | Preconditions | State that must hold before the procedure may start — identity verified, no active self-exclusion, account not frozen. If a precondition fails, the procedure does not run | Read |
| AIP-02 | Steps | The ordered sequence of lookups, decisions, messages, and actions | — |
| AIP-03 | Forbidden actions | Things this procedure may never do, stated explicitly. Enforced at runtime, not by convention | — |
| AIP-04 | Escalation paths | Named conditions that hand to a human, and which team receives it | Escalate |
| AIP-05 | Logging | What is recorded on completion, failure, and escalation | — |
| AIP-06 | Required capabilities | Which back-office functions the procedure needs. If an operator's stack cannot provide one, the procedure is disabled for that operator rather than half-executed | — |
| AIP-07 | Approval record | Who approved this version and when. Unapproved procedures cannot run in production | — |

### 5.3 Authoring, review and change control

| ID | Requirement | Detail | Type |
|---|---|---|---|
| AIP-10 | Non-engineers can author | A support ops lead or compliance officer can write and edit a procedure without writing code | — |
| AIP-11 | Versioned | Every change creates a new version. Old versions are retained and remain readable | — |
| AIP-12 | Approval gate | A procedure that touches money, entitlements, or account state requires named human approval before it can run in production | — |
| AIP-13 | Pre-approval validation | Before approval, the system checks the procedure for unreachable steps, missing capabilities, disclosure before identity verification, and actions the operator's licence does not permit. Failures block approval | — |
| AIP-14 | Test before live | Procedures can be run against recorded or synthetic conversations and the outcomes inspected, without touching a live account | — |
| AIP-15 | Instant disable | Any procedure can be turned off immediately, per operator, without a deploy | — |
| AIP-16 | Change audit | Who changed what, when, and what the previous version said — retained for the full regulatory retention period | — |
| AIP-17 | Composition | A procedure can call another procedure, so shared sequences (identity check, escalation packaging) are written once | — |
| AIP-18 | Per-operator variants | An operator can adapt a standard procedure to their rules without forking it away from platform updates | — |

### 5.4 Coverage

| ID | Requirement | Detail | Type |
|---|---|---|---|
| AIP-20 | Full taxonomy coverage | Procedures must exist for every sub-category in §6–§11, not only the high-volume ones. A category with no procedure routes to a human by default, never to a guess | — |
| AIP-21 | Unknown intent handling | When the classifier cannot place a message with confidence, the agent asks a clarifying question or escalates. It does not pick the closest match and proceed | Escalate |
| AIP-22 | Multi-intent messages | A message containing two requests ("where's my withdrawal and can you close my account") is decomposed and each part handled, with the protective request taking priority | — |
| AIP-23 | Mid-conversation intent change | If the player changes subject, the agent abandons the current procedure cleanly rather than continuing to execute it | — |

### 5.5 What the model does and does not do

| ID | Requirement | Detail | Type |
|---|---|---|---|
| AIP-30 | Model classifies | The model determines what the player is asking about and how distressed they are | — |
| AIP-31 | Model composes | The model writes the reply, in the player's language and an appropriate tone | — |
| AIP-32 | Model does not choose actions | The model never selects which system to call or which step comes next. That is the procedure's job | — |
| AIP-33 | Model does not invent facts | Every factual claim in a reply — balance, status, date, amount, requirement — traces to a system response. Anything not retrieved is not stated | — |
| AIP-34 | Grounding check | Before a reply is sent, it is checked for claims that do not trace to retrieved data. Ungrounded replies are regenerated or escalated | — |

---

## 6. Financial and transactional

The largest category — 35–40% of volume — and the one where the player is angriest, because it is their money and it is missing. Most of these conversations are not actually complicated. They are conversations where nobody has yet looked at the transaction.

### 6.1 Withdrawal tracking and delays

> *"I requested a $1,200 withdrawal to my Visa card 3 days ago, but it's still showing as 'Pending'. Why haven't I received my cash?"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| PAY-01 | Withdrawal status | Reports the current state of a pending withdrawal — requested, approved, sent, cleared — with the date of each transition | Read |
| PAY-02 | Delay root cause | Identifies why it is held: unmet wagering, missing documents, manual risk review, threshold breach, or normal banking time. Names the actual reason, not a generic one | Read |
| PAY-03 | Resolution steps | States exactly what the player must do to unblock it, with specific figures — "$25 of wagering remaining on slots", not "complete your wagering" | Read |
| PAY-04 | Expected timing | Gives the remaining expected window based on the actual payment method and its current state, not a generic policy range | Read |
| PAY-05 | Cancelled withdrawal | Explains why a withdrawal was cancelled and returned to the account balance, including whether the player, the system, or a risk rule did it | Read |
| PAY-06 | Reverse withdrawal | Explains the reversal setting and, where the licence permits reversal, how to turn it off. Where a jurisdiction prohibits reversal, states that plainly | Explain |
| PAY-07 | Withdrawal limits | Reports the player's applicable daily, weekly and monthly limits by method and account level, and how much of each remains | Read |
| PAY-08 | Method availability | Explains which withdrawal methods are available on this account and why one may be unavailable | Explain |
| PAY-09 | Risk review escalation | When a withdrawal is in manual review, escalates to the risk team with the transaction, the trigger, and the player's history assembled | Escalate |

### 6.2 Deposits and uncredited funds

> *"My bank account shows $150 was debited by your site, but my casino wallet balance still says $0.00!"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| PAY-10 | Deposit lookup | Finds the deposit by amount, method, and approximate time when the player cannot give a reference | Read |
| PAY-11 | Status reconciliation | Compares what the payment provider says against what the account balance shows, and reports the discrepancy rather than either side alone | Read |
| PAY-12 | Completed but not credited | Where the provider confirms success but the balance does not reflect it, escalates to payments with both records attached. Never tells the player to wait when the money has demonstrably arrived | Escalate |
| PAY-13 | Pending deposits | Explains the clearing window for the specific method and the current network or provider state | Read |
| PAY-14 | Failed deposits | Explains why it failed and what to do differently. Where a refund is due, confirms whether it has been issued and when it should land | Read |
| PAY-15 | Declined card errors | Translates the provider's error code into a plain-language cause and the player's next step | Explain |
| PAY-16 | Bank-side blocks | Where the decline came from the player's own bank — including gambling-transaction blocks — says so, since nothing we do will fix it | Explain |
| PAY-17 | Crypto confirmations | Reports required confirmation count, confirmations received, and the transaction's state on chain | Read |
| PAY-18 | Crypto mismatches | Handles wrong network, wrong asset, underpayment, and stuck transactions, escalating where manual recovery is needed | Escalate |
| PAY-19 | Deposit limits | Reports remaining deposit headroom against any limits in force, including self-imposed ones | Read |

### 6.3 Payment methods, currency and rules

> *"I deposited using my MasterCard, but the system won't let me select MasterCard to withdraw. Why?"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| PAY-20 | Closed-loop rule | Explains why funds must return to the instrument they came from, and which of the player's methods are eligible for withdrawal | Explain |
| PAY-21 | Third-party payments | Explains that an account belonging to someone else — spouse, family, friend — cannot be used, and why this is an AML requirement rather than a preference | Explain |
| PAY-22 | Currency conversion | Explains an FX charge on a specific transaction, showing rate and amount, and who levied it | Explain |
| PAY-23 | Fee queries | Explains any fee applied to a specific transaction and its source | Read |
| PAY-24 | Method management | Explains how to add, remove or verify a payment method and what verification each requires | Explain |
| PAY-25 | Chargeback contact | Recognises a message that indicates a chargeback or bank dispute and routes it immediately to the payments and risk teams without attempting to resolve it | Escalate |

---

## 7. Account verification — KYC and AML

The single largest source of friction in a player's life, and the point where most withdrawal complaints actually originate. The requirement here is not to speed up verification. It is to make its state legible, so the player stops guessing.

One constraint runs through this whole section: where an account is under an anti-money-laundering investigation, the agent must not reveal that an investigation exists or why. Disclosing it is a criminal offence in several licensed markets. The agent gives the player the neutral, approved wording and escalates. This is a platform-level rule, not an operator setting.

### 7.1 Document status and rejections

> *"My utility bill was rejected because it didn't show my full name. Can I submit a PDF of my internet bill instead?"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| KYC-01 | Verification status | Reports the player's overall verification state and which specific checks are outstanding | Read |
| KYC-02 | Document status | Reports the state of each submitted document — received, in review, approved, rejected — with dates | Read |
| KYC-03 | Rejection reason | Gives the actual reason a document was rejected: glare, cropped edges, expired, name mismatch, address mismatch | Read |
| KYC-04 | How to fix it | Tells the player exactly what to submit instead, with the requirements for that document type | Explain |
| KYC-05 | Acceptable documents | Lists what counts as valid proof of identity, address, and payment method for this player's jurisdiction | Explain |
| KYC-06 | Expected timing | Gives realistic review timing based on the current queue, not a marketing figure | Read |
| KYC-07 | Upload help | Walks the player through the upload process and diagnoses common failures — file size, format, portal errors | Explain |
| KYC-08 | Rejection loop | Detects a player rejected repeatedly on the same document and escalates to a human rather than sending them round again | Escalate |
| KYC-09 | Verification effects | Explains what verification unlocks — withdrawal eligibility, higher limits, full account access | Explain |

### 7.2 Enhanced due diligence and source of funds

> *"You locked my account and are asking for my bank statements and payslips. Why do you need my private financial records just for me to play?"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| KYC-10 | Explain the request | Explains that source-of-funds checks are a regulatory obligation triggered by account activity, using approved wording that does not imply suspicion of the player | Explain |
| KYC-11 | Document guidance | Explains what evidence satisfies the check, including for self-employed players, pensioners, and players with non-salary income | Explain |
| KYC-12 | Submission status | Reports what has been received and what is still outstanding | Read |
| KYC-13 | No investigation disclosure | Never confirms, denies, or hints at an AML investigation, its trigger, or its progress. Uses approved neutral wording and escalates | Escalate |
| KYC-14 | Account restriction status | Explains what the player can and cannot currently do while the check is open | Read |
| KYC-15 | Route to compliance | Hands source-of-funds conversations to the compliance team with the trigger and submission history assembled | Escalate |

### 7.3 Profile and identity corrections

> *"I accidentally misspelled my surname when signing up. How can I correct it to match my passport?"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| KYC-20 | Contact detail updates | Updates email or phone after identity re-verification, and notifies the previous address that a change occurred | Change |
| KYC-21 | Address change | Explains the process and evidence needed to change a registered address, and takes the submission | Explain |
| KYC-22 | Name corrections | Never changes a registered name directly. Explains the evidence needed and routes to the verification team | Escalate |
| KYC-23 | Date of birth | Never changes a date of birth under any circumstance. Routes to compliance, since a DOB change can indicate underage registration or account takeover | Escalate |
| KYC-24 | Change during withdrawal | Refuses profile changes while a withdrawal is pending, explains why, and offers to proceed once it settles | Explain |
| KYC-25 | Marketing preferences | Updates marketing contact preferences directly, with immediate effect | Change |

---

## 8. Bonuses, promotions and VIP

Confusion rather than anger. These are usually arithmetic questions the player has no way to do themselves, because the numbers are inside our systems.

### 8.1 Missing and uncredited promotions

> *"I deposited $100 using code WELCOME100, but I only got my cash deposit and no bonus money!"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| BON-01 | Eligibility check | Checks whether the player qualified for the promotion and, if not, states the specific condition that failed | Read |
| BON-02 | Missing code | Determines whether a promo code was applied at deposit, and explains the outcome if it was not | Read |
| BON-03 | Threshold shortfall | Where the deposit fell below the minimum, states the actual amount deposited and the required minimum | Read |
| BON-04 | Distribution delay | Distinguishes a batch-processing delay from a genuine failure and gives the expected crediting time | Read |
| BON-05 | Free spins | Reports whether spins were credited, on which game, how many remain, and when they expire | Read |
| BON-06 | Cashback | Explains the calculation basis, the qualifying period, and when it pays out | Read |
| BON-07 | Opt-in status | Confirms whether the player opted into a campaign and when | Read |
| BON-08 | Manual credit | Credits an owed bonus that failed to apply, within pre-approved value ceilings and once per qualifying event | Change |
| BON-09 | Above-ceiling escalation | Escalates to the promotions team where the amount exceeds the automatic ceiling or eligibility is genuinely ambiguous | Escalate |

### 8.2 Wagering and playthrough

> *"I spent $200 playing Blackjack — why did my bonus wagering requirement only go down by $20?"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| BON-10 | Remaining wagering | States the exact amount of wagering left, in currency, not as a multiplier | Read |
| BON-11 | Game weighting | Explains how much each game type contributes — slots typically 100%, live blackjack often 10% — and shows the effect on the player's actual play | Explain |
| BON-12 | Calculation basis | States whether wagering is calculated on the bonus alone or on deposit plus bonus, since the two differ by a factor of two and this is the most common cause of dispute | Explain |
| BON-13 | Contribution breakdown | Shows what the player's recent play contributed toward the requirement, by game | Read |
| BON-14 | Expiry | States when the active bonus expires and what happens to the bonus balance and any winnings at expiry | Read |
| BON-15 | Forfeiture | Explains what the player loses by forfeiting, and processes the forfeit on explicit confirmation | Change |
| BON-16 | Max bet breach | Where a bonus was voided for exceeding the maximum permitted bet, identifies the specific breaching bet and explains the rule | Read |
| BON-17 | Locked withdrawal | Where withdrawal is blocked by active wagering, connects the two clearly — this is the most common hidden cause of a withdrawal complaint | Read |
| BON-18 | Excluded games | Lists games that do not contribute or are barred while a bonus is active | Explain |

### 8.3 VIP tiers and loyalty

> *"Can I exchange my 5,000 reward points for cash instead of bonus credits?"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| VIP-01 | Points balance | Reports current loyalty points and their cash or bonus equivalent | Read |
| VIP-02 | Tier status | Reports current tier, points to the next tier, and the qualifying period | Read |
| VIP-03 | Tier benefits | Explains what each tier provides — limits, cashback rate, dedicated manager, event access | Explain |
| VIP-04 | Redemption | Explains redemption options and processes an in-policy redemption | Change |
| VIP-05 | Tier loss | Explains why a tier was lost or downgraded and what maintaining it requires | Read |
| VIP-06 | Status matching | Recognises a request to match VIP status from another operator and routes it to the VIP team. Never commits to a tier | Escalate |
| VIP-07 | VIP recognition | Identifies a VIP player at the start of a conversation and applies their servicing rules — priority routing, named contact, higher ceilings | Read |
| VIP-08 | VIP escalation path | Routes VIP escalations to the VIP team rather than general support | Escalate |
| VIP-09 | Bespoke offers | Never invents, promises, or negotiates a bespoke offer. Routes to the VIP team | Escalate |
| VIP-10 | RG override | Suppresses every VIP benefit, offer, and retention action where responsible gaming signals are present. Player safety outranks player value without exception | Protective |

---

## 9. Gameplay, settlement and technical

Anger, and the category where being objectively right matters most, because the player believes they were cheated. Our advantage is that the round logs exist and the player cannot see them.

### 9.1 Sportsbook settlement disputes

> *"My bet on the football match was settled as a loss, but the team won in extra time!"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| SPT-01 | Settlement explanation | Retrieves the bet, the official result, and the settlement rule applied, and explains the outcome against the actual market terms | Read |
| SPT-02 | Regular time vs extra time | Explains that most football markets settle on 90 minutes plus stoppage, which is the single most common settlement dispute | Explain |
| SPT-03 | Void bets | Explains why a bet was voided and returned at odds of 1.00 — non-participant, market error, abandoned event, insufficient playing time | Read |
| SPT-04 | Rule 4 deductions | Explains a deduction applied after a withdrawal from an event, showing the deduction rate and the effect on the return | Explain |
| SPT-05 | Dead heat | Explains dead-heat settlement, where the stake is divided and paid at full odds — routinely mistaken for underpayment | Explain |
| SPT-06 | Postponed events | Explains how postponement and abandonment are handled, and the period after which bets are voided | Explain |
| SPT-07 | Cash out unavailable | Explains why cash out was suspended — price movement, live event suspension, VAR review, market closure | Read |
| SPT-08 | Cash out value | Explains how a cash-out figure was derived and why it moved | Explain |
| SPT-09 | Accumulator settlement | Walks through a multiple leg by leg, showing which legs won, lost, or voided, and how the return was calculated | Read |
| SPT-10 | Palpable error | Explains a bet resettled because of an obviously incorrect price, and escalates where the player disputes it | Escalate |
| SPT-11 | Bet history | Retrieves a specific bet by date, event, or amount when the player cannot give a reference | Read |
| SPT-12 | Genuine settlement error | Where our data indicates the settlement was actually wrong, escalates to the trading team with the bet, the rule, and the official result attached. Never resettles autonomously | Escalate |
| SPT-13 | Result dispute | Where the player disputes the official feed result, escalates rather than repeating our data at them | Escalate |

### 9.2 Casino and live dealer interruptions

> *"The slot machine froze right when 3 Scatter symbols hit for the Bonus Round! Did I lose my bonus feature?"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| CAS-01 | Round state | Retrieves the state of an interrupted round from the game server and reports what actually happened | Read |
| CAS-02 | Incomplete round recovery | Explains how an incomplete round resolves and whether the player can resume it | Read |
| CAS-03 | Bonus feature interruption | Determines whether a triggered feature was recorded before the interruption and what happens to it | Read |
| CAS-04 | Disconnection handling | Explains the game's disconnection protocol — auto-stand, auto-complete, or held open — for the specific game | Explain |
| CAS-05 | Deducted but not played | Where a stake was taken and no round was recorded, escalates for refund with the round reference attached | Escalate |
| CAS-06 | Session expiry | Explains a session timeout and what happened to the in-flight bet | Read |
| CAS-07 | Live dealer disputes | Retrieves the live round record and explains the outcome. Escalates where the player disputes dealer conduct | Escalate |
| CAS-08 | Game history | Retrieves a specific round by game, date, or amount | Read |
| CAS-09 | Fairness questions | Explains RNG certification and independent testing without being defensive, and escalates a formal fairness complaint | Escalate |
| CAS-10 | Provider outage | Recognises a known game provider outage and tells the player plainly rather than investigating their account | Read |

### 9.3 Geolocation and access errors

> *"I'm sitting on my couch in New Jersey, but your app says 'Location Verification Failed'."*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| TEC-01 | Geolocation failure | Explains why a location check failed and the practical fixes — disable VPN, enable location services, switch from Wi-Fi to mobile data | Explain |
| TEC-02 | Border proximity | Explains that players near a licensed-zone boundary may fail checks legitimately, and what to try | Explain |
| TEC-03 | VPN detection | Explains that VPN or proxy use blocks play, and that this is a licence condition rather than our preference | Explain |
| TEC-04 | Error code translation | Translates any player-facing error code into a plain-language cause and next step | Explain |
| TEC-05 | Repeated failures | Escalates a player who keeps failing location checks from a location that should be valid | Escalate |
| TEC-06 | App and browser issues | Diagnoses common client problems — version, cache, permissions, network | Explain |
| TEC-07 | Known incidents | Where a platform incident is in progress, tells the player, gives the expected restoration time, and does not raise a ticket per player | Read |
| TEC-08 | Restricted jurisdiction | Explains clearly when a player is in a market we cannot serve, without implying a workaround exists | Explain |

---

## 10. Responsible gaming and player safety

The critical category, and the one that overrides all others. Safety protocols supersede every business and retention objective — this is stated in our sources and it is not negotiable per operator.

### 10.1 How screening works

Every inbound message is screened for gambling distress before anything else happens. Not after intent classification, not only inside RG procedures, and not only when the conversation looks like it might be about gambling harm. First, always, on everything.

The reason is that distress arrives inside other conversations. A player asking why their bonus was voided may, three messages later, say they cannot stop. If screening only ran on messages classified as RG-related, we would miss exactly the cases that matter most.

| ID | Requirement | Detail | Type |
|---|---|---|---|
| RG-01 | Screen every message | 100% of inbound messages are screened, in every channel and language, including messages on conversations already handed to a human | Protective |
| RG-02 | Two independent layers | A deterministic phrase-based check and a model-based classifier run in parallel. Either one firing triggers the response. They are never required to agree — combining them by agreement would let each suppress the other | Protective |
| RG-03 | Screen before anything else | Screening runs before intent classification, before any account lookup, and before any procedure begins | Protective |
| RG-04 | Fail safe | If the classifier is unavailable, the deterministic layer still runs and the conversation is escalated. A screening outage never results in unscreened conversations | Protective |
| RG-05 | Behavioural signals | Screening also considers account behaviour — rapid deposit escalation, session length, loss chasing, repeated limit-increase requests — not just message text | Protective |
| RG-06 | Conversation-level state | Once RG concern is raised, it persists for the whole conversation and cannot be cleared by a subsequent normal-sounding message | Protective |

### 10.2 Hard stop

| ID | Requirement | Detail | Type |
|---|---|---|---|
| RG-10 | Freeze generation | On detection, normal generative responses stop instantly. The agent does not continue the previous procedure, whatever it was | Protective |
| RG-11 | Approved wording only | The response comes from compliance-approved wording, not free composition | Protective |
| RG-12 | Suppress commercial content | All offers, bonuses, promotions, upsells and retention messaging are suppressed immediately and for the remainder of the conversation | Protective |
| RG-13 | Immediate escalation | The conversation goes to a human RG specialist at top priority, with the trigger and context attached | Escalate |
| RG-14 | Localised helplines | Presents the correct support resources for the player's jurisdiction — BeGambleAware, GAMSTOP, Spelpaus, and local equivalents | Protective |
| RG-15 | Offer protective actions | Offers cooling-off and self-exclusion directly, without requiring the player to ask twice or navigate elsewhere | Protective |
| RG-16 | Crisis language | Where a message indicates risk to the player's safety rather than gambling harm alone, follows a separate crisis path with emergency resources and immediate human involvement | Escalate |
| RG-17 | No debate | The agent never argues with, minimises, or seeks to verify a player's statement of distress | Protective |

### 10.3 Limits

> *"I want to set a weekly deposit limit of $200 starting right now."*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| RG-20 | Set a limit | Applies deposit, loss, wager, or session limits on request | Protective |
| RG-21 | Decrease immediately | A limit made more restrictive takes effect immediately, with no cooling-off period and no friction | Protective |
| RG-22 | Increase requires waiting | A limit made less restrictive is subject to the statutory waiting period for the jurisdiction. The agent explains this and never expedites it | Explain |
| RG-23 | Never remove early | Never removes or loosens a cooling-off period, limit, or exclusion before its term expires, regardless of how the player asks or who asks on their behalf | Protective |
| RG-24 | Current limits | Reports all limits in force, their values, remaining headroom, and when any pending change takes effect | Read |
| RG-25 | Reality checks | Configures session reminders and time alerts | Protective |
| RG-26 | Limit-increase signal | Treats a request to raise limits as a risk signal to be considered alongside behavioural data, not as a routine settings change | Protective |

### 10.4 Self-exclusion and closure

> *"I want to close my account for 6 months because I need a break from gambling."*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| RG-30 | Cooling-off | Applies a short-term break immediately on request | Protective |
| RG-31 | Self-exclusion | Applies self-exclusion for the requested term immediately. Never delays, never requires a reason, never routes it through a retention conversation first | Protective |
| RG-32 | No retention attempt | Makes no offer and no attempt to talk the player out of it. The request is honoured, not negotiated | Protective |
| RG-33 | Marketing suppression | Suppresses all marketing across every channel immediately on exclusion, not on the next campaign cycle | Protective |
| RG-34 | Multi-brand enforcement | Where the operator runs several brands, applies the exclusion across all of them | Protective |
| RG-35 | National schemes | Explains how to register with the national self-exclusion scheme and routes registration where we can assist | Explain |
| RG-36 | Balance handling | Explains what happens to the remaining balance and any pending withdrawals on exclusion | Read |
| RG-37 | Excluded player contact | Where an excluded player makes contact, does not re-engage them, does not offer reactivation, and routes anything beyond a factual answer to a human | Protective |
| RG-38 | Reactivation | Never reactivates an excluded account. Routes to the RG team, which applies the jurisdiction's process | Escalate |
| RG-39 | Account closure | Distinguishes a request to close from a request to self-exclude, confirms which the player means, and applies the more protective interpretation when it is ambiguous | Protective |
| RG-40 | Confirmation record | Confirms every protective action to the player in writing and records it in the audit trail with a timestamp | Protective |

### 10.5 Age and vulnerability

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| RG-45 | Underage indication | Where anything indicates the account holder may be under age, freezes engagement and escalates to compliance immediately | Escalate |
| RG-46 | Third-party concern | Handles a report from a family member or friend about a player, without disclosing any account information to the reporter | Escalate |
| RG-47 | Vulnerability markers | Recognises indications of financial hardship, bereavement, or cognitive impairment and adjusts handling accordingly | Protective |

---

## 11. Account access and security

Anxiety. A player who thinks they have been hacked needs a fast, competent answer. The tension here is that every helpful action is also the action an attacker wants.

### 11.1 Authentication and recovery

> *"I got a new phone and lost access to my Authenticator app for 2FA. How can I log in?"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| SEC-01 | Password reset | Triggers a password reset through the standard verified flow. Never sets, reveals, or transmits a password | Change |
| SEC-02 | Reset link not arriving | Diagnoses delivery failures — spam filtering, wrong address on file, blocked domain — and resends | Change |
| SEC-03 | Locked after failed attempts | Explains the lockout, its duration, and unlocks after successful identity verification where policy permits | Change |
| SEC-04 | 2FA recovery | Explains the recovery process. Never disables or bypasses two-factor authentication itself — this is the highest-value target for account takeover | Escalate |
| SEC-05 | Account recovery | Where a player has lost access to their registered email, routes to a manual identity-verification process and never accepts a new address on the player's word | Escalate |
| SEC-06 | Identity challenge | Applies the operator's identity challenge before any account-specific disclosure, with a stronger challenge for security-related requests | Read |
| SEC-07 | Failed verification | Where a player cannot verify, discloses nothing and offers the manual recovery route. Never partially confirms, never hints | Protective |

### 11.2 Security alerts and unauthorised access

> *"There's a withdrawal request on my account that I didn't authorize!"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| SEC-10 | Login alert explanation | Explains a login-notification email, including the time and approximate location of the login | Read |
| SEC-11 | Suspected takeover | On any report of unauthorised access, immediately locks the account, terminates all sessions, and escalates to fraud — before investigating anything | Protective |
| SEC-12 | Unauthorised transaction | Halts any pending withdrawal the player disputes and escalates to fraud with the transaction and session history assembled | Protective |
| SEC-13 | Session termination | Ends all active sessions on request | Protective |
| SEC-14 | Login history | Shows recent login activity so the player can identify what was not them | Read |
| SEC-15 | Security hardening | Guides the player through enabling two-factor authentication and strengthening their credentials | Explain |
| SEC-16 | No investigation detail | Explains that a fraud review is under way and what to expect, without disclosing detection methods or investigation specifics | Explain |

### 11.3 Multiple accounts and household rules

> *"My roommate and I both have accounts on your site. Why was my bonus canceled for 'Duplicate IP'?"*

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| SEC-20 | Duplicate account explanation | Explains the one-account-per-person rule and why the platform enforces it | Explain |
| SEC-21 | Household bonus rules | Explains that promotions are typically limited to one per household or IP address, which is why a legitimate second player may be affected | Explain |
| SEC-22 | Genuine household dispute | Where the player says they share an address with another genuine player, gathers the account references and routes to fraud. Never reinstates a voided bonus autonomously | Escalate |
| SEC-23 | Second account request | Explains that a second account is not permitted and does not suggest a route around it | Explain |
| SEC-24 | Duplicate closure | Explains what happens to a duplicate account and any balance on it | Explain |
| SEC-25 | Account sharing | Explains that accounts may not be shared or transferred, and escalates where sharing appears to be happening | Escalate |

---

## 12. Escalation and human handover

An escalation is a product feature, not a failure. Handled well, it is the second-best outcome. Handled badly, it is worse than never having engaged.

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| ESC-01 | Always-escalate floor | A platform-level list of situations that always reach a human — distress, underage indication, suspected takeover, chargeback, formal complaint, legal or regulator contact, media enquiry, threats of harm. Operators may add to this list and may never remove from it | Escalate |
| ESC-02 | Frustration escalation | Escalates a player who is clearly angry or has repeated themselves, before their patience is exhausted rather than after | Escalate |
| ESC-03 | Explicit request | Escalates immediately when the player asks for a human. Never asks them to justify it, never attempts one more time first | Escalate |
| ESC-04 | Failure to resolve | Escalates when the agent cannot resolve the request, rather than looping | Escalate |
| ESC-05 | Low confidence | Escalates when intent confidence is below threshold or retrieved data is contradictory | Escalate |
| ESC-06 | Context package | Hands over the full transcript, the identified intent, everything retrieved, every action taken, and the reason for escalation — so the human never asks the player to start again | Escalate |
| ESC-07 | Team routing | Routes to the right team: payments, risk, compliance, RG, fraud, VIP, trading | Escalate |
| ESC-08 | Priority | Sets handover priority from category and detected distress. RG and safety escalations go top of queue unconditionally | Escalate |
| ESC-09 | Stop on human reply | Once a human replies, the agent stops responding on that conversation and does not resume unless explicitly handed back | Escalate |
| ESC-10 | Set expectations | Tells the player they are being transferred, to whom, and roughly how long it will take | Escalate |
| ESC-11 | Out of hours | Where the receiving team is unavailable, says so honestly and gives a realistic response time. Never implies someone is about to reply | Escalate |
| ESC-12 | Complaint handling | Recognises a formal complaint, records it against the regulatory clock, and routes it to the complaints process rather than treating it as a query | Escalate |
| ESC-13 | Handback | Supports a human returning a conversation to the agent, with the agent aware of what the human already said | — |

---

## 13. Trust, security and audit

The controls that let a regulated operator run this at all. These are enforced by the platform, around every procedure — they are not something a procedure author declares or can opt out of.

### 13.1 Data protection

| ID | Requirement | Detail | Type |
|---|---|---|---|
| TRU-01 | Edge sanitisation | Incoming messages pass through a high-performance scrubbing stage before reaching any model. Card numbers, bank details, national identity numbers, passwords, and one-time codes are removed at the edge | — |
| TRU-02 | Sanitise before tokenisation | Scrubbing happens before the message reaches a model provider, not after — otherwise the data has already left | — |
| TRU-03 | Fail closed | If sanitisation fails or is unavailable, the message is not sent to a model. The conversation degrades to human handling rather than proceeding unscrubbed | Protective |
| TRU-04 | Never echo secrets | Even where a player volunteers a password or card number, the agent never repeats it back and tells the player not to share it | Protective |
| TRU-05 | Minimum necessary data | Each procedure retrieves only the fields it needs. Nothing is fetched speculatively | — |
| TRU-06 | Provenance tagging | Every retrieved fact is tagged with the account it came from at the point of retrieval, and a reply cannot include a fact tagged to a different account. This is the only control that catches another player's data appearing in a response | Protective |
| TRU-07 | No retention, no training | Player dialogue and operator data are never retained by model providers and never used for model training | — |
| TRU-08 | Residency | Operator and player data stays in the required jurisdiction for the operator's licence | — |
| TRU-09 | Encryption | Data encrypted in transit and at rest throughout | — |
| TRU-10 | Erasure | Supports a player's right to erasure while preserving the regulatory audit record, by destroying the key that makes their personal data readable rather than deleting the record | — |
| TRU-11 | Tenant isolation | One operator's data, procedures, and configuration are never reachable from another operator's environment | — |
| TRU-12 | SOC 2 Type II | The platform operates under verified SOC 2 Type II controls | — |

### 13.2 Action safety

| ID | Requirement | Detail | Type |
|---|---|---|---|
| TRU-20 | Verify before disclose | No account-specific information is disclosed before identity is verified. Enforced by the platform, not by each procedure remembering to | Protective |
| TRU-21 | Verify before change | No account change without verified identity, at the strength the change warrants | Protective |
| TRU-22 | Value ceilings | Every automatic action that moves value has a configured ceiling. Above it, a human approves | — |
| TRU-23 | Rate limits | Per-player and per-operator caps on how often an action can be taken automatically, to bound the damage from any single fault | — |
| TRU-24 | Exactly once | Every change executes at most once, even when messages are retried or duplicated. Where the outcome is genuinely uncertain, the action freezes and a human resolves it. We never blind-retry an action that moves money | — |
| TRU-25 | Outcome verification | After a change, the agent confirms it actually took effect before telling the player it did — back offices are not always immediately consistent | — |
| TRU-26 | Reversibility | Every automatic change records what would be needed to reverse it | — |
| TRU-27 | Capability gating | Where an operator's stack cannot support a procedure's required capabilities, the procedure is disabled for that operator rather than partially executed | — |
| TRU-28 | Kill switch | Any action type, procedure, or the whole agent can be stopped instantly, per operator, without a deploy | Protective |

### 13.3 Audit

| ID | Requirement | Detail | Type |
|---|---|---|---|
| TRU-30 | Complete record | Every message, classification, decision, retrieval, action, and escalation is logged | — |
| TRU-31 | Tamper evidence | The audit record is append-only and tamper-evident | — |
| TRU-32 | Decision traceability | For any past reply, we can reconstruct which procedure ran, which version, what data it saw, and why it said what it said | — |
| TRU-33 | Retention | Records retained for the period each jurisdiction requires | — |
| TRU-34 | Regulator export | Audit records exportable in a form a regulator or auditor can work with | — |
| TRU-35 | RG action log | Every RG detection, action, and escalation is separately reportable, since this is the evidence of duty of care | — |
| TRU-36 | Approval trail | Who approved each procedure version, and when, is part of the permanent record | — |

---

## 14. Proactive engagement and retention *(optional scope)*

Marked optional in our source material. It is a genuinely different product mode — the agent starting conversations rather than answering them — and carries different regulatory risk, since an unsolicited message to the wrong player is a marketing compliance issue.

| ID | Requirement | What the agent does | Type |
|---|---|---|---|
| PRO-01 | Churn signals | Identifies disengagement patterns and declining sentiment before the player leaves | Read |
| PRO-02 | Re-engagement | Delivers a personalised re-engagement offer to an at-risk player, within approved campaign rules | Change |
| PRO-03 | Milestone recognition | Recognises first deposit, tier upgrade, anniversary and similar events in real time | Read |
| PRO-04 | Milestone reward | Credits the milestone reward and confirms it to the player | Change |
| PRO-05 | RG suppression | Suppresses every proactive message where any RG signal, limit, or exclusion is present. This gate is checked at send time, not at campaign build time | Protective |
| PRO-06 | Marketing consent | Respects marketing preferences and channel consent on every outbound message | Protective |
| PRO-07 | Frequency capping | Caps how often a player receives proactive contact | Protective |
| PRO-08 | Product feedback loop | Aggregates support interactions into recurring friction themes for product and design teams | Read |
| PRO-09 | Attribution | Measures the retention and revenue effect of proactive actions, so the mode can be justified or stopped | Read |

---

## 15. Integrations

The agent sits in the middle of the operator's existing stack and executes across it. No infrastructure replacement.

| ID | System | What we need from it | Examples named in source |
|---|---|---|---|
| INT-01 | Player account management / back office | Player state, balance, transaction history, KYC status, limits, exclusions, bonus state | Softswiss, EveryMatrix, White Hat Gaming, custom platforms |
| INT-02 | Payment providers | Transaction status, settlement state, refunds | Operator-dependent |
| INT-03 | Helpdesk and ticketing | Create, update, categorise and close tickets; agent handover with full transcript | Zoho, Zendesk, Freshdesk |
| INT-04 | CRM and campaign automation | Bonus eligibility, milestone gifts, retention campaigns | Smartico, Flows, Optimove |
| INT-05 | Game and sportsbook data | Round records, bet history, settlement rules, official results | Provider-dependent |
| INT-06 | Identity and KYC | Verification state, document status, rejection reasons | Provider-dependent |
| INT-07 | Internal comms | Operational alerts, KPI summaries, RG escalation notifications, health status | Slack, Microsoft Teams |
| INT-08 | Player channels | Where the agent talks to players | To be confirmed — see §17.4 |

| ID | Requirement | Detail |
|---|---|---|
| INT-10 | Capability-based | Integrations are described by what they can do, not by vendor. A new back office that provides the same capabilities works without changing any procedure |
| INT-11 | Graceful degradation | Where a system is unavailable, affected procedures disable themselves and route to humans. The agent never guesses at data it could not retrieve |
| INT-12 | Custom platforms | Operators on in-house back offices can be integrated through the same capability contracts |
| INT-13 | Read-only start | An operator can run in read-only mode first and enable changes once they are satisfied |

---

## 16. Non-functional requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-01 | Volume | 80,000–100,000+ live chats per operator per month |
| NFR-02 | Peak elasticity | Absorbs sports-event and tournament spikes without degradation or queueing |
| NFR-03 | Availability | 99.99% uptime |
| NFR-04 | Redundancy | Multi-region deployment — subject to the residency conflict in §17.3 |
| NFR-05 | First response | Instant. No queue at any load |
| NFR-06 | Action latency | An action and its confirmation complete inside a normal conversational pause |
| NFR-07 | Time to value | Integration and procedure deployment in weeks, not quarters |
| NFR-08 | Language | English at launch. 120+ languages is optional scope — see §17.5 |
| NFR-09 | Domain vocabulary | Correct handling of iGaming terminology — turnover, wagering requirement, cashout, RTP, accumulator, free spins — in every supported language |
| NFR-10 | Cost per contact | Materially below the human baseline, measured per resolved contact |
| NFR-11 | Screening latency | RG screening adds no perceptible delay, since it runs on every message |
| NFR-12 | Degradation order | Under load, protective functions and RG screening are the last things to degrade, never the first |
| NFR-13 | Observability | Live visibility of automation rate, escalation rate, RG triggers, action volumes and failures, per operator |
| NFR-14 | Onboarding | A new operator can be configured and live without platform code changes |

---

## 17. Open questions

These are conflicts or gaps in the source material. Each needs a decision before the corresponding requirements are final.

**17.1 — Two different automation targets.** §1.1 of the PRD states both "80% to 90%+ total support automation" and "a 40% reduction in manual support requirements". These describe different products. If 85% of conversations are fully automated, manual workload falls by far more than 40%. One of these is likely a conservative external commitment and the other an internal ambition, but we should know which is which before either appears in a contract.

**17.2 — Volume figure is per operator, per month.** NFR-01 says 80,000–100,000+ monthly chats per operator. We have no concurrency figure, and concurrency is what actually sizes the system. 100,000 monthly chats is roughly 140 an hour on average, but a major fixture could put a large multiple of that into a fifteen-minute window. We need an expected peak concurrency, not just a monthly total.

**17.3 — Multi-region redundancy versus data residency.** NFR-04 requires redundant multi-region deployment; TRU-08 requires data to stay in the licensed jurisdiction. For a single-jurisdiction operator these directly conflict. The resolution is probably multi-zone redundancy inside a region rather than across regions, which changes what 99.99% means.

**17.4 — Channels are unspecified.** Neither source names the channels the agent operates in — live chat, email, in-app, WhatsApp, SMS. Channel choice affects latency expectations, identity verification strength, and message-length constraints. It should not stay open long.

**17.5 — Language scope.** The PRD lists "instant responses in English" under the Player persona and "120+ languages" as an optional NFR. RG screening in particular must work in every language we accept messages in — a distress signal we cannot read is a distress signal we miss. Whatever we support for conversation, we must support for screening.

**17.6 — Volume shares for five of six categories.** Only financial has a figure. We should instrument the rest during the first integration and revisit prioritisation once we have real distribution.

**17.7 — Pricing model is out of scope here.** Cost per contact appears as an internal target (NFR-10). How the product is priced to operators is not addressed in either source.

---

## Appendix — Requirement index

| Area | Prefix | Count |
|---|---|---|
| Responsible gaming | RG | 35 |
| Trust, security and audit | TRU | 28 |
| Financial and transactional | PAY | 25 |
| AI Procedures engine | AIP | 25 |
| Account verification (KYC/AML) | KYC | 21 |
| Account access and security | SEC | 20 |
| Bonuses and promotions | BON | 18 |
| Non-functional | NFR | 14 |
| Sportsbook settlement | SPT | 13 |
| Escalation and handover | ESC | 13 |
| Integrations | INT | 12 |
| VIP and loyalty | VIP | 10 |
| Casino and live dealer | CAS | 10 |
| Proactive engagement *(optional)* | PRO | 9 |
| Geolocation and technical | TEC | 8 |
| **Total** | | **261** |

By action type: **Read 57**, **Escalate 43**, **Protective 40**, **Explain 40**, **Change 10**, with the remaining 71 being platform, integration and non-functional requirements that carry no player-facing action type.

Two things in that spread are worth noticing.

**Only 10 requirements change a player's account.** Everything else reads, explains, protects, or hands over. That is the shape we want: the read path carries the volume and can ship before any change capability is switched on — see INT-13. It also means the expensive safety machinery in §13.2 guards a small, enumerable set of actions rather than the whole product.

**Protective and Escalate together outnumber Read.** That is not padding. In a regulated market the agent's most common correct action, after answering, is to restrict or to hand over — and both need specifying as carefully as the answers do.
