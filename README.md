# FieldIQ Product Specs

Product specs behind [FieldIQ](https://fieldiq-pro-pilot.lovable.app), an AI field-selling platform for CPG sales reps. Each spec connects to its eval, so you can follow a feature from problem → decision → eval finding → fix.

| Doc | What it covers |
|---|---|
| [`commcheck-prd.md`](commcheck-prd.md) | **CommCheck PRD.** Pre-visit commercial intelligence, with seven AI design decisions and their tradeoffs, success metrics, and scope boundaries |
| [`draft-order-agent-spec.md`](draft-order-agent-spec.md) | **Draft Order Agent spec.** The order the AI prepares, reason codes, the system prompt and JSON contract, UI states, outlet-specific behavior, and the eval criticality model |
| [`architecture.md`](architecture.md) | **Architecture and evals.** Server-side gateway, structured output enforcement, typed failure envelope, what was deliberately *not* built, and the agent inventory |

## Decisions worth reading first

- **Three cards, not one ranked list** (CommCheck, Decision 1). *Act today*, *opportunity*, and *escalate* need different responses from the rep, so they shouldn't compete in one list.
- **Format enforced structurally, not instructed** (CommCheck, Decision 6). An eval measured 0% compliance with "return ONLY valid JSON." The fix was `response_format: json_object` at the gateway, not a better-worded prompt.
- **Shortfall and breach are different events** (CommCheck, Decision 7). One eval failure traced back to my own spec: it treated a gap the rep can act on and a breach above the rep's authority as the same thing.
- **A fabricated history is a critical failure** (Draft Order spec). If a rep tells a buyer "like we discussed" about something never flagged, that does more damage to credibility than a missed recommendation.
- **No model where rules will do** (Architecture). OnboardIQ is a four-stage state machine with no LLM, the same call as choosing rules-based route planning over ML on a 22-market platform.
- **No agent framework, no vector DB** (Architecture). Every agent is a single-shot task over typed data, so retrieval is a filter, not a similarity search.

## Related

- [fieldiq-evals](https://github.com/namratharuds-stack/fieldiq-evals): the eval harnesses these specs reference
- [cpg-whitespace-map](https://github.com/namratharuds-stack/cpg-whitespace-map): the 12 gaps these features came from

## Notes

- All outlets, SKUs, prices, and buyer names are FieldIQ demo data.
- ContractRadar and PromoPostMortem have PRDs that aren't included here.

---

[Namratha Rudrappa](https://namratharudrappa.com) · Senior PM, Applied AI for CPG commercial execution
