# PRD: CommCheck — Pre-Visit Commercial Intelligence for Field Reps
**Version:** 2.0  
**Author:** Namratha Rudrappa, Senior Product Manager  
**Status:** SHIPPED — live module inside FieldIQ, eval harness run  
**Architecture:** [`architecture.md`](architecture.md) · **Evals:** [fieldiq-evals/commcheck](https://github.com/namratharuds-stack/fieldiq-evals/tree/main/commcheck)

---

## 1. Problem Statement

Field sales reps in CPG companies visit 15–20 outlets daily. Before each visit, they have access to route plans, shelf compliance scores, and SKU order history — all surfaced by SFA tools like BeatRoute, FieldAssist, and Salesforce Consumer Goods Cloud.

What no tool surfaces: **the commercial and contractual obligations that are at risk at each outlet before the rep walks in.**

Contract volume commitments, co-op fund deadlines, payment term breaches, SKU listing requirements — this data lives in CLM systems (Vistex, Icertis, Conga) or finance tools accessible only to commercial managers and legal teams at HQ. It never reaches the field rep.

The result: reps discover commercial landmines mid-visit, with no plan and no authority to resolve them. By the time the signal escalates to a manager, the quarter is already at risk.

**The gap is architectural.** CLM tools stop at the commercial manager. SFA tools stop at SKU recommendations. No tool has built the translation layer between them — contract obligation data, converted into a plain-language briefing a field rep reads in 90 seconds before entering an outlet.

---

## 2. User Definition

**Primary User:** Field Sales Representative at a CPG company  
- Manages 15–20 outlet visits per day across General Trade, Modern Trade, or Key Accounts  
- Uses a mobile SFA app for route planning, order capture, and visit reporting  
- Has no access to CLM or finance systems  
- Needs pre-visit context in under 2 minutes, on mobile, often with poor connectivity  

**Secondary User:** Field Sales Manager  
- Manages a team of 8–15 reps  
- Currently discovers commercial issues in weekly review meetings — too late to act  
- Needs a structured escalation signal from reps, not informal WhatsApp messages  

**Out of scope for v1:** KAMs, commercial managers, legal teams — they already have CLM dashboards.

---

## 3. Jobs to Be Done

| Job | Current Solution | Gap |
|---|---|---|
| Know which outlet to visit today | BeatRoute / FieldAssist routing | Solved |
| Know what SKUs to push | Order AI Agent (BeatRoute) | Solved |
| Know shelf compliance status | Trax / Store360 | Solved |
| Know what commercial obligation is at risk at this outlet | Nothing | **Unsolved** |
| Know what to escalate to manager before it's too late | Nothing structured | **Unsolved** |

---

## 4. Solution Overview

**CommCheck** is a pre-visit commercial intelligence tool for field reps.

Before leaving the depot, a rep enters their planned outlets for the day. CommCheck synthesizes what they know about each outlet's commercial standing and generates three output cards per outlet:

**Card 1 — Act Today (Red)**  
Outlets with an active commercial risk the rep can influence in the visit. Specific action attached.

**Card 2 — Opportunity (Green)**  
Outlets where a commercial condition creates a revenue opening today — promo not activated, SKU listing gap, rebate tier within reach.

**Card 3 — Escalate (Amber)**  
Outlets where the issue is above rep authority — contract volume breach, payment terms exceeded, loyalty tier drop imminent. Pre-written escalation message ready to send to manager.

---

## 5. User Flow

```
MORNING (before leaving depot)
        │
        ▼
Rep opens CommCheck → enters planned outlets for the day
(5 fields per outlet, ~90 seconds per outlet)
        │
        ▼
AI synthesizes signals → generates 3 cards per outlet
        │
        ▼
Rep reviews cards → adjusts visit priority if needed
        │
        ▼
Rep visits outlets with context → acts on card guidance
        │
        ▼
Post-visit: Rep marks escalations as "Sent" or "Resolved"
```

---

## 6. Input Fields (Per Outlet)

Designed for mobile-first, minimal typing, works offline:

| Field | Input Type | Why It Matters |
|---|---|---|
| Outlet name & tier | Text + dropdown (Gold/Silver/Bronze/General) | Drives priority weighting |
| Days since last visit | Number | Flags coverage gaps |
| Last order vs typical | Dropdown (Higher / Same / Lower / No recent order) | Sales trajectory signal |
| Current promo activated? | Yes / No / Don't know | Revenue opportunity trigger |
| Contract target on track? | Yes / Behind / Don't know | Commercial risk trigger |
| Any open issue from last visit? | Free text (optional) | Context for AI reasoning |
| Payment status | Dropdown (Current / Overdue / Unknown) | Escalation trigger |

**Total input time per outlet: ~90 seconds**

---

## 7. Output Design

### Card 1 — Act Today (Red border)
```
🔴 ACT TODAY — [Outlet Name]
Risk: Contract volume 18% behind with 3 weeks left in quarter
What to do: Lead with Q3 volume bridge. Offer to review 
forward-buy options before end of month. Don't take new 
order without addressing the gap.
```

### Card 2 — Opportunity (Green border)
```
🟢 OPPORTUNITY — [Outlet Name]  
Signal: Promo launched 2 weeks ago — not yet activated at 
this outlet. Competitor display confirmed nearby.
What to do: Bring promotional materials. Lead with competitor 
context. This outlet is Gold tier — activation here counts 
toward co-op fund eligibility.
```

### Card 3 — Escalate (Amber border)
```
🟡 ESCALATE — [Outlet Name]
Issue: Payment terms exceeded by 14 days. New order intake 
not recommended until resolved.
Send to manager: "[Outlet name] has an overdue payment of 
14 days. Holding new order pending resolution. Please advise 
before end of day."
[Copy message] button
```

---

## 8. AI Design Decisions & Tradeoffs

### Decision 1 — Three cards, not one ranked list
A single ranked list mixes action types and creates noise. A rep's response to "act today" is fundamentally different from their response to "escalate." Separating them reduces cognitive load and maps directly to what the rep can and cannot do in the visit.

**Tradeoff:** More complex UI. Accepted because clarity of action matters more than simplicity of display for a user with 90 seconds to prepare.

### Decision 2 — Plain language output, not scores
Every existing tool outputs a number or a score. Reps don't act on scores — they act on language they can use in a conversation with an outlet buyer. CommCheck outputs sentences, not percentages.

**Tradeoff:** Harder to build consistent quality in AI output. Mitigated by structured prompting with domain-specific CPG context in the system prompt — and, after the eval, by structural format enforcement rather than prompt instruction alone.

### Decision 3 — Pre-written escalation message
The escalation card includes a one-tap copyable message to send to the manager. This removes the rep's biggest friction point — knowing what to say and how to say it — and increases the likelihood that issues actually get escalated instead of sitting in the rep's head.

**Tradeoff:** Message may not perfectly fit every situation. Solved by making it editable before sending.

### Decision 4 — Manual input, no API integration
v1 deliberately avoids API connections to CLM or SFA systems. Reps already know this information — last order size, days since last visit, whether a promo is activated. Requiring them to enter it removes the dependency on enterprise data quality and proves the concept on decision logic alone.

**Tradeoff:** Data entry burden. Mitigated by mobile-first form design and progressive autofill in v2.

### Decision 5 — Morning timing, not real-time alerts
CommCheck is a start-of-day tool, not a live feed. Reps need a plan before they leave, not interruptions while driving between outlets. Timing of the intervention matters as much as the intervention itself.

**Tradeoff:** Misses intra-day changes. Accepted for v1; real-time push notifications are a v2 feature.

### Decision 6 — Format enforced structurally, not instructed

The system prompt originally ended with "Return ONLY valid JSON. No preamble,
no markdown, no explanation." The eval harness measured **0% compliance** with
that instruction across 19 cases. Every response wrapped the JSON in something.

The fix was not a better-worded prohibition. It was `response_format:
json_object` at the gateway, plus defensive `indexOf` extraction as a second
line of defense before parse.

**The generalizable lesson:** a prohibition in a prompt functions as a request.
If a behavior has to hold, it needs a mechanism, not a sentence.

**Tradeoff:** couples the feature to providers that support constrained
decoding. Accepted — the gateway abstracts the provider, and any provider
without it is not a candidate for this workload.

### Decision 7 — Shortfall and breach are distinct terms

The eval found one rule failure that traced to the spec, not the model: the
prompt used "behind target" and "contract breach" interchangeably, so the model
routed some breaches to act_today cards. These are commercially different
events with different owners — a shortfall is the rep's to influence, a breach
is above their authority.

**The generalizable lesson:** spec ambiguity degrades output silently. The
model does not raise a question; it picks an interpretation and proceeds.

---

## 8b. Architecture

| Layer | Decision |
|---|---|
| Call path | Client → TanStack Start server function → model-agnostic AI gateway → provider |
| Model | Gemini 2.5 Flash — latency and cost on mobile; a gateway config value, not a code dependency |
| Format | `response_format: json_object` + defensive `indexOf` extraction |
| Failure | Typed envelope `{ ok: true, data }` / `{ ok: false, error }` — UI branches on `ok` |
| Tools | Read-only MCP server, five tools over the territory dataset |
| Not used | No agent framework, no vector DB, no embeddings — single-shot task over structured data |
| Removed in v2 | Client-side provider calls, `x-api-key` in browser headers, session API key modal |

---

## 8c. Eval Harness — [`fieldiq-evals`](https://github.com/namratharuds-stack/fieldiq-evals)

**19 cases across three layers:** schema validity, rule adherence, adversarial
/ prompt injection.

| # | Finding | Lesson |
|---|---|---|
| 1 | **0% compliance** with the "JSON only" instruction | Prohibitions are requests. Enforcement needs a mechanism. |
| 2 | Silent rule extension — tier rules applied to outlet types the spec never covered, unflagged | Models fill spec gaps quietly instead of surfacing them |
| 3 | Rule failure traced to spec ambiguity (shortfall vs. breach) | Precision in the spec, not just the model, drives correctness |

Both structural findings shipped as fixes: format enforcement at the gateway,
and disambiguated definitions plus a "state the gap, don't fill it" instruction
in the system prompt.

---

## 9. Success Metrics

| Metric | Target | How Measured |
|---|---|---|
| Escalation rate | >30% of red/amber cards result in manager escalation | Message "Sent" vs total amber cards |
| Visit outcome improvement | Rep reports "card was useful" >70% of time | Post-visit 1-tap feedback |
| Time to complete input | <3 minutes for 5 outlets | Session timing |
| Escalation resolution time | Manager responds within 4 hours >80% of time | Timestamp tracking |

---

## 10. What This Is NOT

- Not a CLM replacement — CommCheck translates CLM data, it doesn't manage contracts
- Not a route planner — it works alongside BeatRoute/FieldAssist, not instead of them
- Not a shelf audit tool — Trax/Store360 own that layer
- Not a manager dashboard — the manager view is v2

---

## 11. v2 Roadmap

| Feature | Rationale |
|---|---|
| API integration with Salesforce CG Cloud | Auto-populate visit data, remove manual input |
| CLM data connector (Vistex / Icertis) | Pull live contract obligations, remove manual entry |
| Manager dashboard | Portfolio view of all rep escalations by territory |
| Push notifications | Intra-day alerts when contract milestone is breached |
| Voice input | Rep dictates outlet update while driving |

---

## 12. Connection to Prior Work

This prototype is not a side project. It is a productization of a problem I encountered repeatedly across:

- **22-market sales execution platform:** Reps had no visibility into contract standing at outlet level. Issues were discovered in monthly commercial reviews, quarters after the opportunity to intervene had passed.

- **B2B Commerce & Loyalty portal (Salesforce Commerce Cloud):** Distributor tier drops and co-op fund eligibility changes were managed by commercial teams with zero signal reaching the field reps managing those relationships daily.

CommCheck is the bridge I would have built during that work if we had prioritized field-level commercial intelligence. It is now a portfolio artifact demonstrating the product thinking that came from building inside that problem for 7.5 years.

---

---

*Built by [Namratha Rudrappa](https://namratharudrappa.com) · Senior Product Manager · CPG Commercial Execution & AI*
