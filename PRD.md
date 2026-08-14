# AI Player Support Agent — Product Requirements

**Owner:** [TBC]  ·  **Status:** Draft, for review
**Reviewers needed:** Compliance, Engineering, Fraud/Payments, Commercial


## 1. Why we're doing this

A mid-size operator runs something like 80,000 to 100,000 live chats a month. The volume is concentrated — a fairly small set of question types produces most of it. Where's my money, why is my account restricted, what happened to my bonus, why did my bet settle that way, why can't I get in.

The trap is assuming concentrated means simple. "Where is my deposit" is one sentence from the player and at least five different situations underneath: the bank declined it, it stopped somewhere in the payment pipeline, it's a phantom charge that never actually reached us, it went to the wrong crypto network, or a provider outage swallowed it. Different lookup each time, different answer each time. Working out which one you're looking at *is* the job, and it's why a single question turns into sixteen requirements in Module A.

The mix also depends on what kind of operator you are. A casino's heaviest traffic is deposits, bonuses and verification. A sportsbook's is settlement disputes, cash-out failures and rejected in-play bets — a completely different set of systems to go and read.

What they have in common is that none of them can be answered from a help centre. They're questions about the state of one person's money, or one specific game round, or one specific bet, and answering honestly means reading whatever live system holds that. So operators throw headcount at it. It's expensive, it's slow at 2am, and it's worst on exactly the days volume spikes.

Existing chatbots have generally made this worse rather than better. They recite the bonus terms at someone who wants to know their own wagering progress, and the player ends up in the queue anyway, now annoyed.


---

## 2. What we're building

An agent that reads the operator's live systems and answers from them. It runs the operator's own support procedures, screens every message for gambling harm, executes the protective actions players ask for, and hands anything it shouldn't decide to a human with the case already worked up.

Sold as a platform, not a bespoke build. Each operator gets an isolated environment with their own procedures, settings and credentials.

The distinction we keep coming back to in reviews:

| A chatbot | This |
|---|---|
| Tells you the bonus terms | Tells you your wagering progress and what's blocking your withdrawal |
| Escalates when it's confused | Escalates when it should, and knows the difference |
| Answers from a knowledge base | Answers from live systems |
| Compliance is an instruction in a prompt | Compliance is a limit on what it can physically do |

That last row is the one that actually matters commercially, and it's the one we undersell. Competitors market automation rate. Nobody in this market is marketing "the agent is structurally incapable of offering a bonus to a self-excluded player," and that is the thing a compliance officer will care about in the room.

---

## 3. Goals

| Goal | Measure | Target |
|---|---|---|
| Automate what the agent is allowed to handle | Eligible-resolution rate — of the contacts it may resolve, the share it closes without a human | 80–90% by V2 |
| Reduce total human workload | Contained rate — of all inbound, the share that never reaches a human. Lower on purpose, because complaints, harm signals and AML holds always go to a person | 50–55% by V2 |
| Be right about people's money | Wrong-answer rate on account questions | Under 0.5% |
| Feel instant | Time to first feedback | Under 1s |
| Answer fast enough | Simple questions ~2s, investigations 5–8s | See §9 |
| Players prefer it | CSAT, measured only on conversations it was allowed to handle | 4.7 |
| Cost far less than a person | Per contact, against a $3–5 human baseline | ~$0.10 |
| Survive the big days | Concurrent chats on a major fixture | 10,000+ |

---

## 4. Who we're building for

Two people actually talk to this agent.

| | What they want | What hurts now | What they get |
|---|---|---|---|
| **Player** | An answer about their own money, right now | Slow replies, scripted answers, and no idea why a withdrawal is stuck or what would unstick it. Being told the terms when they asked about their account | Real account data instead of policy text, in their own language, at 3am. A straight answer about what's blocking them and what clears it |
| **VIP player** | To be recognised without having to explain who they are, and answers that reflect the terms they actually have rather than the published ones | Queued behind routine tickets. Quoted standard limits when theirs are different. Chasing a host who promised something over WhatsApp that never reached the CRM | Instant tier recognition, priority routing, their real caps and entitlements, and a handoff to their own host with the context already attached |

The VIP row has a hard edge that's worth stating in the persona rather than burying in a requirement. Tier changes **how fast** we respond and **who** the conversation goes to. It never changes a remedy, a goodwill ceiling, an RG threshold or a verification gate. That's CMP-06, it's enforced as a runtime assertion with a negative-test suite behind it, and it exists because internal audit and regulators test for exactly that difference.

The practical consequence is that VIP-01 reads the *assigned* tier rather than deriving it from turnover. A large share of real VIPs sit on manual overrides — retained after a bad month, poached from a competitor and dropped straight into the top tier, grandfathered from a legacy programme. An agent that works tier out from deposit volume gets it wrong for precisely the players who complain hardest.

Compliance, support operations and fraud aren't listed here because they don't use the agent — they buy it, govern it, or clean up after it. Their requirements run through the document rather than sitting in this table: compliance and responsible gambling in §7 and §8, operations in §9 and §10, fraud across Module F and the payout-change controls in Module A.

---

## 5. Scope

**In:** inbound player support over chat, with the channel abstracted so we can add more later. Reading live operator systems and answering from them. Creating records — cases, complaints, evidence holds, document submissions. Protective actions the player asks for, from V1. Account-changing actions behind guardrails, from V2. Agent-initiated contact, V3.

**Out:** deciding disputes. Adjudicating game outcomes. Ruling on whether a bet settled correctly. Any decision that awards or confiscates money without a human. Marketing campaign management. Acquisition and onboarding flows. Everything in §8.

**Assumed, and we should confirm each with the first partner:**

- They can expose player state over an API in near real time
- They have written support procedures we can encode, or will work with us to write them
- Their compliance function will review and sign off each procedure before it goes live
- There's a named person who owns the RG escalation queue

The third one is easy to nod along to in a sales call and hard to actually staff. Worth pinning down early.

---

## 6. Requirements

117 across ten modules. Within each module they're roughly ordered by contact volume.

Notes column is only filled in where there's something a reader needs. Most rows are self-explanatory and don't have one.

**Phases:**

- **V1** — reads and explains. Also creates records: cases, complaints, evidence holds, uploads. No money, no entitlement.
- **V1\*** — protective actions. Restrictive only, one-way, no amount involved. Set or lower a limit, take a break, self-exclude, freeze withdrawals. Shipped early deliberately, see §7.2.
- **V2** — changes the account in a permissive or money-moving direction.
- **V3** — the agent starts the conversation.

---

### Module A — Payments & Cashier (PAY)

Everything between the player's bank and their balance. Highest volume module, and the one where we have the most confidence in the requirements because it's the best-understood part of the domain.

Needs from the operator: deposit and withdrawal records, payment provider decline data, live payment-method config, payout limits.

| ID | Feature | What it does | Notes | Phase |
|---|---|---|---|---|
| PAY-01 | Card decline decoder | Explains a failed payment in plain words — no funds, bank blocked gambling, security check abandoned — and says whether the bank or we caused it. Points to a method that actually works for that country and card. | Biggest single payment driver. "Try again" generates a second ticket and can trigger a velocity block | V1 |
| PAY-02 | Withdrawal status tracker | Finds the exact stage a payout has reached and gives an ETA from the operator's real batch cut-off rather than a canned "3–5 days". Opens a case if it's past SLA. | | V1 |
| PAY-03 | Withdrawable balance breakdown | Splits the wallet into cash, bonus, locked cash, money tied up in open bets, pending payouts. States what can actually be withdrawn today and the biggest blocker. Shows what withdrawing now would forfeit. | Most "your maths is wrong" tickets are this. The lobby shows one number and the cashier shows another and nobody has ever explained the gap | V1 |
| PAY-04 | Missing deposit tracer | Follows a missing deposit across our ledger, the provider and the gateway, and says where it stopped. | | V1 |
| PAY-05 | Duplicate charge explainer | Distinguishes a real double charge from a phantom pending authorisation that will drop off in a few days. | Stops chargebacks being filed against charges that never happened | V1 |
| PAY-06 | Return-to-source router | Explains which card or wallet a payout must go back to and why one withdrawal split across several. Flags deposit-only methods. | | V1 |
| PAY-07 | Crypto deposit tracer | Checks confirmations against the network threshold. Diagnoses wrong-network sends, missing memo tags, underpayments. Never promises recovery. | Most crypto errors are genuinely unrecoverable and a false promise here becomes a complaint | V1 |
| PAY-08 | Withdrawal cancellation gate | Handles "cancel my withdrawal and put it back". Checks market and RG status first. Where reversal is banned or the player is flagged, refuses and explains, and never offers it unprompted. | Prohibited in UK and Ontario. "You can cancel and keep playing" is the most natural helpful sentence in the whole product and a licence breach in London | V2 |
| PAY-09 | Payout allowance calculator | States remaining daily, weekly and monthly allowance and the next reset, reading the caps on that specific player rather than the tier table. | VIPs sit on manual overrides. Quoting the tier table gives the wrong answer to the people who complain loudest | V1 |
| PAY-10 | Payment method guidance | Lists what's actually live for that player's country, currency and tier, from the account's real config rather than the marketing page. | | V1 |
| PAY-11 | Provider outage routing | On a known outage, says it's our problem, tells the player not to retry, offers a working alternative. | Retries during an outage compound decline rates and trigger fraud blocks. A 20-minute incident becomes a week-long account problem | V1, notify V3 |
| PAY-12 | Currency and fee explainer | Explains the gap between what was entered and what arrived, quoting the rate stamped on that transaction rather than today's. | | V1 |
| PAY-13 | Payment method manager | Lets a player manage saved deposit methods. Adding or changing a payout destination always needs step-up and proof of ownership. | Getting new bank details onto a verified account is the attacker's whole objective | V2 |
| PAY-14 | Bounced payout re-issue | Names the field that failed, collects corrections under step-up, re-queues through the normal risk path. | | V2 |
| PAY-15 | Chargeback containment | Recognises a card dispute, confirms only that the account is restricted, hands over at high priority. No merits, no evidence, never offers money to make it go away. | Offering a refund to withdraw a dispute breaches scheme rules and damages our own representment | V1 |
| PAY-16 | EDD hold status | Confirms a large withdrawal is under review and lists only the documents Compliance formally asked for. No threshold, no trigger, no timeline. | Explaining why the review fired can be a criminal offence. See §8 | V1 |

---

### Module B — Bonus & Promotions (BON)

Needs from the operator: bonus engine state, campaign terms with version history, game weightings, free-round allocation records.

| ID | Feature | What it does | Notes | Phase |
|---|---|---|---|---|
| BON-01 | Wagering progress tracker | Rebuilds a bet-by-bet ledger showing how much each round moved the requirement after game weighting. States the basis actually recorded on that campaign. | "40× bonus" and "40× deposit+bonus" differ by exactly 2× in real money. Getting this wrong misrepresents a contract term | V1 |
| BON-02 | Bonus code failure diagnostician | Names the exact condition that failed and offers an alternative they genuinely qualify for. | | V1 |
| BON-03 | Missing deposit-match investigator | Works out whether the match should have granted at all. Issues it where the failure was ours and no prior grant exists. | Bonus engines lag 30–90 seconds. Half of these are impatience | V2 |
| BON-04 | Free spin delivery check | Separates "granted here but never allocated by the provider" from "wrong game or stake size" from "already played". | | V2 |
| BON-05 | Free spin winnings explainer | Explains total win, whether winnings carry their own wagering, and whether a conversion cap truncated it. Quotes the terms in force at grant. | | V1 |
| BON-06 | Bonus expiry countdown | Exact expiry in the player's timezone, turnover remaining, and that expiry destroys the bonus and everything won from it. | Players assume only the unused part goes. Finding out afterwards is a complaint | V1, countdown V3 |
| BON-07 | Cashback statement rebuild | Rebuilds the settled cashback line by line. Reads the settled figure, never recalculates. | | V1 |
| BON-08 | Guarded bonus forfeiture | Shows exactly what will be destroyed before doing it, and requires typed confirmation. | Irreversible decision people make in thirty seconds while frustrated | V2 |
| BON-09 | Tournament reconciliation | Explains scoring, reconciles qualifying rounds against displayed position including leaderboard lag. | Most disputes here are the lag, not the scoring | V1 |
| BON-10 | Max-bet breach case file | Presents the breaching round, stake, time and clause version factually. Never reinstates, never says the confiscation is final. | Detected retroactively at withdrawal, so these arrive furious. The remedy is genuinely disputed — some operators void only the excess | V1 |
| BON-11 | Promotional restriction statement | Confirms a restriction applies, in approved wording, and gives the complaint route. Never says what triggered it. | Detection logic circulates on affiliate forums within a day of being explained once | V1 |
| BON-12 | Missions and quests tracker | Explains why a step hasn't ticked and claims completed rewards. | | V2 |

---

### Module C — Sportsbook (SPRT)

New module, added after the domain review. The governing idea is that a bet slip is a frozen contract: price, place terms, payout cap and the applicable rule text are all snapshotted at the moment the bet was struck, not at the moment someone asks about it.

That single fact drives most of the design here, and it's also the most likely production bug — retrieval that always returns current config will quote today's extra-place offer for a bet struck three weeks ago, in writing, and then we have to honour it.

Needs from the operator: bet and leg records, settlement audit trail, versioned sports rules, cash-out logs, event results.

Gating dependency: this whole module needs the operator to supply versioned rule text. If they can't, most of it doesn't ship for them. See open question 5.

| ID | Feature | What it does | Notes | Phase |
|---|---|---|---|---|
| SPRT-01 | Open bet tracker | Lists open and recently settled slips and names the leg holding things up. | Highest volume sportsbook question on a busy Saturday | V1 |
| SPRT-02 | Settlement verification | Pulls the trading system's audit record and the rule frozen on that bet, and explains the settlement against the rule actually applied. | Never from the model's own knowledge of sport. It will be fluent and wrong, and the wrong answer is in writing | V1 |
| SPRT-03 | Void and Rule 4 explainer | Explains why a selection voided and shows the recalculated return from the settlement engine. | The maths must never be computed by the model | V1 |
| SPRT-04 | Cash-out failure diagnostics | Explains a failed cash-out and rebuilds partial cash-out maths. Confirms a cashed-out bet is final. | Timing evidence settles what is otherwise an unwinnable argument | V1 |
| SPRT-05 | Bet rejection explainer | Explains a declined or amended bet from the rejection log. Where the real cause is account-level stake limiting, gives the neutral market-limit answer and hands off. | Exposing risk posture is directly monetisable by arbitrage groups | V1 |
| SPRT-06 | Free bet and acca insurance | Explains stake-not-returned returns, qualification odds, exclusions, expiry. Re-issues only where an incident record proves a platform fault. | | V1, reissue V2 |
| SPRT-07 | Bet builder leg void | Works out whether a voided leg collapsed to 1.00 with the rest standing, or voided the whole slip, from the policy on that specific slip. | Bet builder policy differs from standard accumulator policy. Assuming the general rule gives a confidently wrong answer | V1 |
| SPRT-08 | Each-way and dead-heat calculator | Retrieves the place terms frozen at strike and the divisor applied, then walks through win and place parts separately. | A dead heat divides the stake and pays full odds. It does not halve the odds, and explaining it that way sounds authoritative and is wrong | V1 |
| SPRT-09 | Postponed and abandoned events | Says whether the bet is void, standing or already resettled, quoting the clause verbatim. | Retrieval only. Void windows differ by sport, competition and operator | V1 |
| SPRT-10 | Resettlement notifier | Explains a resettlement after an official result change. Positive ones confirmed. Clawbacks go to a human with the case pre-packaged. | | V1 |
| SPRT-11 | Live stream entitlement | Checks entitlement and clears a stranded session locking the player out. | Time-critical. The match is on now | V1, reset V2 |
| SPRT-12 | Bet history statement | Generates a filtered statement and delivers it through the authenticated channel, not the chat transcript. | | V1 |
| SPRT-13 | Maximum payout cap disclosure | Shows the clause in force at strike and the uncapped theoretical return. States the cap isn't negotiable in chat. | Player finds out about the cap after the win of their life. One of the most-referred issues to dispute bodies | V1 |
| SPRT-14 | Palpable error intake | Presents the clause and both prices without saying whether it was correctly applied. Opens a trading review and a complaint record. | Top driver of dispute referrals against sportsbooks. Defending it or conceding it both become evidence | V1 |
| SPRT-15 | Integrity hold handling | Returns the approved holding statement and suppresses all settlement reasoning on that slip. Escalates silently. | | V1 |

---

### Module D — Casino & Live Games (CAS)

Also new after the domain review. The unit of truth here is the game round, and the thing to understand is that round state and wallet state are separate systems that routinely disagree. The provider owns the round, we own the ledger, and a rollback arriving after we've force-resolved something produces a double credit.

Needs from the operator: provider round logs, wallet ledger, game catalogue and certification data, live table status, incident register.

| ID | Feature | What it does | Notes | Phase |
|---|---|---|---|---|
| CAS-01 | Disconnected round recovery | Classifies the round as paid, unsettled, still open, or rolled back by comparing the provider log against our ledger and the transaction status. | Most common casino ticket. Also the one most likely to double-credit if handled naively. A losing spin isn't refundable — the outcome was fixed when the stake was accepted | V1, resolve V2 |
| CAS-02 | Session balance reconciliation | Rebuilds a plain session statement separating turnover from money actually spent. Routes to the RG check if the pattern suggests chasing. | Players confuse turnover with losses and arrive believing they lost ten times what they did | V1 |
| CAS-03 | Game launch failure triage | Checks provider health before troubleshooting, so a known outage gets a status update rather than a clear-your-cache script. | | V1, notify V3 |
| CAS-04 | Game availability explainer | Explains why a game disappeared and suggests certified alternatives. | A player insisting it worked from another country is logged as a spoofing signal, not argued with | V1 |
| CAS-05 | Rules and paytable explainer | Answers mechanics questions from the title's live paytable for that market. | Vendor marketing copy describes a build the player may not be playing | V1 |
| CAS-06 | RTP and fairness handler | Returns the RTP variant actually deployed for that brand and market, with the certification reference. Never says a game is "due" or the player was unlucky. | The same slot ships in several certified variants. Quoting the headline figure states a legally incorrect number | V1 |
| CAS-07 | Live stream triage | Diagnoses buffering and "my bet wasn't accepted in time" and settles it from the round log. | | V1 |
| CAS-08 | Table limits guidance | Explains a rejected stake and which limits are negotiable. None of the regulatory ones are. | | V1 |
| CAS-09 | Live dealer dispute intake | Captures the claim, pulls the provider's game state, and places a video-retention hold. Never rules on whether the dealer erred. | The hold is the whole value. Retention windows are days, not months, so this has to happen in the first minutes | V1 |
| CAS-10 | Autoplay and spin-speed | Explains why autoplay or turbo is unavailable. Read-only by design — the agent can never enable these. | Offering to enable a prohibited feature is a direct breach | V1 |
| CAS-11 | Buy-feature enquiry | Confirms the purchase was debited and delivered as one round, and whether it was allowed under the active bonus. | Feature buys are commonly excluded and can forfeit winnings. Players rarely know | V1 |
| CAS-12 | Malfunction detection | Spots candidate defects and raises a potential multi-player incident. May quote the malfunction clause, never uses it to void a win. | A defect affecting several players is a reportable incident, not a support ticket | V1 |
| CAS-13 | Jackpot dispute | Explains qualification rules. For an actual hit reports "pending provider and compliance verification" only. | Never confirms it's payable, or the amount, or when. The provider holds and validates the pot | V1 |

---

### Module E — Identity, AML & Financial Crime (KYC)

Needs from the operator: verification partner case state, document requirements by market, risk flags split into disclosable and internal, exclusion registry access.

The split between disclosable and internal risk flags is not optional plumbing. It's the mechanism that makes KYC-09 enforceable rather than aspirational, and it needs to exist on the operator's side.

| ID | Feature | What it does | Notes | Phase |
|---|---|---|---|---|
| KYC-01 | Verification status decoder | Translates rejections into plain words with the deadline remaining. | Vendor reject codes are unreadable, so players resubmit the same rejected document three times | V1 |
| KYC-02 | Inline document upload | Upload widget in the chat for the exact document outstanding, with pre-checks before submission. | Biggest single deflection in this module. Turns explaining a rejection into resolving it | V1 |
| KYC-03 | Pre-withdrawal readiness | Lists every outstanding requirement in one message rather than drip-feeding. VIP tier has no influence here. | Drip-feeding is the top driver of "I've sent you everything three times" | V1 |
| KYC-04 | Requirements advisor | Answers "what counts as proof of address" from the accepted-document list for that market. | | V1 |
| KYC-05 | Source of funds intake | Explains what's being asked for and lists acceptable evidence categories. No threshold, no hint at what would pass, no timeline. | Telling someone what evidence would pass is itself a control failure | V1 |
| KYC-06 | Third-party funding guard | Detects an instrument not in the account holder's name and requests ownership evidence. Accepts no explanation as sufficient. | "It's my wife's card, she said it's fine" is a classic mule pattern, not a resolution | V1 |
| KYC-07 | Identity amendment gatekeeper | Detects requests to change name, DOB, national ID, country or currency and executes none of them, ever. | A silent edit here defeats sanctions and self-exclusion matching | V1 |
| KYC-08 | Duplicate account detection | Identifies linked accounts and says which stays active. Always checks the exclusion register first. | Most duplicates are innocent. The minority that aren't have exactly inverted money treatment | V1 |
| KYC-09 | AML non-disclosure guardrail | Freezes generated output where the record carries a sanctions hit or open investigation, sends approved wording, escalates silently. No change in tone or reply speed. | Disclosure here is a criminal offence with personal liability. This is the canonical suppression control for the whole product | V1 |
| KYC-10 | Underage containment | Suspends play and all money movement, hands to compliance, says nothing about the balance. | Inverse of a normal closure — stakes returned, winnings voided. An automation that closes and pays out does exactly the wrong thing | V1\* |
| KYC-11 | Data request intake | Logs access and deletion requests, clock starting from the player's message, and explains mandatory retention up front. | | V1 |

---

### Module F — Account Security & Access (SEC)

Split out of KYC and Platform during the domain review. The distinction: Module E asks whether the identity is real and clean. This one asks whether the person in the chat right now is the account holder. Different discipline, different owner, and merging them buried the takeover controls inside an AML module where nobody would look for them.

Needs from the operator: authentication logs, lock ledger with disclosure classification, session and device records, location provider, message gateway logs.

| ID | Feature | What it does | Notes | Phase |
|---|---|---|---|---|
| SEC-01 | Login failure triage | Classifies why they can't get in and resolves the one branch that's safe to self-serve. Reset links go to the email on file only, never one given in chat, and never in the same session as a contact change. | The takeover chain is: change the email, then reset the password. Breaking the sequence breaks the attack | V1, reset V2 |
| SEC-02 | Lock reason disclosure router | Splits locks three ways — ordinary explained fully, RG to an officer, investigation gets the vague script. Triggers on what the system returns, not what the player asked. | The player asks about login and the system returns an investigation flag. That's the case that catches people out | V1 |
| SEC-03 | 2FA and code diagnostics | Checks whether the code was delivered, blocked or bounced, and resends to the registered channel only. | "I lost my phone, send it to this number" is the most rehearsed script against gambling support desks | V1, factor change V2 |
| SEC-04 | Location block resolver | Gives the one correct fix. For genuinely out-of-jurisdiction, explains play is unavailable. | Must be structurally incapable of advising anything that helps someone appear to be elsewhere | V1 |
| SEC-05 | Step-up verification | Shared building block. Issues an out-of-band challenge and provides a pass/fail other requirements depend on. | Not a player-facing feature. Required before payout changes, re-issues, point redemption, session termination | V1 |
| SEC-06 | Profile and contact update | Handles the fields a player may change themselves. Email changes need step-up plus a reversal notice to the old address. | The reversal notice is what tells a real player they're being taken over | V2 |
| SEC-07 | Session and device resolver | Lists active sessions and ends others after step-up. Doubles as a self-service "is someone else in my account" check. | | V1\* |
| SEC-08 | Takeover triage | Freeze withdrawals first, revoke sessions, force a reset, build a fraud case. Never decides whether losses are reimbursed. | Attackers monetise through payout, not login. Freezing withdrawals first is the only step that actually stops loss | V1\* |
| SEC-09 | VPN advisory | Explains a rejected session and says to turn it off entirely. Can't answer "which VPN works". | Repeat patterned attempts go to the fraud signal store, not the support queue | V1 |

---

### Module G — Responsible Gambling (RG)

The only module where success is measured in time-to-human rather than deflection. Worth saying that explicitly, because every instinct in a support product pushes the other way.

Needs from the operator: limits and consumption, exclusion state including national schemes, harm markers, a per-market rules table, approved wording per market.

The per-market rules table is a genuine dependency and not a config flag. The UK needs 24 hours plus active reconfirmation on a limit increase. Sweden needs 72. Germany enforces a national cross-operator ceiling we can't exceed regardless of what the player asks. A single `cooling_period_hours` field cannot express that space.

| ID | Feature | What it does | Notes | Phase |
|---|---|---|---|---|
| RG-01 | Limit status diagnostic | Explains a blocked deposit precisely. "€200 of your €250 weekly limit is used, it resets Monday." | Deflects a long tail of "my card is failing" tickets that are actually limit tickets | V1 |
| RG-02 | Immediate limit set and reduction | Sets or lowers a limit straight away and confirms the new value. Never suggests a value, never offers a lighter alternative, never asks "are you sure". | Any friction here is an enforcement finding | V1\* |
| RG-03 | Limit increase with cooling-off | Registers a pending change only, per market rules. Blocks entirely where harm markers are active. | | V2 |
| RG-04 | Cool-off / time-out | Applies the break immediately, ends the session, blocks deposits, suppresses marketing for the duration. | Marketing reaching someone on a break is the most visible failure available to us | V1\* |
| RG-05 | Closure vs self-exclusion triage | Any gambling-related reason for closing is handled as self-exclusion, not administrative closure. On ambiguity, defaults to the more protective option. | Most frequently cited failure in published settlements. The account quietly reopens next week | V1\* |
| RG-06 | National scheme explainer | Explains a register block and that we can't see, shorten or make an exception to it. | Players assume the operator can override it. Implying we might is worse than saying nothing | V1 |
| RG-07 | Marketing suppression interlock | Before any offer, invitation or bonus anywhere in the product, re-checks harm markers, exclusions, limits and consent at the moment of sending. | Canonical control. A player can self-exclude between a campaign being built and the message going out | V1\* |
| RG-08 | Self-exclusion enrolment | Applies exclusion with no friction, no retention offer, no confirmation upsell. Propagates to the national register and sister brands. | | V1\* |
| RG-09 | Harm hard stop | Screens every message whatever the ticket is about. On a trigger, freezes generated output, sends approved wording plus local helplines, escalates to a trained officer. | Runs ahead of everything else, including the conversation-state check. Distress hides inside ordinary questions | V1 |
| RG-10 | Reinstatement intake | States the rule and hands to an officer without exception. Never lifts or shortens, never suggests a new account or a sister brand. | Excluded players present as calm and articulate and frame it as an admin error. This is a permissions problem, not a tone problem | V1 |
| RG-11 | Safer gambling signposting | Serves the right organisations for that country and language from maintained config. Also serves people asking on someone else's behalf. | Must never come from model memory. Wrong helpline numbers are worse than none | V1 |
| RG-12 | Activity statement | Deposits, withdrawals, net position, time played. Neutral facts, no comment on winning, losing or whether to keep playing. | Any commentary reads as encouragement or judgement | V1 |
| RG-13 | Reality check config | Shortening always allowed and immediate. Lengthening capped. Disabling refused where the regulator requires the tool. | The asymmetry is the point | V1\*/V2 |
| RG-14 | Third-party concern intake | Handles someone reporting concern about a player, and players claiming refunds of past losses. Confirms nothing about whether an account exists. | Confirming it exists breaches confidentiality. Ignoring welfare information breaches the licence. Take it, say nothing, escalate | V1 |

---

### Module H — VIP & Retention (VIP)

Needs from the operator: assigned tier and value band, loyalty ledger, host ownership and availability, booked offer entitlements.

One rule runs through the whole module: tier changes speed and channel, never remedies or compliance gates. That's CMP-06, and it's here because internal audit and regulators specifically test for it.

| ID | Feature | What it does | Notes | Phase |
|---|---|---|---|---|
| VIP-01 | Tier recognition and routing | Reads the assigned tier and owning host at session open and applies the matching service policy. Tier is never derived from turnover. | A large share of real VIPs sit on manual overrides — retained after a bad month, poached and placed straight into Platinum. Deriving tier gives the wrong answer to exactly the players who complain hardest | V1 |
| VIP-02 | Loyalty points and progress | Answers points and progress from the live ledger. Exact thresholds only where published in that market. | | V1 |
| VIP-03 | Tier benefit explainer | Shows benefits filtered by market, hiding any that can't lawfully be described there. | | V1 |
| VIP-04 | Comp point redemption | Converts points at the tier rate. Blocked inside the cooling window after any password, email or payment change. | Points to cash then withdraw is a standard takeover cash-out route | V2 |
| VIP-05 | Host escalation | Routes to the named host with context and reports their real callback window. | "Someone will contact you" is what VIPs complain about, more than the wait itself | V1 |
| VIP-06 | Promised offer verification | Searches for a booked entitlement and releases it if found. If there's no record, says so neutrally and routes to the host. | Hosts commit over WhatsApp and it never reaches the CRM. Never grants on a verbal claim, never contradicts the host | V1, release V2 |
| VIP-07 | Pre-approved retention offer | Grants only from a pre-authorised envelope the host set, decrementing each time. Everything passes the RG check. | The agent executes a human's budget. It never exercises commercial judgement | V2 |
| VIP-08 | Tier downgrade explanation | Explains the demotion, grace period and requalification window. | Never says or implies "deposit more to keep your tier". That's a targeted inducement to increase spend | V1 |
| VIP-09 | VIP cashier terms | Explains elevated caps and waivers from the config actually bound to the account. | | V1 |
| VIP-10 | Churn signal detection | Raises a prioritised host task. Logs and alerts only, never picks or fires a save offer. | Churn language and harm language overlap almost completely, and the harm check has to win every tie. This is the requirement people push back on and it should not move | V1 |
| VIP-11 | Dormant player reactivation | Restores context and clears the practical blockers. Unconditional exclusion re-check before any contact. | Dormancy is often an elapsed exclusion or a harm episode, not disinterest | V1/V2/V3 |
| VIP-12 | Event invitation and RSVP | Hospitality admin. Hands anything involving spend to the events team. | | V2 |
| VIP-13 | Upgrade request | Reports the published qualification basis and registers interest. Never promotes, never promises, never implies more deposits would secure a tier. | VIP status legally requires documented affordability checks signed off by a named person | V1 |

---

### Module I — Platform & Account Admin (PLT)

Least developed module in this draft. The requirements are right but thin, and I'd expect this to grow once we see real ticket data — a lot of "it's broken" volume probably lands here and gets misattributed to other modules.

Needs from the operator: versioned terms, device and app compatibility data, message delivery logs, incident register, consent records.

| ID | Feature | What it does | Notes | Phase |
|---|---|---|---|---|
| PLT-01 | Version-pinned terms | Answers policy questions using the version actually in force — the one accepted, or the one live when the disputed event happened. | Quoting today's terms for a three-week-old bet produces a written statement we then have to honour | V1 |
| PLT-02 | App and browser diagnostics | Specific fix for that device and build. Suppresses generic advice while a live incident is running. | | V1 |
| PLT-03 | Deliverability investigation | Investigates "I never got the email" from delivery records. | Sits underneath password resets, one-time codes and withdrawal confirmations, so it removes tickets in three other modules. Scores above its own volume for that reason | V1, resubscribe V2 |
| PLT-04 | Outage broadcaster | Matches incidents to the specific product failing, and gives only the ETA the incident record contains. | Inventing an ETA during an outage turns one complaint into two | V1, restore notify V3 |
| PLT-05 | Communication preferences | Updates consent per channel and records the wording consented to. Refuses to suppress transactional and regulatory messages. | | V1\*/V2 |
| PLT-06 | Session timeout explainer | Distinguishes five look-alike causes of "it keeps logging me out". Read-only. | Four of the five are protective and must not be weakened through a tech-support conversation | V1 |
| PLT-07 | Identifier and timezone resolution | Turns whatever the player supplies into the identifier the target system needs. | Most casino, sportsbook and bonus procedures can't start without this. Players never have the ID we need. Easy to miss when planning, blocks a lot when missing | V1 |
| PLT-08 | Accessibility handler | Adapts output, saves the preference, offers other channels where chat itself is the barrier. | Also surfaces to RG as a possible vulnerability signal | V2 |

---

### Module J — Complaints & Service Recovery (CMP)

Added late, and in hindsight it should have been obvious. Complaint creation was appearing as a bolt-on inside seven other requirements and the goodwill ceiling was defined in four different places. Both are one mechanism wearing several costumes.

Under UK licence conditions a complaint exists the moment dissatisfaction is expressed. The clock starts then, not when someone fills in a form.

Needs from the operator: complaint register, dispute-body config per market, service-recovery matrix, goodwill history.

| ID | Feature | What it does | Notes | Phase |
|---|---|---|---|---|
| CMP-01 | Complaint record creation | Detects dissatisfaction in any module and creates the record before the acknowledgement is sent, clock starting from the player's message. | Resolving it conversationally without a record produces exactly the pattern regulators prosecute as complaint suppression | V1 |
| CMP-02 | Dispute body signposting | Gives the correct free dispute body for the licence the player registered under, proactively at deadlock. | Routes on licensing entity, never country. The same operator's UK brand and .com brand have completely different routes | V1 |
| CMP-03 | Evidence pack assembly | Assembles the transaction trail, round or bet logs, terms version at the time, prior goodwill, and attaches it to the handover. | The agent's most valuable output in every do-not-automate case. It can't decide, but it can do all the work up to the decision | V1 |
| CMP-04 | Complaint status tracking | Stage, elapsed and remaining statutory time, next milestone. Never predicts the outcome. | | V1 |
| CMP-05 | Goodwill guardrail | Single money-out path for the whole product. Issues re-credits only where an independent record proves an operator-side fault, within hard caps, keyed on the real object so two chat windows can't claim it twice. | The ceiling number and which budget it draws from are not ours to set. Open question 7 | V2 |
| CMP-06 | Remedy consistency control | Enforces that outcomes, ceilings, RG thresholds and verification gates are identical regardless of tier. Speed and channel may differ. | Runtime assertion plus a negative-test suite in CI. Regulators test for this specifically | V1 |

---

## 7. Compliance and responsible gambling

### 7.1 How harm detection works

Every message gets screened for distress and gambling harm before any other decision, including on conversations that are closed, handed over, or already escalated. A player in crisis doesn't check the conversation status before typing.

On a trigger the agent stops generating text, sends only wording the operator has approved for that market, offers self-exclusion or a break in the same conversation, escalates to a trained officer at top priority, and gives the correct local support contacts from maintained configuration.

### 7.2 Why protective actions ship in V1

This is the one place we're deliberately breaking our own rule that nothing changes an account before the V2 accuracy gate, so it's worth explaining properly.

Friction in responsible gambling is deliberately asymmetric. The correct design is zero friction on protecting yourself and high friction on undoing it. An agent that answers "where's my deposit" in two seconds and puts "I need to stop gambling" in a queue has inverted exactly the asymmetry the law is built around, and that inversion turns up in published enforcement settlements.

These actions are safe to ship early because they have no wrong direction. Restrictive only, one-way, no amount involved, parameters bound deterministically, approved wording only, and a trained human is still notified. The action removes the delay before protection lands. It doesn't remove the human.

Our external claim changes from "the agent can't change anything" to "the agent can't move money", which is still true and is arguably a stronger claim anyway.

Needs Compliance and CISO sign-off, not just product. Open question 1.

### 7.3 Data handling

Card numbers, bank details and identity numbers are stripped from the player's message before the AI sees it, while leaving ordinary language intact so "I've lost the rent money" still reaches the harm screen.

Each procedure declares what player data it may touch and everything else is dropped before it reaches the AI. The AI only ever receives reasons that may be disclosed to a player — internal risk codes never reach it at all.

No model training on player conversations. Each operator's data stays in its own region.

One thing worth flagging to whoever owns security: chat transcripts containing card numbers would pull the whole platform into PCI scope, which changes the audit surface and the enterprise sales cycle. The scrubber isn't only a compliance nicety.

---

## 8. What we don't automate

This is scope, not guidance. Each of these has to be structurally impossible — the capability doesn't exist for the agent — rather than discouraged in a prompt.

**Where disclosure is a criminal offence.** Never disclose or hint at a sanctions hit or money-laundering investigation, including through a change in tone or a slower reply. Never explain why a large-withdrawal review fired, or name a threshold or a timeline. Never explain the type of a restricted lock when the player only asked about login. Never explain why a promotional restriction was applied. Never confirm to a third party that an account exists — though we do have to take welfare information they give us and escalate it.

The uncomfortable thing about this group is that the dangerous sentences are the *good* ones. "Your withdrawal was held because it exceeded our review threshold" is specific, accurate, empathetic and potentially criminal.

**Where the licence is at stake.** Never lift, shorten or promise the end of a self-exclusion. Never suggest a new account, a different email or a sister brand. Never add friction to a protective action — no "are you sure", no shorter alternative, no retention offer. Never offer a limit increase as a way to complete a blocked deposit. Never generate text on a harm hard stop, or send a survey or promo afterwards. Never treat a gambling-motivated closure as administrative. Never grant a bonus or goodwill to a self-excluded or harm-flagged player. Never promote a player to VIP or imply more deposits would secure a tier. Never present withdrawal reversal in markets where it's banned.

If I had to pick the single most likely way an AI support agent breaches a licence, it's the goodwill one. De-escalation is exactly what the model will reach for.

**Where money is awarded or taken away.** Never uphold or reverse a voided bonus. Never use a malfunction clause to void a win. Never confirm a jackpot is payable, or its amount or timing. Never rule on whether a live dealer erred. Never defend a voided bet as correct or hint it may be honoured. Never reclaim money already credited or withdrawn. Never negotiate a maximum payout cap. Never discuss a card dispute's merits or offer money to withdraw one. Never refund a deposit. Never offer self-service on an underage account or mention the balance. Never apply a softer threshold or higher ceiling because the player is VIP.

**Where security depends on it.** Never tell a player what evidence would pass a source-of-funds check. Never edit name, date of birth, national ID, country or currency. Never send a password reset to an address given in the conversation. Never send a one-time code to a channel given during the conversation. Never help a player appear to be somewhere permitted. Never change a payout destination without step-up and proof of ownership. Never grant an offer on a claimed host promise with no record. Never reveal internal risk scoring or stake limiting. Never resolve dissatisfaction without creating the complaint record.

---

## 9. Quality bar

**Languages.** 30+, with correct handling of gambling terms — parlay, rollover, playthrough, cashout, RTP, accumulator, dead-heat. A mistranslated wagering term is a wrong answer, not a style problem.

**Peak load.** 10,000+ concurrent chats on a major fixture day, with a defined degradation ladder rather than a collapse. Safety screening gets no load exemption. Worth noting the surge hour is also the highest-risk hour for gambling harm — in-play loss chasing is the sportsbook harm signature — so the thing we can't shed load on is also the thing that matters most that evening.

**Availability.** 99.99% on the safety path: intake, screening, escalation. 99.9% on the answering path, which degrades to a human. We can't contract four-nines on the answering path because a model provider outage is a planned graceful degradation, not an incident.

**Speed.** First feedback under 1s. First words under 1s at p95. Simple answer around 2s. Investigation 5–8s at p95. Hard stop at 15s, then escalate.

**Audit.** Every decision recorded and committed before the message goes out. Records can't be altered and any change is detectable. A lawful deletion request removes the personal data while leaving the decision record intact and verifiable.

**Isolation.** One operator's data is never reachable from another's, enforced at the data layer rather than in application code.

---

## 10. Dependencies

Most of this depends on what the operator can actually expose. Every procedure declares what it needs, and where an operator can't supply it that procedure is switched off for them and those questions go to a human.

Realistically a typical operator will run around 60 of the 117 at launch. This is a commercial conversation, not a footnote — the dashboard shows which procedures are live and which are dark and why, so "what am I paying for" is visible up front rather than discovered in month two.

Three hard gates. An operator who can't meet these can't have the corresponding phase at all:

- **Can't distinguish our sent messages from a human agent's** → blocks everything. Most help desks echo our own outbound back through the same webhook, and if we can't tell them apart we either talk over a human or hand the conversation to ourselves.
- **Can't accept our reference on an outbound action** → blocks all V2. Without it we can't reconcile after a timeout, and a duplicate refund can't be undone.
- **Can't supply exclusion status at the moment of sending** → blocks all V3. A nightly export can't tell you what's true now.

---

## 11. Plan

| Phase | Timing | What ships |
|---|---|---|
| 1. Foundation and guardrails | Months 1–2 | Core chat, harm detection, message scrubbing, complaint records, account reads, and the protective action set — limits, breaks, self-exclusion, marketing suppression, withdrawal freeze |
| 2. Depth and coverage | Months 3–4 | Payments depth, bonus and wagering, document upload with regional routing, help-desk integration, step-up verification, identifier resolution |
| 3. Autonomous operations | Months 5–6 | V2 actions behind the accuracy gate: goodwill engine, bonus grants, payment-method management, limit increases. Draft suggestions for human agents |
| 4. Deep vertical | Months 7–9 | Sportsbook, casino round recovery and dispute intake, VIP routing and host handoff |
| 5. Proactive and voice | Months 10–12 | Proactive status updates, expiry countdowns, reactivation, all behind the send-time RG check. Voice |

Phase 3 doesn't start on a date. It starts when phases 1 and 2 have shown a sustained wrong-answer rate under 0.5%, zero generated text on a prohibited topic, zero cases of the agent talking over a human, and an audit trail that's survived a real compliance review.

Actions are where a wrong answer becomes a wrong transaction. The accuracy bar is the gate, not the calendar. I'd expect pressure on this and I don't think it should move.

The timings above assume a design partner is in place from month one. They're indicative, not committed.

---

## 12. Open questions

Things that need a decision from someone who isn't me.

**1. Do we ship protective RG actions in V1?** [Compliance, CISO]
Recommendation is yes. The alternative ships an agent whose protective path is slower than its depositing path, which is the inversion that appears in enforcement settlements. The counter-argument is that it breaks the clean "can't change anything" claim, and I think "can't move money" is a fine substitute.

**2. What resolution rate do we contract?** [Commercial, with whoever owns the first customer relationship]
Recommendation is to publish both numbers — 80–90% eligible, no more than about 55% contained. Needs someone willing to say in the room that a correct escalation is a success. If nobody will, contract on eligible-resolution only and list the exclusions.

**3. Who tells the customer sub-2-second responses aren't achievable with mandatory screening?** [Product, Commercial]
The latency table in §9 is what we can actually do. Moving screening off the critical path would hit the number and destroy the control the product is built on. Not a trade I'd make.

**4. Do we build 99.99% end to end?** [Engineering, Finance]
Recommendation is to split it — four-nines on safety, three-nines on answering. End-to-end fights data residency directly, since the regional cell is the compliance boundary and we can't fail over across regions for a UK or Ontario tenant. It also roughly doubles infrastructure cost.

**5. Who owns the sports and game rules corpus?** [Legal, Commercial]
Recommendation is that the operator supplies it and we index and version it. Owning it makes us liable for settlement accuracy we can't verify. This gates most of Module C — an operator who can't supply versioned rule text doesn't get SPRT-03, 07, 08 or 09, and that's the correct outcome rather than a failure.

**6. Sportsbook depth or payments depth first?** [Product, once we know the first customer]
Recommendation is cross-cutting safety work first, six to eight weeks, then payments unless the first partner is sports-led. Payments reuses connectors we already need.

**7. Does the agent move money in V2, and at what ceiling, from whose budget?** [Finance, Compliance]
Recommendation is yes under CMP-05's guardrails. The ceiling number and the budget line aren't product's to set.

**8. Do we build document upload in V1?** [Engineering, Compliance]
Recommendation is yes but capability-gated per market. It's the largest single deflection in Module E, but a compliant multi-region pipeline sending documents to a fourth party is a project in its own right and shouldn't gate the rest of V1.

---

## Appendix A — Things to know about this industry

Written for anyone joining without an iGaming background. Most of these came out of the domain review and every one of them has bitten a product somewhere.

**Protective friction runs backwards.** Zero friction on protecting yourself, high friction on undoing it. "Are you sure?" on a self-exclusion is an enforcement finding, not good UX.

**Tipping off is criminal and helpfulness is the failure mode.** The dangerous sentences are the accurate, empathetic ones.

**A complaint exists the moment someone is unhappy**, not when they fill in a form. Talking them round without a record is complaint suppression.

**The contract freezes at the event, not the question.** A bet's terms are fixed when it's placed. Retrieval that always returns current terms will quote today's offer for a three-week-old bet.

**"Balance" is at least five numbers.** Cash, bonus, locked cash, money in open bets, pending payouts. An endpoint returning two of them can't answer the biggest question in payments.

**Wagering is turnover and the basis doubles the number.** "40× bonus" and "40× deposit+bonus" differ by exactly 2× in real money. Max-bet breaches are caught retroactively at withdrawal, which is why those tickets arrive furious.

**Jurisdiction means the licence, not the country.** The same operator's two brands have different verification timing, complaint routes and product availability for players living in the same city.

**Never let the AI generate a rule, an RTP or a settlement calculation.** The same slot ships in several certified RTP variants. A dead heat divides the stake and pays full odds — it does not halve the odds.

**Game state and wallet state routinely disagree**, and two chat windows will farm your refund path if the deduplication key is wrong.

**VIP is a regulated category and the churn signal is the harm signal.** Chasing losses scores as both. "Big loss, send cashback" is the precise pattern that produces enforcement action.

---

## Appendix B — What each module needs from the operator

Engineering reference for scoping and for testing whether a prospective operator can support a module at all. Actual endpoints come from each platform during integration.

| Module | Needs |
|---|---|
| PAY | Deposit and withdrawal records, payout stage timeline, provider decline and authorisation data, payment-method config by country and licence, player-level payout limits, per-transaction currency detail, crypto confirmations |
| BON | Active bonuses and wagering ledger, campaign terms with version history, game weighting table, free-round grants and provider allocation state, cashback statements, tournament rules and standings |
| SPRT | Bet and leg records, settlement audit trail, versioned market and sport rules, cash-out request log, event results and revisions, per-bet terms snapshot, payout cap clause |
| CAS | Provider round logs, wallet transaction ledger, unfinished-round policy per provider, game catalogue and certification by market, live table status, paytable by jurisdiction, incident register |
| KYC | Verification partner case state, disclosable rejection reasons, accepted-document matrix by licence, due-diligence cases and evidence requests, payment instrument ownership, linked-account and exclusion-registry match |
| SEC | Authentication attempt log, lock ledger split by disclosability, one-time-code gateway delivery records, location-check provider results, active session and device list, profile change history |
| RG | Limits with consumption and reset, pending limit changes, per-market rules table, exclusion state including national schemes, harm markers, approved wording and support organisations by market, activity summary |
| VIP | Assigned tier and value band, loyalty ledger and tier progress, host ownership and availability, booked offer entitlements and envelopes, tier history, churn score |
| PLT | Versioned terms with acceptance records, device and app compatibility matrix, message delivery and suppression records, incident and maintenance register, consent history |
| CMP | Complaint register with statutory clock, dispute-body config by licence, service-recovery matrix, goodwill history, adjustment ledger |

Cross-cutting: everything needs player profile with licensing entity, live RG status, and the lock ledger. The exclusion check and the harm screen run on every conversation regardless of module.
