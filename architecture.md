# FieldIQ — Architecture & Evals
**Author:** Namratha Rudrappa | Senior Product Manager | CPG Commercial Execution & AI
**Version:** 3.0 — current deployed state
**Live:** https://fieldiq-pro-pilot.lovable.app

---

## 1. DEPLOYED ARCHITECTURE

| Layer | Decision | Why |
|---|---|---|
| **Client** | React SPA, mobile-first, max-width 430px | Rep is on a phone in a parking lot at 7am |
| **Server** | TanStack Start server functions | All model calls execute server-side — no key ever reaches the browser |
| **Model access** | Model-agnostic AI gateway | Provider is a config value, not a code dependency |
| **Current model** | Gemini 2.5 Flash | Latency and cost on mobile, not capability ceiling. The gateway makes this reversible. |
| **Structured output** | `response_format: json_object` + defensive `indexOf` extraction | Prompt instructions alone produced 0% compliance. Enforcement had to be structural. |
| **Failure handling** | Typed envelope on every agent call: `ok: true \| ok: false` | Callers branch on a type, not on a try/catch of a parse error |
| **Tool surface** | Read-only MCP server exposing five tools | Territory data is queryable by an external agent without write risk |

### Deliberately NOT built

- **No agent framework.** Every agent is a single-shot task over structured
  data. An orchestration framework would add a dependency and a failure surface
  with nothing to orchestrate.
- **No vector DB, no embeddings.** The corpus is a typed outlet dataset, not
  unstructured documents. Retrieval is a filter, not a similarity search.
- **No client-side API keys, no key modal, no localStorage.** Removed in v2.

### Deprecated (do not reintroduce)

- Direct browser → `api.anthropic.com` calls
- `x-api-key` in client headers
- Session-memory API key modal on first load
- Model hardcoded at the call site

---

## 2. AGENT INVENTORY

| # | Agent | Type | Status |
|---|---|---|---|
| 1 | **CommCheck** — pre-visit commercial intelligence | LLM | Shipped |
| 2 | **Draft Order Agent** — AI-prepared order with visible reasoning | LLM | Shipped |
| 3 | **Territory Pulse** — cross-outlet territory synthesis | LLM | Shipped |
| 4 | **Loss Loop** — lost-opportunity detection and closure | LLM | Shipped |
| 5 | **OnboardIQ** — new outlet onboarding tracker | **Rules-based, no LLM** | Shipped |

### Why OnboardIQ has no model in it

Onboarding status is a four-stage state machine with fixed day thresholds. The
inputs are enumerated, the transitions are ordered, and the output is a
template with variables substituted in. There is no ambiguity for a model to
resolve — introducing one would add latency, cost, and non-determinism to
arithmetic.

This is the counterpart to the same call made in production: rules-based route
planning was chosen over ML on the 22-market sales execution platform for the
same reason. **Knowing where a model does not earn its keep is the harder
judgment.**

---

## 3. EVAL HARNESSES

Two real harnesses, written and run, with documented findings.

### 3.1 [`commcheck`](https://github.com/namratharuds-stack/fieldiq-evals/tree/main/commcheck) — CommCheck

**19 cases across three layers:** schema validity, rule adherence,
adversarial / prompt injection.

| # | Finding | Lesson |
|---|---|---|
| 1 | **0% compliance** with the "return ONLY valid JSON, no preamble, no markdown" instruction — violated on every single case | Prohibitions in a prompt function as *requests*, not constraints. Enforcement requires a structural mechanism. |
| 2 | Silent rule extension — the model applied tier-specific rules to outlet types the spec never covered, without flagging the gap | Models fill spec gaps quietly rather than surfacing them |
| 3 | One rule failure traced to spec ambiguity — "behind target" and "contract breach" were conflated in the prompt | Precision in the spec, not just the model, drives correctness |

**Fix shipped:** `response_format: json_object` enforced at the gateway, plus
defensive `indexOf` extraction as a second line. Spec language disambiguated:
*shortfall* (volume behind pace) and *breach* (contractual term violated) are
now distinct terms with distinct card types.

### 3.2 [`draft-order`](https://github.com/namratharuds-stack/fieldiq-evals/tree/main/draft-order) — Draft Order Agent

**26 cases across 13 categories.** Per-case quantity bands, exact-match reason
codes, and a criticality model.

**Always-critical failures** (any one fails the gate regardless of score):
- Recommending a SKU not in `activeSkus` or `eligibleSkus`
- MOQ violation
- Missing escalation on a payment-overdue outlet

**Seeded fixture:** a deliberately broken agent reproducing four documented
failure modes was run through the grader. The harness had to catch them — this
validates the *grader*, not the model.

**Gate result: deliberate FAIL at 22/26.**

> The FAIL is the deliverable. A green board proves nothing if the test can't
> go red. This one can, and did, on exactly the cases it was built to catch.

---

*[Namratha Rudrappa](https://namratharudrappa.com) · Senior Product Manager · CPG Commercial Execution & AI*
