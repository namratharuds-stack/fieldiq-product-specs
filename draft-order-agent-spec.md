# FieldIQ — Draft Order Agent: Product Spec
**Author:** Namratha Rudrappa · **Status:** Shipped in [FieldIQ](https://fieldiq-pro-pilot.lovable.app)

> Architecture: [`architecture.md`](architecture.md) · Evals: [fieldiq-evals/draft-order](https://github.com/namratharuds-stack/fieldiq-evals/tree/main/draft-order)

The Draft Order Agent prepares a recommended order for the rep before they walk into an outlet, based on contract gaps, order history, SKU whitespace, promotional requirements, and open Loss Loop items. It lives as the **Order Agent** tab on the outlet detail screen.

All outlets, SKUs, prices, and buyer names in this spec are FieldIQ demo data.

---

## WHAT THE DRAFT ORDER AGENT DOES

This is RGM in action, not RGM as a separate screen. The agent's reasoning
chips — `contract_gap`, `promo_requirement`, `whitespace_opportunity`,
`velocity_reorder`, and now `loss_loop_recovery` — are revenue-growth-management
decision rules, surfaced at the exact moment a rep can act on one, instead of
sitting in a category-management deck. 

The agent analyzes everything known about the outlet — contract standing,
order history, SKU whitespace, promo requirements, **and any opportunity Loss
Loop flagged on a prior visit that was never closed** — and generates:

1. **A recommended draft order** — specific SKUs, quantities, and case prices
2. **The reasoning behind each line item** — why this SKU, why this quantity
3. **Contract impact** — what this order does to MTD achievement and contract target
4. **Loss Loop closure** — line items that specifically resolve a previously flagged, still-open opportunity, marked distinctly from a fresh recommendation
5. **Rep edit mode** — rep can adjust quantities before submitting
6. **One-tap approval** — "Submit Draft Order" button

The rep walks into the outlet with a pre-built order. Their job shifts from calculating what to order to reviewing what the AI prepared and having the commercial conversation. An order that quietly closes a three-week-old flagged opportunity is worth more to the rep than one that doesn't say so.

---

## VISUAL DESIGN

Match existing FieldIQ dark design system exactly:
- Background: #080C18
- Card surface: #111827
- Elevated: #1C2540
- Primary accent: #3B82F6
- Success: #10B981
- Warning: #F59E0B
- Danger: #EF4444
- Text primary: #F9FAFB
- Text secondary: #9CA3AF

**New design element for this tab only:**
The draft order lines use a "reasoning chip" pattern — each SKU line has a small colored pill showing WHY it was included:
- 🔴 "Contract gap" — red pill
- 🟡 "Promo requirement" — amber pill  
- 🟢 "Whitespace opportunity" — green pill
- 🔵 "Velocity reorder" — blue pill
- 🟣 "Loss Loop recovery" — purple pill (`#8B5CF6`) — this line specifically closes an opportunity Loss Loop flagged on a prior visit and that never got acted on

This makes the AI reasoning visible at a glance — rep understands not just WHAT to order but WHY. The purple chip carries extra weight: it tells the rep this isn't a new idea, it's something already promised or already known about the account, finally getting closed.

---

## TAB CONTENT — THREE STATES

### State 1 — Pre-Generation (default on tab open)

```
Header card (blue left border):
┌─────────────────────────────────────────┐
│ ⚡ DRAFT ORDER AGENT                     │
│ AI-prepared order based on:             │
│ • Contract gap vs target                │
│ • 4-week order trend                    │
│ • Active promotional requirements       │
│ • SKU whitespace opportunities          │
│                                         │
│ [Generate Draft Order →]  ← green btn  │
│                                         │
│ Outlet context shown below:             │
│ MTD: $X,XXX / $X,XXX (XX%)            │
│ Last order: $X,XXX (X days ago)        │
│ Contract status: [On Track / Behind]   │
│ Promo activated: [Yes / No]            │
└─────────────────────────────────────────┘
```

---

### State 2 — Loading

```
Center of tab area:
⚡ (pulsing blue icon)
"Analyzing outlet signals..."
"Calculating contract gap..."
"Identifying SKU opportunities..."
(cycling through these 3 messages, 800ms each)
```

---

### State 3 — Draft Order Generated

```
┌─────────────────────────────────────────┐
│ DRAFT ORDER — [Outlet Name]             │
│ Prepared [today's date] · [X] items    │
│                                         │
│ CONTRACT IMPACT                         │
│ Before this order: XX% of target       │
│ After this order:  XX% of target       │
│ [progress bar showing improvement]     │
└─────────────────────────────────────────┘

ORDER LINES (one card per SKU):

┌─────────────────────────────────────────┐
│ Choc Fudge Bar 12pk          $24.99/cs │
│ [🔴 Contract gap]                       │
│                                         │
│ Reasoning: "48 cases needed to close   │
│ contract gap. Current MTD: 24 cases.   │
│ Target: 48. Ordering 24 cases brings   │
│ you to 100% on this SKU."              │
│                                         │
│ Qty: [−] [24] [+]    Total: $599.76   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Vanilla Almond Bar 12pk      $24.99/cs │
│ [🟢 Whitespace opportunity]             │
│                                         │
│ Reasoning: "Not yet listed. Top-3 SKU  │
│ at comparable Walmart accounts in NJ.  │
│ Trial order of 12 cases recommended." │
│                                         │
│ Qty: [−] [12] [+]    Total: $299.88   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ BBQ Chips 8pk                $14.99/cs │
│ [🔵 Velocity reorder]                   │
│                                         │
│ Reasoning: "Consistent weekly velocity │
│ of 6 cases. Reorder at standard rate  │
│ to maintain shelf availability."       │
│                                         │
│ Qty: [−] [6] [+]    Total: $89.94    │
└─────────────────────────────────────────┘

─────────────────────────────────────────
ORDER SUMMARY
Items: X SKUs · X cases total
Subtotal: $X,XXX.XX

CONTRACT ACHIEVEMENT AFTER THIS ORDER:
[████████████░░] 94% of target
+24pts improvement vs current

─────────────────────────────────────────

[✏️ Edit quantities above before submitting]

[⚡ Submit Draft Order]  ← primary green btn, full width

[↺ Regenerate]  ← ghost button, smaller
```

---

## MODEL INTEGRATION — SERVER-SIDE GATEWAY

Call path: client → TanStack Start server function → model-agnostic AI gateway
→ provider. Current model **Gemini 2.5 Flash**. No provider endpoint, key, or
header is reachable from the browser.

- `response_format: json_object` enforced at the gateway, with defensive
  `indexOf` extraction before parse
- Returns a typed envelope: `{ ok: true, data }` / `{ ok: false, error }`
- Single-shot task over structured outlet data — no framework, no retrieval layer

**System prompt:**
```
You are the FieldIQ Draft Order Agent — an AI that prepares recommended orders for CPG field sales reps before they visit an outlet.

Your job is to analyze everything known about an outlet and generate a specific, justified draft order that:
1. Closes contract volume gaps before quarter end
2. Maintains velocity on active SKUs
3. Activates promotional requirements
4. Introduces whitespace SKUs where appropriate
5. Closes any opportunity Loss Loop flagged on a prior visit that is still open

You understand CPG commercial execution deeply:
- Contract gaps should be prioritized — if an outlet is behind target, the order should close the gap
- Velocity reorders maintain shelf availability — reorder at the average weekly rate
- Whitespace SKUs are eligible but unlisted SKUs — recommend trial quantities (half normal velocity)
- Promotional SKUs that are not yet activated must be included if a promo is running
- If a Loss Loop item is open for this outlet, and a SKU in activeSkus or eligibleSkus would close it, prioritize that line ahead of a same-category fresh recommendation — a promise kept outranks a new idea
- Never recommend ordering a SKU that is not in activeSkus or eligibleSkus
- Never recommend a quantity below the SKU's minimum order quantity (MOQ)
- If the outlet's payment status is overdue, the order must carry an escalation
  flag. Do not generate a clean order against an overdue account.
- If a rule does not cover the situation you are given, state that in the
  reasoning rather than extending an adjacent rule to fit
- Be specific and commercial in reasoning — use numbers, reference the contract, reference comparable accounts

Return ONLY valid JSON with this exact structure:
{
  "outlet_name": "string",
  "order_rationale": "1 sentence explaining the overall order strategy",
  "lines": [
    {
      "sku_id": "string",
      "sku_name": "string",
      "quantity": number,
      "unit_price": number,
      "line_total": number,
      "reason_type": "contract_gap" | "velocity_reorder" | "promo_requirement" | "whitespace_opportunity" | "loss_loop_recovery",
      "reasoning": "1-2 sentences of specific reasoning for this SKU and quantity — use numbers, reference contract or trend data",
      "loss_loop_ref": "string — only present when reason_type is loss_loop_recovery: the specific prior signal this closes (e.g. which commCheckCard or open issue), null otherwise"
    }
  ],
  "total_value": number,
  "mtd_before": number,
  "mtd_after": number,
  "target": number,
  "achievement_before": number,
  "achievement_after": number,
  "rep_talking_point": "One specific thing the rep should say when presenting this order to the buyer — commercial, specific, not generic"
}

Rules:
- Only include SKUs from activeSkus (reorder) or eligibleSkus (new listing)
- Contract gap SKUs: calculate quantity needed to close the gap to target
- Velocity reorder SKUs: use average of last 4 weeks as quantity
- Whitespace SKUs: recommend 50% of typical velocity at comparable accounts
- Promo SKUs not yet activated: always include with minimum activation quantity
- Loss Loop recovery SKUs: only use this reason_type when the line traces to an
  actual prior signal passed in the input (a commCheckCard or openIssue) — never
  fabricate a prior flag to justify a recommendation
- reason_type must be an exact match to one of the five enumerated values —
  no synonyms, no new categories
- Return only the JSON object described above.
```

**User message format — pass ALL outlet data:**
```
Generate a draft order for this outlet:

Outlet: [name]
Tier: [tier]
MTD Sales: $[salesMTD]
MTD Target: $[salesTarget]
Achievement: [salesAchievement]%
Last order value: $[lastOrderValue]
Typical order value: $[typicalOrderValue]
Promo activated: [promoActivated]
Contract on track: [contractOnTrack]
Payment status: [paymentStatus]
Days since last visit: [lastVisit]
Open issue: [openIssue or "None"]

Active SKUs (currently ordering):
[For each in activeSkus: sku_id, name, last 4 weeks ordered quantities from skuBreakdown, price]

Eligible SKUs (listed but not ordering):
[For each in eligibleSkus: sku_id, name, price from SKU_CATALOG]

Order history last 4 weeks:
[orderHistory mapped as week: value]

SKU breakdown (ordered vs target):
[skuBreakdown mapped as name: ordered/target, gap, price]

Open Loss Loop items for this outlet (previously flagged, not yet closed):
[commCheckCards and openIssue mapped as: type, signal, date/visit context — or "None"]
```

---

## REP TALKING POINT CARD

After the order lines and before the Submit button, show a special card:

```
┌─────────────────────────────────────────┐
│ 💬 YOUR OPENING LINE                    │
│                                         │
│ "[rep_talking_point from AI response]" │
│                                         │
│ Use this when presenting the order     │
│ to [buyer name].                       │
└─────────────────────────────────────────┘
```

This is the most differentiated element in the tab. The AI doesn't just tell the rep WHAT to order — it tells them WHAT TO SAY when they present it. That's a genuine behavior change enabler.

---

## INTERACTION DETAILS

**Quantity editing:**
- [−] and [+] buttons adjust quantity in increments of 6 (standard case pack)
- Minimum quantity: 0 (rep can remove a line)
- When quantity changes, recalculate line total and order summary in real time
- When quantity changes on a contract gap SKU, update the contract achievement bar in real time

**Submit button behavior:**
- Shows a confirmation state: "Order submitted ✓" with a green checkmark
- Order summary shown: "X SKUs · X cases · $X,XXX submitted"
- Small text: "This order has been logged in FieldIQ"
- Does NOT actually submit anywhere (prototype — visual confirmation only)

**Regenerate button:**
- Clears the current draft and calls the API again
- Useful if rep wants to see a different approach

**Error state:**
- If API call fails: "Unable to generate draft order. Check your connection and try again."
- Show the outlet context card with a manual "Try again" button

---

## OUTLET-SPECIFIC BEHAVIOR

The tab should behave differently based on outlet status:

**act-now outlets:** Lead with contract gap SKUs. Order rationale emphasizes urgency. Achievement bar shown in red before, green after.

**escalate outlets (payment overdue):** Show a warning banner at the top of the tab: "⚠️ Payment overdue — confirm with manager before submitting this order." Generate order anyway but flag it. **The escalation flag on an overdue account is an always-critical eval assertion — an order generated without it fails the gate regardless of every other score.**

**opportunity outlets:** Lead with whitespace SKUs. Rationale emphasizes growth opportunity.

**on-track outlets:** Standard velocity reorder. Simple, no urgency language.

**onboarding outlets (Trader Joe's):** Show a different state: "⏳ Account not yet active — draft order will be available once catalog setup is complete." No generate button shown.

---

## SAMPLE EXPECTED OUTPUT — Walmart Route 9 Woodbridge

When testing with Walmart Route 9, the agent should generate approximately:

Lines:
1. Choc Fudge Bar 12pk — 24 cases — Contract gap — $599.76
2. Peanut Butter Bar 12pk — 18 cases — Contract gap — $449.82  
3. BBQ Chips 8pk — 12 cases — Contract gap — $179.88
4. Sour Cream Chips 8pk — 18 cases — Contract gap — $269.82
5. Vanilla Almond Bar 12pk — 12 cases — **Loss Loop recovery** — $299.88
6. Ranch Chips 8pk — 6 cases — **Loss Loop recovery** — $89.94

Total: ~$1,889.10
Achievement before: 70% → Achievement after: ~86%

Lines 5 and 6 recategorized from a plain whitespace pitch: this outlet's own
`commCheckCards` already flagged both SKUs as an unclosed opportunity —
*"Vanilla Almond Bar and Ranch Chips not yet listed at this location... Pitch a
4-week trial listing."* Nobody acted on it. The agent traces that signal
(`loss_loop_ref`) and marks the line purple instead of green — the rep sees
this isn't a new idea, it's one that's been sitting open.

Rep talking point: "Mike, I've looked at your contract gap and put together an order that gets you to 86% of target — all we need is 24 cases of Choc Fudge and 18 of PB to close most of it. I've also finally added the Vanilla Almond trial we talked about a few visits back — it's moving well at your Route 1 Edison store."

---

## EVAL HARNESS — [`fieldiq-evals/draft-order`](https://github.com/namratharuds-stack/fieldiq-evals/tree/main/draft-order)

**26 cases across 13 categories**, gated at 22/26 — documented, already run.
It validates the grader, not just the model.

> **Not yet run:** the `loss_loop_recovery` reason type and its traceability
> assertion below are additions to this spec, not yet covered by the harness.
> Before this counts as tested, the 26-case suite needs new cases covering it — at minimum, one seeded fixture with a fabricated
> `loss_loop_ref` to prove the new critical-failure check actually catches
> what it's built to catch, the same discipline the existing suite already
> holds itself to.

### Assertion design

| Assertion type | What it checks |
|---|---|
| Quantity bands | Per-case min/max — a recommendation outside the band fails, exact-match is not required |
| Reason codes | Exact match against the five enumerated `reason_type` values. No synonyms. |
| Loss Loop traceability | Every `loss_loop_recovery` line's `loss_loop_ref` must trace to an actual prior `commCheckCard` or `openIssue` in the input — a fabricated reference is a failure, not just an inaccuracy |
| SKU authorization | Every line must appear in `activeSkus` or `eligibleSkus` |
| MOQ | No line below the SKU minimum |
| Escalation presence | Overdue-payment outlets must carry the flag |
| Arithmetic | `line_total`, `total_value`, `achievement_after` must reconcile |

### Criticality model

Four failures are **always critical** — any one of them fails the gate
regardless of the aggregate score:

1. Unauthorized SKU recommended
2. MOQ violation
3. Missing escalation on an overdue account
4. Fabricated `loss_loop_recovery` reference — a rep telling a buyer "like we discussed" about something that was never actually flagged is a worse credibility failure than a missed recommendation

Everything else is scored. This mirrors how the failures actually behave
commercially: a wrong quantity is a conversation, an unauthorized SKU is a
credibility loss in front of a buyer.

### Seeded fixture

A deliberately broken agent reproducing **four documented failure modes** was
run through the grader. The harness had to catch all four. This is the point of
the fixture — it proves the *test* works, independent of how the model scores.

### Gate result

**Deliberate FAIL at 22/26**, with the critical failures correctly caught.

> "Anyone can post a green board if the test can't go red. Mine can, and did,
> on exactly the cases it was built to catch. The FAIL is the deliverable."

---

*[Namratha Rudrappa](https://namratharudrappa.com) · Senior Product Manager · CPG Commercial Execution & AI*
