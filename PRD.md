# Product Requirements Document
## Autonomous AI Player Support Agent for iGaming

**Version:** 2.0 · **Status:** Draft for product and engineering review
**Market:** iGaming — sportsbooks, online casinos, lottery, esports

> **Scope of this document.** What the product does and why. Technical architecture is in `Design.md`; endpoint-level integration detail is in Appendix B, for engineering reference only.

---

# 1. Overview

## The problem

A mid-size gambling operator handles 80,000–100,000 live chats a month, and the volume concentrates. A modest set of question types produces most of it: where is my money, why is my account restricted, what happened to my bonus, why did my bet settle that way, why can't I get in.

**But a concentrated question is not a simple one.** "Where is my deposit" has at least five different root causes — the bank declined it, it stopped somewhere in the payment pipeline, it is a phantom charge that never reached us, it went to the wrong crypto network, or a provider outage swallowed it. Each needs a different lookup and produces a different answer, and telling them apart is the entire job. That is why one question becomes sixteen payment requirements.

The mix also shifts by vertical. A casino operator's heaviest traffic is deposits, bonuses and verification. A sportsbook's is settlement disputes, cash-out failures and rejected in-play bets — a completely different set of systems to read.

What every one of these has in common is that **none can be answered from a help centre.** They are questions about the state of one person's money, one specific game round, or one specific bet, and an honest answer requires reading the live system that holds it. So operators throw people at the problem — expensive, slow at 2am, and worst exactly when volume spikes on a cup final.


Existing chatbots make this worse. They recite the bonus terms to someone who wants to know their own wagering progress, and the player ends up in the queue anyway, angrier.

## The product

An AI agent that reads the player's real account and answers from it. It runs the operator's own support procedures, spots problem gambling, and hands over to a human whenever it should not answer.

Sold as a platform. Each operator gets an isolated setup with their own procedures, settings and credentials.

## What makes it different

| A chatbot | This agent |
|---|---|
| Tells you the bonus terms | Tells you *your* wagering progress and what is blocking your withdrawal |
| Escalates when confused | Escalates when it *should* — and knows the difference |
| Answers from a knowledge base | Answers from the operator's live systems |
| Compliance is a prompt instruction | Compliance is a structural limit on what it can do |

---

# 2. Goals & Success Metrics

| Goal | Measure | Target |
|---|---|---|
| Handle the routine volume | Eligible-resolution rate — contacts the agent is *allowed* to resolve, handled end to end | 80–90% by V2 |
| Be honest about total coverage | Contained rate — share of **all** inbound handled end to end | 50–55% by V2 |
| Be right about people's money | Wrong-answer rate on account questions | Under 0.5% |
| Feel instant | First feedback to the player | Under 1 second |
| Answer fast enough | Simple questions ~2s; investigations 5–8s | See §8 |
| Players prefer it | CSAT on conversations the agent was allowed to handle | 4.7 / 5 |
| Cost far less than a person | Cost per contact vs $3–5 human baseline | ~$0.10 |
| Survive the big days | Concurrent chats during a major fixture | 10,000+ |

> **Why two resolution numbers.** A responsible-gambling escalation, a complaint, and an anti-money-laundering hold are all **successes** when routed correctly — but they count as misses against a single blended figure. Publishing one number without saying which denominator it uses guarantees an argument with the first customer in month three.

---

# 3. Who This Is For

| Persona | What they want | What hurts today | What the agent gives them |
|---|---|---|---|
| **The player** | An answer about their own money, now | Slow replies, scripted answers, no idea why a withdrawal is stuck | Real account data, in their language, at 3am |
| **VIP & Retention Manager** | Keep high-value players happy | VIPs stuck behind routine tickets; hosts' verbal promises never reach the CRM | Tier recognition, priority routing, warm handoff with context |
| **Compliance & RG Officer** | No licence breaches, no missed harm signals | Invented terms, promotional language, missed distress | Deterministic triggers, approved wording only, complete audit trail |
| **Support Operations Lead** | Low queue times, controlled cost | Seasonal churn, tournament spikes | Help-desk integration, honest capability reporting, clean fallback |
| **Fraud & Payments Analyst** | Stop takeover without blocking real players | Support desks are the softest social-engineering target | Structural blocks on payout changes, step-up checks, evidence packs |

---

# 4. Scope

## In scope

- Inbound player support across chat, with the channel abstracted so more can be added
- Reading player account state and answering from it
- Creating records — cases, complaints, evidence holds, document submissions
- Protective actions the player asks for: limits, breaks, self-exclusion *(from V1)*
- Account-changing actions behind guardrails *(from V2)*
- Agent-initiated contact *(V3)*

## Out of scope

- Deciding disputes, adjudicating game outcomes, or ruling on whether a bet settled correctly
- Any decision that confiscates or awards money without a human
- Marketing campaign management
- Player acquisition or onboarding flows
- Anything on the do-not-automate list in §7

## Assumptions

- The operator can expose player state through an API in near real time
- The operator has written support procedures we can encode, or will work with us to write them
- The operator's compliance function will review and sign off every procedure before it goes live
- A named person at the operator owns the responsible-gambling escalation queue

---

# 5. Functional Requirements

**117 requirements across 10 modules**, ordered within each module by contact volume, highest first.

### Phase key

| Phase | Meaning |
|---|---|
| **V1** | Reads and explains. Also creates records — cases, complaints, evidence holds, uploads. No money, no entitlement |
| **V1\*** | **Protective actions.** Restrictive only, one-way, no amount: set or lower a limit, take a break, self-exclude, freeze withdrawals. Shipped early on purpose — see §6.2 |
| **V2** | Changes the account in a permissive or money-moving direction: grant a bonus, refund, raise a limit, change credentials |
| **V3** | The agent starts the conversation |

---

## Module A — PAY · Payments & Cashier

*Everything between the player's bank and their operator balance.*
**Needs from the operator:** deposit and withdrawal records, payment-provider decline data, live payment-method configuration, payout limits.

| ID | Feature | What it does | Why it matters | Phase |
|---|---|---|---|---|
| PAY-01 | Card Decline Decoder | Explains a failed payment in plain words — no funds, bank blocked gambling, security check abandoned — and says whether the bank or the operator caused it. Points to a method that actually works for that country and card. | The single largest payment contact driver. "Try again" is the answer that generates a second ticket and a velocity block | V1 |
| PAY-02 | Withdrawal Status Tracker | Finds the exact stage a payout has reached and gives a real ETA from the operator's actual batch cut-off — never a canned "3–5 days". If it is late, opens a case instead of repeating the timeline. | "Where is my money" is the most emotionally charged routine question in the product | V1 |
| PAY-03 | Withdrawable Balance Breakdown | Splits the wallet into cash, bonus, locked cash, money tied up in open bets and pending payouts, then states exactly what can be withdrawn today and the single biggest blocker. Shows what withdrawing now would forfeit. | Almost every "your maths is wrong" ticket is this gap. The lobby shows one number, the cashier another, and nobody has ever explained the difference | V1 |
| PAY-04 | Missing Deposit Tracer | Follows a missing deposit across the operator ledger, the payment provider and the gateway, and says where it stopped. Where the money clearly left the bank, produces an evidenced escalation. | The player is certain they paid. Being able to prove where it went is the difference between a resolution and a chargeback | V1 |
| PAY-05 | Duplicate Charge Explainer | Tells a real double charge apart from a phantom one — a failed authorisation still showing as pending that will drop off in a few days. | Prevents a chargeback filed against a charge that never actually happened | V1 |
| PAY-06 | Return-to-Source Router | Explains which card or wallet a payout must return to and why one withdrawal was split across several. Flags deposit-only methods and names the fallback. | Players read closed-loop rules as obstruction. Explained clearly and up front, they read as fraud protection | V1 |
| PAY-07 | Crypto Deposit Tracer | Checks confirmations against the network threshold and diagnoses wrong-network sends, missing memo tags and underpayments. Never promises recovery. | Crypto errors are usually unrecoverable, and a false promise here becomes a formal complaint | V1 |
| PAY-08 | Withdrawal Cancellation Gate | Handles "cancel my withdrawal and put it back". Checks the player's market and RG status first. Where reversal is banned or the player is flagged, refuses and explains — and never offers it. | Reverse withdrawal is prohibited in the UK and Ontario. "You can cancel and keep playing" is the most natural helpful sentence in the product and a licence breach in London | V2 |
| PAY-09 | Payout Allowance Calculator | States the remaining daily, weekly and monthly withdrawal allowance and the next reset, reading the caps set on that specific player — which override the tier table. | VIPs and negotiated accounts sit on manual overrides. Quoting the tier table gives the wrong answer to the players who complain hardest | V1 |
| PAY-10 | Payment Method Guidance | Lists the methods actually live for that player's country, currency and tier, with limits and fees — from the account's real configuration, not the marketing page. | The marketing page lists everything. The account allows a subset. The gap is a ticket | V1 |
| PAY-11 | Provider Outage Routing | On a known payment outage, says it is our problem, tells the player **not** to retry, and offers a working alternative. | Retries during an outage compound decline rates and trigger fraud blocks, turning a 20-minute incident into a week-long account problem | V1 · notify V3 |
| PAY-12 | Currency & Fee Explainer | Explains the gap between what was entered and what arrived — conversions, bank markup, fixed fees — quoting the rate stamped on that transaction, never today's. | "I deposited €100 and got €96" is a trust question, not a maths question | V1 |
| PAY-13 | Payment Method Manager | Lets a player manage their saved deposit methods. **Adding or changing a payout destination always requires step-up verification and proof of ownership.** | Getting new bank details onto a verified account is an attacker's entire objective | V2 |
| PAY-14 | Bounced Payout Re-issue | On a returned payout, names the exact field that failed, collects corrections under step-up, and re-queues through the normal risk path. | Otherwise the player waits, re-submits blind, and bounces again | V2 |
| PAY-15 | Chargeback Containment | Recognises a card dispute, confirms only that the account is restricted, and hands over at high priority. Discusses no merits and never offers money to make it go away. | Offering a refund to withdraw a dispute breaches card-scheme rules and damages our own case | V1 |
| PAY-16 | Enhanced Due Diligence Status | Confirms a large withdrawal is under review and lists **only** the documents Compliance formally requested. States no threshold, no trigger, no timeline. | Explaining *why* a review fired can be a criminal offence. See §7 | V1 |

---

## Module B — BON · Bonus, Promotion & Gamification

*The bonus lifecycle, from grant to forfeiture.*
**Needs from the operator:** bonus engine state, campaign terms with version history, game weightings, free-round allocation records.

| ID | Feature | What it does | Why it matters | Phase |
|---|---|---|---|---|
| BON-01 | Wagering Progress Tracker | Rebuilds a bet-by-bet ledger showing how much each round moved the requirement after game weighting, and states the basis **actually recorded on that campaign** — bonus-only or deposit-plus-bonus. | Those two bases differ by exactly 2× in real money. Stating the wrong one misrepresents a contract term | V1 |
| BON-02 | Bonus Code Failure Diagnostician | Names the exact condition that failed — campaign closed, country excluded, code used, minimum not met — and offers a live alternative they genuinely qualify for. | "Invalid code" is the least useful error message in the product | V1 |
| BON-03 | Missing Deposit-Match Investigator | Decides whether the match should have granted at all. Where the failure was ours and no prior grant exists, issues it. Otherwise explains the unmet condition and does not credit. | Bonus engines lag 30–90 seconds. Half these tickets are impatience; the other half are real | V2 |
| BON-04 | Free Spin Delivery Check | Separates "granted here but never allocated by the game provider" from "wrong game or stake size" from "already played". Re-syncs where the provider genuinely failed. | The player sees nothing in either case, but only one of them is fixable by us | V2 |
| BON-05 | Free Spin Winnings Explainer | Explains what happened after a free-spin set: total win, whether winnings carry their own wagering, and whether a conversion cap cut the balance. Quotes the terms in force **when the bonus was granted**. | Winning £400 and seeing £50 credited is a complaint unless the cap was explained | V1 |
| BON-06 | Bonus Expiry Countdown | Gives the exact expiry in the player's timezone and — the part everyone gets wrong — that expiry destroys the bonus **and everything won from it**. | Players believe only the unused part expires. Discovering otherwise afterwards is a formal complaint | V1 · countdown V3 |
| BON-07 | Cashback Statement Rebuild | Rebuilds the settled cashback line by line: qualifying losses, exclusions, void bets, tier rate, cap. Reads the settled figure — never recalculates it. | Cashback disputes are arithmetic arguments the operator always loses without a line-by-line trail | V1 |
| BON-08 | Guarded Bonus Forfeiture | Handles "cancel my bonus so I can withdraw" by first showing exactly what will be destroyed and what cash will remain, then requiring typed confirmation. | This is an irreversible decision players make in thirty seconds while frustrated | V2 |
| BON-09 | Tournament & Leaderboard Reconciliation | Explains scoring from the campaign config, reconciles qualifying rounds against displayed position including leaderboard lag, and confirms when prizes pay. | Leaderboards update on a delay. Most disputes are the delay, not the scoring | V1 |
| BON-10 | Max-Bet Breach Case File | Where winnings were voided for an oversized bet, presents the exact round, stake, time and clause version factually. **Never reinstates and never says the confiscation is final.** | Breaches are detected retroactively at withdrawal, so these tickets arrive at maximum temperature — and the remedy is genuinely disputed | V1 |
| BON-11 | Promotional Restriction Statement | Confirms only that a restriction applies, in approved neutral wording, and gives the complaint route. **Never reveals what triggered it.** | Detection logic circulates on affiliate forums within a day of being explained once | V1 |
| BON-12 | Missions & Quests Tracker | Explains why a step has not ticked — wrong game, stake below threshold, outside the window, ingestion lag — and claims completed rewards. | Gamification is designed to be checked constantly, so it generates constant contacts | V2 |

---

## Module C — SPRT · Sportsbook

*A bet slip is a frozen contract. Everything here reads the terms as they were when the bet was struck.*
**Needs from the operator:** bet and leg records, settlement audit trail, versioned sports rules, cash-out logs, event result feed.

| ID | Feature | What it does | Why it matters | Phase |
|---|---|---|---|---|
| SPRT-01 | Open Bet & Settlement Tracker | Lists open and recently settled slips and names the specific leg holding up the slip and when it settles. | On a busy Saturday this is the single highest-volume sportsbook question | V1 |
| SPRT-02 | Settlement Verification | For a disputed settlement, pulls the trading system's audit record and the market rule frozen on that bet, and explains the settlement against the rule **actually applied**. Never from the model's own knowledge of sport. | A model answering from memory will be fluent and wrong, and the wrong answer is in writing | V1 |
| SPRT-03 | Void & Rule 4 Explainer | Explains why a selection voided and shows the recalculated return **from the settlement engine's figures**. | Rule 4 deductions are counterintuitive and the maths must never be computed by the model | V1 |
| SPRT-04 | Cash-Out Failure Diagnostics | Explains a failed cash-out — price moved, market suspended mid-request, bonus-funded stake — and rebuilds partial cash-out maths. Confirms a cashed-out bet is final. | "I pressed cash out and nothing happened" is a trust event, and the timing evidence settles it objectively | V1 |
| SPRT-05 | Bet Rejection & Odds Change Explainer | Explains a declined or amended bet from the rejection log. **Where the real cause is account-level stake limiting, gives the neutral market-limit answer and hands off.** | Revealing risk posture is directly monetisable by arbitrage groups | V1 |
| SPRT-06 | Free Bet & Acca Insurance Mechanics | Explains stake-not-returned returns, qualification odds, exclusions and expiry, and reconciles expected against credited. Re-issues only where an incident record proves a platform fault. | Free bet mechanics are the most misunderstood promotion in sport | V1 read · V2 reissue |
| SPRT-07 | Bet Builder Leg Void Handler | Determines whether a voided leg collapsed to odds of 1.00 with the rest standing, or voided the whole slip — reading the policy on **that specific slip**, not the general rule. | Bet builder void policy differs from standard accumulator policy, and assuming the wrong one gives a confidently wrong answer | V1 |
| SPRT-08 | Each-Way & Dead-Heat Calculator | Retrieves the place terms frozen at strike and the dead-heat divisor applied, then walks through win and place parts separately. | A dead heat divides the **stake** and pays full odds. Explaining it as halved odds is wrong and sounds authoritative | V1 |
| SPRT-09 | Postponed & Abandoned Event Tracker | Says whether the bet is void, standing, or already resettled, quoting the governing clause verbatim. | Void windows differ by sport, competition and operator. This is retrieval, never generation | V1 |
| SPRT-10 | Resettlement & VAR Revision Notifier | Explains a resettlement after an official result change. **Positive ones are confirmed; clawbacks go to a human with the case pre-packaged.** | Reclaiming money already credited or withdrawn is contested territory and can drive a balance negative | V1 |
| SPRT-11 | Live Stream Entitlement Support | Checks streaming entitlement and clears a stranded session locking the player out. Explains blackouts as rights restrictions. | Time-critical: the player wants to watch the match that is on now | V1 read · V2 reset |
| SPRT-12 | Bet History Statement | Generates a filtered statement and delivers it **through the authenticated channel only**, never restated in the chat transcript. | Bet history is financial data and does not belong in a transcript that gets copied around | V1 |
| SPRT-13 | Maximum Payout Cap Disclosure | Where returns were capped, shows the clause in force at strike and the uncapped theoretical return. States the cap is not negotiable in chat and offers escalation. | The player learns of the cap after the win of their life. Among the most-referred issues to dispute bodies | V1 |
| SPRT-14 | Palpable Error Dispute Intake | Presents the obvious-error clause and the original and corrected price **without saying whether it was correctly applied**. Opens a trading review and a complaint record. | The top driver of dispute referrals against sportsbooks. Either defending or conceding becomes evidence | V1 |
| SPRT-15 | Integrity Hold Handling | Returns only the approved holding statement and suppresses all settlement reasoning on that slip. Escalates silently. | Prejudices a sporting-body investigation, and where suspicion underlies it, breaches disclosure law | V1 |

---

## Module D — CAS · Casino & Live Games

*The game round is the unit of truth.*
**Needs from the operator:** provider round logs, wallet ledger, game catalogue and certification data, live table status, incident register.

| ID | Feature | What it does | Why it matters | Phase |
|---|---|---|---|---|
| CAS-01 | Disconnected Round Recovery | Classifies the round as paid, unsettled, still open, or rolled back by comparing the provider log against our ledger. Explains that a losing spin is not refundable — the outcome was fixed when the stake was accepted. | The most common casino ticket, and the one most likely to produce a double credit if handled naively | V1 read · V2 resolve |
| CAS-02 | Session Balance Reconciliation | Rebuilds a plain session statement — opening balance, rounds, staked, won, **net position** — separating turnover from real money spent. Routes to the RG check if the pattern suggests chasing. | Players confuse turnover with losses and arrive believing they lost ten times what they did | V1 |
| CAS-03 | Game Launch Failure Triage | Checks provider health **before** troubleshooting, so a known outage gets a status update rather than a clear-your-cache script. | Sending a player through five diagnostic steps during a provider outage is the fastest way to lose them | V1 · notify V3 |
| CAS-04 | Game Availability Explainer | Explains why a game disappeared — not certified here, geo-blocked, delisted, withdrawn for recertification — and suggests certified alternatives. | "It worked last week" is usually a licensing change nobody told the player about | V1 |
| CAS-05 | Game Rules & Paytable Explainer | Answers mechanics questions from the title's **live paytable for that market**, not vendor marketing copy. Explains how a specific win was calculated. | Game builds differ by market. Vendor copy describes a game the player may not be playing | V1 |
| CAS-06 | RTP & Fairness Enquiry Handler | Returns the RTP variant **actually deployed** for that brand and market, with the certification reference, and explains it as a long-run figure that never applies to one session. Never says a game is "due" or the player was "unlucky". | The same slot ships in several certified RTP variants. Quoting the headline figure states a legally incorrect number | V1 |
| CAS-07 | Live Stream Performance Triage | Diagnoses buffering, black screens and "my bet wasn't accepted in time" and settles the argument objectively from the round log. | Live casino disputes are otherwise unwinnable arguments about what the player saw | V1 |
| CAS-08 | Table Limits & Stake Guidance | Explains a rejected stake and makes clear which limits are negotiable — none of the regulatory ones. Recommends a table that fits. | Players read a regulator-mandated stake cap as the operator being difficult | V1 |
| CAS-09 | Live Dealer Dispute Intake | Captures the claim, pulls the provider's game-state record, and **places a video-retention hold before the footage ages out**. Never rules on whether the dealer erred. | Retention windows are days, not months. The hold placed in the first minutes is the whole value | V1 |
| CAS-10 | Autoplay & Spin-Speed Responder | Explains why autoplay or turbo is unavailable and why spins have a minimum duration in some markets. **Read-only by design — the agent can never enable these.** | Offering to enable a prohibited feature is a direct licence breach | V1 |
| CAS-11 | Buy-Feature Purchase Enquiry | Confirms a feature buy was debited and delivered as one round, and whether it was allowed under the player's active bonus. | Feature buys are commonly excluded under bonus terms and can forfeit winnings — players rarely know this | V1 |
| CAS-12 | Suspected Malfunction Detection | Spots candidate defects and raises a potential **multi-player incident**. May quote the published malfunction clause; **must never use it to void a win.** | A defect affecting several players is a reportable incident, not a support ticket | V1 |
| CAS-13 | Progressive Jackpot Dispute | Explains qualification rules. For an actual hit, reports "pending provider and compliance verification" only — **never confirms it is payable, or its amount, or when.** | The pot is held and validated by the provider. A premature confirmation is a promise we cannot keep | V1 |

---

## Module E — KYC · Identity, AML & Financial Crime

*Who the player is, and whether their money is clean.*
**Needs from the operator:** verification partner case state, document requirements by market, risk flags split into disclosable and internal, exclusion registry access.

| ID | Feature | What it does | Why it matters | Phase |
|---|---|---|---|---|
| KYC-01 | Verification Status Decoder | Translates verification rejections into plain words — "the corners of your ID were cropped", "your bill is more than three months old" — with the deadline remaining. | Vendor reject codes are unreadable, so players re-submit the same rejected document three times | V1 |
| KYC-02 | Inline Secure Document Upload | Shows an upload widget in the chat for the exact document outstanding, checks glare, crop and expiry before submitting, and confirms the review time. | The single largest deflection in this module — it turns explaining a rejection into resolving it | V1 |
| KYC-03 | Pre-Withdrawal Readiness Check | Lists **every** outstanding requirement in one message rather than drip-feeding them. Explains the withdrawal is held, not cancelled. VIP tier has no influence here. | Drip-feeding requirements is the top driver of "I've sent you everything three times" | V1 |
| KYC-04 | Verification Requirements Advisor | Answers "what counts as proof of address" from the accepted-document list for that player's market, including recency rules and excluded types. | Prevents the rejection before it happens | V1 |
| KYC-05 | Source of Funds Evidence Intake | Explains what is being asked for and lists acceptable evidence categories. **States requirements only — no threshold, no hint at what would pass, no timeline.** | Telling a player what evidence would pass is itself a control failure | V1 |
| KYC-06 | Third-Party Funding Guard | Detects a card or payout destination not in the account holder's name, explains the rule neutrally, and requests ownership evidence. **Accepts no explanation as sufficient.** | "It's my wife's card, she said it's fine" is a classic money-mule pattern, not a resolution | V1 |
| KYC-07 | Identity Amendment Gatekeeper | Detects requests to change name, date of birth, national ID, country or currency and **executes none of them, ever**. Collects documents and opens a two-person compliance review. | A silent edit to these fields defeats sanctions and self-exclusion matching | V1 |
| KYC-08 | Duplicate Account Detection | Identifies linked accounts and says which stays active. **Always checks the exclusion register first** — a linked self-excluded identity is a breach with the opposite money treatment. | Most duplicates are innocent. The minority that are not have exactly inverted handling | V1 |
| KYC-09 | AML Non-Disclosure Guardrail | Where the record carries a sanctions hit or open investigation, freezes generated output, sends only approved wording, and escalates silently — with **no change in tone or reply speed**. | Disclosure here is a criminal offence with personal liability. See §7 | V1 |
| KYC-10 | Underage Containment | On a failed age check, suspends play and all money movement and hands to compliance. **Says nothing about the balance.** | Underage is the inverse of a normal closure — stakes returned, winnings voided. An automation that pays out does exactly the wrong thing | V1\* |
| KYC-11 | Data Request Intake | Logs access and deletion requests with the clock starting from the **player's message**, and explains that verification and transaction records are under mandatory retention. | Sets the expectation before the player believes we agreed to delete everything | V1 |

---

## Module F — SEC · Account Security & Access

*Not "is this identity real" — that is Module E. This is "is this person the account holder right now".*
**Needs from the operator:** authentication logs, lock ledger with disclosure classification, session and device records, location-check provider, message gateway logs.

| ID | Feature | What it does | Why it matters | Phase |
|---|---|---|---|---|
| SEC-01 | Login Failure Triage | Classifies why the player cannot get in and resolves the one branch that is safe to self-serve. **Reset links go to the email on file only — never one given in chat — and never in the same session as a contact change.** | The takeover chain is: change the email, then reset the password. Breaking that sequence breaks the attack | V1 · V2 reset |
| SEC-02 | Lock Reason Disclosure Router | Splits locks three ways: ordinary ones explained fully, RG ones to an officer, and investigation ones getting the deliberately vague script. **Triggers on what the system returns, not what the player asked.** | The player asks about login; the system returns an investigation flag. That is the dangerous case | V1 |
| SEC-03 | 2FA & Code Delivery Diagnostics | Checks whether the code was delivered, blocked or bounced, and resends **to the registered channel only**. Never removes a second factor inline. | "I lost my phone, send it to this number" is the most rehearsed script against gambling support desks | V1 · V2 factor change |
| SEC-04 | Location Block Resolver | Gives the one correct fix for a location failure. For genuinely out-of-jurisdiction, explains play is unavailable. **Structurally incapable of advising anything that helps a player appear to be elsewhere.** | Helping someone appear to be somewhere permitted is facilitating unlicensed gambling | V1 |
| SEC-05 | Step-Up Verification Service | A shared building block: issues an out-of-band challenge to registered channels and provides a pass/fail other requirements depend on. | Required before payout changes, re-issues, point redemption and session termination | V1 |
| SEC-06 | Profile & Contact Update | Handles the fields a player may change themselves, validated against policy for their market and verification state. Email changes need step-up plus a reversal notice to the old address. | The reversal notice is what tells a real player they are being taken over | V2 |
| SEC-07 | Session & Device Resolver | Lists active sessions with device and location and ends other sessions after step-up. Doubles as a self-service "is someone else in my account?" check. | Gives a worried player an immediate answer instead of a queue | V1\* |
| SEC-08 | Account Takeover Triage | Runs a fixed sequence: **freeze withdrawals first**, revoke sessions, force a reset, then build a fraud case. Never decides whether losses are reimbursed. | Attackers monetise through payout, not login. Freezing withdrawals first is the only step that stops loss | V1\* |
| SEC-09 | VPN Detection Advisory | Explains a session rejected for VPN signals and tells the player to turn it off entirely. **Cannot answer "which VPN works".** | Repeat patterned attempts are a fraud signal, not a support issue | V1 |

---

## Module G — RG · Responsible Gambling & Player Protection

*The one module whose success metric is time-to-human, not deflection.*
**Needs from the operator:** limits and consumption, exclusion state including national schemes, harm markers, market rules table, approved wording per market.

| ID | Feature | What it does | Why it matters | Phase |
|---|---|---|---|---|
| RG-01 | Limit Status Diagnostic | Explains a blocked deposit precisely: "€200 of your €250 weekly limit is used; it resets Monday." | Deflects a large tail of "my card is failing" tickets that are really limit tickets | V1 |
| RG-02 | Immediate Limit Set & Reduction | Sets or lowers a limit straight away in the chat and confirms the new value. **Never suggests a value, never offers a lighter alternative, never asks "are you sure".** | Any friction here is an enforcement finding | V1\* |
| RG-03 | Limit Increase with Cooling-Off | Registers a **pending** change only, per market rules, and blocks entirely where harm markers are active. | Some markets require active reconfirmation that must not auto-apply. Others cap increases nationally | V2 |
| RG-04 | Cool-Off / Time-Out | Applies the break immediately, ends the session, blocks deposits and **suppresses all marketing for the duration**. States plainly it cannot be lifted early. | Marketing reaching someone on a break is the most visible possible failure | V1\* |
| RG-05 | Closure vs Self-Exclusion Triage | Classifies every closure request by reason. **Any gambling-related reason is handled as self-exclusion**, not administrative closure. On ambiguity, defaults to the more protective option. | The most frequently cited failure in published settlements — the account is quietly reopened next week | V1\* |
| RG-06 | National Scheme Explainer | Explains a block from a national register, what it covers, and that we **cannot see, shorten or make an exception to** it. Never hints at playing elsewhere. | Players believe the operator can override it. It cannot, and implying otherwise is worse than saying nothing | V1 |
| RG-07 | Marketing Suppression Interlock | Before **any** offer, invitation or bonus anywhere in the product, re-checks harm markers, exclusions, limits and consent at the moment of sending — and blocks. | A player can self-exclude between a campaign being built and the message going out | V1\* |
| RG-08 | Self-Exclusion Enrolment | Applies self-exclusion with **no friction, no retention offer, no confirmation upsell**, using fixed approved wording, and propagates to the national register and sister brands. | Friction on self-exclusion is a documented enforcement finding | V1\* |
| RG-09 | Harm & Distress Hard Stop | Screens **every** message, whatever the ticket is about, and on any trigger freezes generated output, sends approved wording plus local helplines, and escalates to a trained officer at top priority. | Distress hides inside ordinary questions. Screening cannot attach to topics | V1 |
| RG-10 | Reinstatement Request Intake | States the rule and hands to a trained officer without exception. **Never lifts or shortens an exclusion, never suggests a new account or a sister brand.** | Excluded players present as calm and persuasive and frame it as an admin error. This is a permissions problem, not a tone problem | V1 |
| RG-11 | Safer Gambling Signposting | Serves the right support organisations for the player's country and language, from maintained configuration. Also serves people asking on someone else's behalf. | Wrong helpline numbers are worse than none. This must never come from model memory | V1 |
| RG-12 | Activity Statement | Generates the regulator-mandated view — deposits, withdrawals, net position, time played — as neutral facts with **no comment on winning, losing or whether to keep playing**. | Any commentary here reads as either encouragement or judgement | V1 |
| RG-13 | Reality Check Configuration | Shortening always allowed and immediate; lengthening capped; disabling refused where the regulator requires the tool. | The asymmetry is the point | V1\* / V2 |
| RG-14 | Third-Party Concern Intake | Handles someone reporting concern about a player, and players claiming refunds of past losses. Captures the information and **confirms nothing about whether an account exists**. | Confirming an account exists breaches confidentiality; ignoring welfare information breaches the licence. Take it, say nothing, escalate | V1 |

---

## Module H — VIP · VIP, Retention & CRM

*Tier changes speed and channel. It never changes remedies or compliance gates.*
**Needs from the operator:** assigned tier and value band, loyalty ledger, host ownership and availability, booked offer entitlements.

| ID | Feature | What it does | Why it matters | Phase |
|---|---|---|---|---|
| VIP-01 | Tier Recognition & Routing | Reads the **assigned** tier and owning host at session open and applies the matching service policy. Tier is never derived from turnover. | A large share of real VIPs sit on manual overrides. Deriving tier gives the wrong answer to exactly the players who complain hardest | V1 |
| VIP-02 | Loyalty Points & Tier Progress | Answers points and progress from the live ledger. Gives an exact threshold **only where it is published** in that market. | Publishing an unpublished threshold turns a discretionary programme into a contractual promise | V1 |
| VIP-03 | Tier Benefit Explainer | Shows benefits for that tier, **filtered by market**, hiding any that cannot lawfully be described there. | Describing an incentive that is restricted in that market is itself the breach | V1 |
| VIP-04 | Comp Point Redemption | Converts points at the tier rate. **Blocked inside the cooling window after any password, email or payment change.** | Points to cash then withdraw is a standard takeover cash-out route | V2 |
| VIP-05 | Host Escalation & Warm Handoff | Routes to the named host with full context and reports the host's **real callback window** — not "someone will contact you". | The vague promise is what VIPs complain about, more than the wait itself | V1 |
| VIP-06 | Promised Offer Verification | Searches for a **booked** entitlement and releases it if found. If there is no record, says so neutrally and routes to the host. **Never grants on a verbal claim, never contradicts the host.** | Hosts commit over messaging apps and those commitments never reach the CRM. Free-money exploits circulate in VIP groups within hours | V1 · V2 release |
| VIP-07 | Pre-Approved Retention Offer | Grants only from a **pre-authorised envelope the host set**, decrementing it each time, and everything passes the RG check. | The agent executes a human's budget. It never exercises commercial judgement | V2 |
| VIP-08 | Tier Downgrade Explanation | Explains a demotion, the grace period and the requalification window. **Never says or implies "deposit more to keep your tier".** | That sentence is a targeted inducement to increase spend | V1 |
| VIP-09 | VIP Cashier Terms | Explains elevated caps and fee waivers from the configuration **actually bound to the account**, not the marketing matrix. | The advertised cap is often gated on verification the player has not completed | V1 |
| VIP-10 | Churn Signal Detection | Raises a prioritised host task. **Logs and alerts only — never picks or fires a save offer.** | Churn language and harm language overlap almost completely, and the harm check must win every tie | V1 |
| VIP-11 | Dormant Player Reactivation | Restores context for a returning player and clears the practical blockers. Any welcome-back offer must already be booked, with an **unconditional exclusion re-check first**. | Dormancy is often an elapsed exclusion or a harm episode, not disinterest | V1 · V2 · V3 |
| VIP-12 | Event Invitation & RSVP | Handles hospitality admin — invitations, RSVPs, guests, logistics — and hands anything involving spend to the events team. | High-touch admin that consumes host time and needs no judgement | V2 |
| VIP-13 | Upgrade Request & Affordability Gate | Reports the published qualification basis and registers interest. **Never promotes, never promises, never implies more deposits would secure a tier.** | VIP status legally requires documented affordability checks signed off by a named senior person | V1 |

---

## Module I — PLT · Platform, Technical & Account Admin

**Needs from the operator:** versioned terms and policies, device and app compatibility data, message delivery logs, incident register, consent records.

| ID | Feature | What it does | Why it matters | Phase |
|---|---|---|---|---|
| PLT-01 | Version-Pinned Terms Answering | Answers policy questions using the **version actually in force** — the one the player accepted, or the one live when the disputed event happened. Cites the clause and version. | Quoting today's terms for a three-week-old bet produces a written statement the operator then has to honour | V1 |
| PLT-02 | App & Browser Diagnostics | Gives the specific fix for that device and build. **Suppresses all generic advice while a live incident is running.** | Generic troubleshooting during an outage wastes the player's time and ours | V1 |
| PLT-03 | Email & Notification Deliverability | Investigates "I never got the email" from delivery records. Sits underneath password resets, one-time codes and withdrawal confirmations. | Removes tickets in three other modules, which is why it scores above its own volume | V1 · V2 resubscribe |
| PLT-04 | Outage & Maintenance Broadcaster | Matches incidents to the specific product the player is failing on, so one provider's problem is not reported as a site-wide outage. Gives **only the ETA the incident record contains**. | Inventing an ETA during an outage is the fastest way to turn one complaint into two | V1 · restore notify V3 |
| PLT-05 | Communication Preferences | Updates consent per channel and category and records what wording was consented to. **Refuses to suppress transactional and regulatory messages** and explains why. | Suppressing a withdrawal confirmation or an RG notice is a compliance failure dressed as a preference | V1\* / V2 |
| PLT-06 | Session Timeout Explainer | Tells apart five look-alike causes of "it keeps logging me out". **Read-only** — any loosening goes through the RG path. | Four of the five causes are protective and must not be weakened by a tech-support conversation | V1 |
| PLT-07 | Identifier & Timezone Resolution | Turns whatever the player supplies — a transaction ID, a screenshot timestamp in their local time — into the identifier the target system needs. | **Most casino, sportsbook and bonus procedures cannot start without this.** Players never have the ID we need | V1 |
| PLT-08 | Accessibility Handler | Recognises accessibility needs, adapts its output, saves the preference, and offers other channels where chat itself is the barrier. | Also surfaces to RG as a possible vulnerability signal | V2 |

---

## Module J — CMP · Complaints, Disputes & Service Recovery

*A complaint is a regulated artefact. This module owns it, and owns the only money-out path in the product.*
**Needs from the operator:** complaint register, dispute-body configuration per market, service-recovery matrix, goodwill history.

| ID | Feature | What it does | Why it matters | Phase |
|---|---|---|---|---|
| CMP-01 | Complaint Record Creation | Detects dissatisfaction in **any** module and creates the formal record before the acknowledgement is sent, with the clock starting from the **player's message**. **Talking a player out of it without a record is prohibited.** | A complaint exists the moment dissatisfaction is expressed. Soothing it away produces exactly the pattern regulators prosecute as complaint suppression | V1 |
| CMP-02 | Dispute Body Signposting | Gives the correct free dispute body for the licence the player registered under, proactively at deadlock, **without discouraging use**. | Routes on licensing entity, never country. The same operator's two brands have entirely different routes | V1 |
| CMP-03 | Evidence Pack Assembly | Assembles everything in one artefact — transaction trail, round or bet logs, terms version at the time, prior goodwill — so the human starts with the facts instead of re-interviewing the player. | **The agent's highest-value output in every do-not-automate case.** It cannot decide, but it can do all the work | V1 |
| CMP-04 | Complaint Status Tracking | Answers "where is my complaint" with stage, elapsed and remaining statutory time and the next milestone. Never predicts the outcome. | Reduces the chase-up contacts that make up most complaint-related volume | V1 |
| CMP-05 | Goodwill & Compensation Guardrail | **The single money-out path for the whole product.** Issues re-credits only where an independent record proves an operator-side fault, within hard caps, keyed on the real object so two chat windows cannot claim it twice. | Without one path and one ceiling, compensation drifts by tier and becomes an audit finding | V2 |
| CMP-06 | Remedy Consistency Control | Enforces and evidences that outcomes, goodwill ceilings, RG thresholds and verification gates are **identical regardless of tier or value**. Speed and channel may differ; remedies may not. | Regulators specifically test whether outcomes vary by player value | V1 |

---

# 6. Rules the Product Must Obey

## 6.1 Responsible gambling

Every message is screened for distress and gambling harm **before any other decision** — including on conversations that are closed, handed over or already escalated. A player in crisis does not check the conversation's status before typing.

On a trigger, the agent:

1. Stops generating text immediately
2. Sends only wording the operator has approved for that market
3. Offers self-exclusion or a break, applied in the same conversation
4. Escalates to a trained officer at top priority
5. Gives the correct local support contacts, from maintained configuration

## 6.2 Why protective actions ship in V1

Everything else that changes an account waits for the V2 accuracy gate. Protective actions do not.

**Friction in responsible gambling is deliberately asymmetric.** The correct design is zero friction on protecting yourself and high friction on reversing it. An agent that answers "where's my deposit" in two seconds but puts "I need to stop gambling" in a queue has inverted exactly the asymmetry the law is built on — and that inversion appears in published enforcement settlements.

These actions are safe to ship early because they have **no wrong direction**: restrictive only, one-way, no amount involved, and a trained human is still notified. The action removes the delay before protection lands. It does not remove the human.

The product claim restates from *"the agent cannot change anything"* to **"the agent cannot move money"** — which stays true.

## 6.3 Data handling

- Card numbers, bank details and identity numbers are stripped from the player's message **before the AI sees it** — while leaving ordinary language intact, so "I've lost the rent money" still reaches the harm screen
- Each procedure declares what player data it may touch. Everything else is dropped before it reaches the AI
- The AI receives only reasons that may be disclosed to a player. Internal risk codes never reach it at all
- No model training on player conversations. Each operator's data stays in its own region

---

# 7. What the Agent Must Never Do

This is a scope boundary, not a guideline. Each item must be **structurally impossible** — the capability does not exist for the agent — rather than discouraged by instruction.

## Where disclosure is a criminal offence

| The agent must never | Why |
|---|---|
| Disclose or hint at a sanctions hit or money-laundering investigation, **including through changed tone or slower replies** | Tipping off is a criminal offence with personal liability. The model's instinct to be specific and helpful is exactly the failure mode |
| Explain **why** a large-withdrawal review fired, or name a threshold or timeline | "Your withdrawal exceeded our review threshold" is specific, accurate, helpful — and potentially criminal |
| Explain the type of a restricted lock when the player only asked about login | The innocent question with the dangerous answer |
| Explain why a promotional restriction was applied | Hands the playbook to bonus hunters, and where it sits on a suspicion report it is tipping off |
| Confirm to a third party that an account exists | Breaches confidentiality — but ignoring welfare information they give breaches the licence. Take it, say nothing, escalate |

## Where the licence is at stake

| The agent must never | Why |
|---|---|
| Lift, shorten or promise the end of a self-exclusion | Excluded players present as calm and persuasive and frame it as an admin error. This is a permissions problem, not a tone problem |
| Suggest a new account, a different email, or a sister brand | Facilitating circumvention |
| Add any friction to a protective action — "are you sure?", a shorter alternative, a retention offer | A documented enforcement finding |
| Offer a limit increase as a way to complete a blocked deposit | A documented licence-review finding, not a conversion win |
| Generate any text on a harm hard stop, or send a survey or promo afterwards | Generated empathy is where a regulator finds the operator minimised the disclosure |
| Treat a gambling-motivated closure as administrative | The most frequently cited failure in published settlements |
| Grant any bonus or goodwill to a self-excluded or harm-flagged player | **The single most likely way an AI support agent breaches a licence** — because de-escalation is exactly what it will reach for |
| Promote a player to VIP or imply more deposits would secure a tier | VIP status legally requires documented affordability checks signed off by a named person |
| Present withdrawal reversal in markets where it is banned | "You can cancel this and keep playing" is the most natural helpful sentence in the product and a licence breach in some markets |

## Where money is awarded or taken away

| The agent must never | Why |
|---|---|
| Uphold **or** reverse a voided bonus | The remedy is genuinely disputed and varies by market. The agent assembles facts only |
| Use a malfunction clause to void a win | A legal determination. A defect affecting several players is a reportable incident, not a ticket |
| Confirm a jackpot is payable, or its amount or timing | The provider holds and validates the pot |
| Rule on whether a live dealer erred | Only the provider can adjudicate its own round. Our value is the video hold placed in seconds |
| Defend a voided bet as correct, or hint it may be honoured | Either statement becomes evidence in a dispute referral |
| Reclaim money already credited or withdrawn | Contested territory, and it can drive a balance negative |
| Negotiate a maximum-payout cap | The player learns of the cap after the win of their life |
| Discuss a card dispute's merits, or offer money to withdraw one | Breaches card-scheme rules and damages our own case |
| Refund a deposit | Only unwagered deposits, and only by Finance |
| Offer self-service on an underage account, or mention the balance | The inverse of a normal closure — stakes returned, winnings voided |
| Apply a softer threshold or higher ceiling because the player is VIP | Regulators specifically test for this. It is the finding in the enforcement notice |

## Where security depends on it

| The agent must never | Why |
|---|---|
| Tell a player what evidence would pass a source-of-funds check | Coaching is itself a control failure |
| Edit name, date of birth, national ID, country or currency | Defeats sanctions and self-exclusion matching |
| Send a password reset to an address given in the conversation | The takeover chain is: change the email, then reset the password |
| Send a one-time code to a channel given during the conversation | The factor's entire value is that the channel was registered in advance |
| Give any guidance that helps a player appear to be somewhere permitted | Facilitating unlicensed gambling |
| Change a payout destination without step-up and proof of ownership | An attacker's entire objective |
| Grant an offer on a claimed host promise with no record | Exploits circulate in VIP groups within hours |
| Reveal internal risk scoring or stake limiting | Directly monetisable by arbitrage groups |
| Resolve dissatisfaction without creating the complaint record | Complaint suppression |

---

# 8. Quality Bar

| Area | Requirement |
|---|---|
| **Languages** | 30+, with correct handling of gambling terms — *parlay, rollover, playthrough, cashout, RTP, accumulator, dead-heat*. A mistranslated wagering term is a wrong answer, not a style issue |
| **Peak load** | 10,000+ concurrent chats on a major fixture day, with a defined degradation ladder rather than a collapse. Safety screening has no load exemption |
| **Availability** | 99.99% on the safety path — intake, screening, escalation. 99.9% on the answering path, which degrades to a human |
| **Speed** | First feedback under 1s · first words under 1s at p95 · simple answer ~2s · investigation 5–8s at p95 · hard stop at 15s, then escalate |
| **Audit** | Every decision recorded and committed **before** the message is sent. Records cannot be altered, and any change is detectable |
| **Erasure** | A lawful deletion request removes the personal data while leaving the decision record intact and verifiable |
| **Isolation** | One operator's data is never reachable from another's. Enforced at the data layer, not in application code |

---

# 9. Dependencies

**On the operator.** Live account data through an API; support procedures we can encode; a compliance reviewer who signs off procedures; a named owner for the responsible-gambling queue.

**Capability-gated features.** Every procedure declares what it needs. **Where an operator cannot supply it, that procedure is switched off for them and those questions go to a human.** A typical operator will run roughly 60 of the 117 at launch.

This is a commercial conversation, not a footnote. The tenant dashboard shows exactly which procedures are live and which are dark, and why — so the answer to "what am I paying for" is visible rather than discovered.

**Three hard gates.** An operator who cannot meet these cannot have the corresponding phase:

| Gate | Blocks |
|---|---|
| Cannot distinguish our own sent messages from a human agent's | Everything — we could not tell when a human took over |
| Cannot accept our reference on an outbound action | All V2 actions — we could not verify what happened after a timeout |
| Cannot supply exclusion status at the moment of sending | All V3 outreach |

---

# 10. Release Plan

| Phase | Timing | What ships |
|---|---|---|
| **1 — Foundation & Guardrails** | Months 1–2 | Core chat, harm detection, message scrubbing, complaint records, account reads, and the **protective action set** — limits, breaks, self-exclusion, marketing suppression, withdrawal freeze |
| **2 — Depth & Coverage** | Months 3–4 | Payments depth, bonus and wagering, document upload with regional routing, help-desk integration, step-up verification, identifier resolution |
| **3 — Autonomous Operations** | Months 5–6 | V2 actions behind the accuracy gate: goodwill engine, bonus grants, payment-method management, limit increases. Draft suggestions for human agents |
| **4 — Deep Vertical** | Months 7–9 | Sportsbook module, casino round recovery and dispute intake, VIP routing and host handoff |
| **5 — Proactive & Voice** | Months 10–12 | Proactive status updates, expiry countdowns, reactivation — all behind the send-time RG check. Voice |

**Phase 3 does not start on a date.** It starts when Phases 1 and 2 have shown a sustained wrong-answer rate under 0.5%, zero generated text on a prohibited topic, zero cases of the agent talking over a human, and an audit trail that has survived a real compliance review.

Actions are where a wrong answer becomes a wrong transaction. The accuracy bar is the gate, not the calendar.

---

# 11. Open Decisions

| # | Decision | Recommendation |
|---|---|---|
| 1 | Ship protective RG actions in V1? | **Yes.** Otherwise the protective path is slower than the depositing path. Needs Compliance sign-off, not just product |
| 2 | What resolution rate do we contract? | **Publish two.** 80–90% on the eligible denominator in V2; no more than ~55% on the contained one. Someone must tell the customer a correct escalation is a success |
| 3 | Who tells the customer sub-2-second responses are not achievable with mandatory screening? | **Publish the table in §8.** Moving screening off the critical path destroys the control the product is built on |
| 4 | Build 99.99% end to end? | **No — split it.** Four-nines on safety, three-nines on answering. End-to-end fights data residency and roughly doubles infrastructure cost |
| 5 | Who owns the sports and game rules? | **The operator supplies them; we index and version them.** Owning them makes us liable for settlement accuracy we cannot verify |
| 6 | Sportsbook or payments depth first? | **Cross-cutting safety first (6–8 weeks), then payments** unless the first customer is sports-led |
| 7 | Does the agent move money in V2, and at what ceiling? | **Yes, under CMP-05's guardrails.** The ceiling and the budget line are not product's to set alone |
| 8 | Build document upload in V1? | **Yes, but per-market.** Largest deflection in its module, but a compliant multi-region pipeline is a project in its own right |

---

# Appendix A — Ten Things to Know About This Industry

For anyone joining without an iGaming background.

1. **Protective friction is deliberately backwards.** Zero friction on protecting yourself; high friction on undoing it. "Are you sure?" on a self-exclusion is an enforcement finding.
2. **Tipping off is criminal, and helpfulness is the failure mode.** The dangerous sentences are the good ones.
3. **A complaint exists the moment someone is unhappy** — not when they fill in a form. Resolving it in conversation without a record is complaint suppression.
4. **The contract freezes at the event, not the question.** A bet's terms are fixed when it is placed. Always returning *current* terms will quote today's offer for a three-week-old bet.
5. **"Balance" is at least five numbers.** Cash, bonus, locked cash, money in open bets, pending payouts. Two of them cannot answer the biggest question in payments.
6. **Wagering is turnover, and the basis doubles the number.** "40× bonus" and "40× deposit+bonus" differ by exactly 2× in real money. Max-bet breaches are caught retroactively at withdrawal, which is why those tickets arrive furious.
7. **Jurisdiction means the licence, not the country.** The same operator's two brands have different verification timing, complaint routes and product availability for players in the same city.
8. **Never let the AI generate a rule, an RTP or a settlement calculation.** The same slot ships in several certified RTP variants. A dead heat divides the *stake* and pays full odds — it does not halve the odds.
9. **Game state and wallet state routinely disagree**, and two chat windows will farm your refund path if the deduplication key is wrong.
10. **VIP is a regulated category, and the churn signal is the harm signal.** Chasing losses scores as both. "Big loss, send cashback" is the precise pattern that produces enforcement action.

---

# Appendix B — System Dependency Map

*Engineering reference. Not part of the functional specification.*

Indicative endpoints assumed by each module. Real paths come from each operator's platform during integration; these exist to size the work and to test whether a prospective operator can support a module at all.

| Module | Core dependencies |
|---|---|
| **PAY** | Deposit and withdrawal records · payout stage timeline · payment-provider decline and authorisation data · payment-method configuration by country and licence · player-level payout limits · currency detail stamped per transaction · crypto confirmations |
| **BON** | Active bonuses and wagering ledger · campaign terms with version history · game weighting table · free-round grants and provider allocation state · cashback statements · tournament rules and standings |
| **SPRT** | Bet and leg records · settlement audit trail · versioned market and sport rules · cash-out request log · event results and revisions · terms snapshot per bet · payout cap clause |
| **CAS** | Provider round logs · wallet transaction ledger · unfinished-round policy per provider · game catalogue and certification by market · live table status · paytable by jurisdiction · incident register |
| **KYC** | Verification partner case state · disclosable rejection reasons · accepted-document matrix by licence · due-diligence case and evidence requests · payment-instrument ownership · linked-account and exclusion-registry match |
| **SEC** | Authentication attempt log · lock ledger split by disclosability · one-time-code gateway delivery records · location-check provider results · active session and device list · profile change history |
| **RG** | Limits with consumption and reset · pending limit changes · market rules table · exclusion state including national schemes · harm markers · approved wording and support organisations by market · activity summary |
| **VIP** | Assigned tier and value band · loyalty ledger and tier progress · host ownership and availability · booked offer entitlements and envelopes · tier history · churn score |
| **PLT** | Versioned terms with acceptance records · device and app compatibility matrix · message delivery and suppression records · incident and maintenance register · consent history · identifier and timezone resolution |
| **CMP** | Complaint register with statutory clock · dispute-body configuration by licence · service-recovery matrix · goodwill history · adjustment ledger |

**Cross-cutting:** every module needs player profile with licensing entity, live responsible-gambling status, and the lock ledger. The exclusion check and the harm screen run on every conversation regardless of module.
