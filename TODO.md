# Forage Project — To-Do

---

## 1. Schema: Multi-Span Fields

**Problem:** Some fields (e.g. Audit Rights) have multiple valid answer spans in the ground truth — up to 6. The model currently extracts only one span, so scoring 1/6 as 50% is wrong; it should be ~16.7%.

**Action Items:**
- [ ] Change the schema format for multi-span fields from a single string to an array/list type
- [ ] For each document, the schema should declare how many spans are expected per field so the model and evaluator both know what to target
- [ ] Identify which of the 41 fields are multi-span (start with Audit Rights, License Grant, Parties, Governing Law)
- [ ] Stephen to confirm the full list of fields that need array type in schema

---

## 2. Confidence Display Fix (Documents Page)

**Context:** Confidence shown per field in the Documents page currently passes through multiple stages:

1. **Base confidence** — LLM's self-reported certainty (0–1 float in JSON response)
2. **Adjusted confidence** — overwritten by validation pipeline: `base × validation_score × penalty × grounding signal`, then bucket-snapped to `passed=1.0 / partial=0.5 / failed=0.0`
3. **MQS override** — final stored value is the MQS score (`er.confidence = mqs.final_score` at line 838)

The 50% display is a UI quirk: `ui.js` (lines 51–63) snaps any value between 0.49–0.98 to "medium → 50%", making nearly everything show as 50%.

**What the score actually is:** The MQS score (measures trustworthiness of extraction — no ground truth involved). It is **not** an accuracy score.

**Action Items:**
- [x] Fix `ConfPill` to display the real float value (e.g. 87%) instead of snapping to 0/50/100
- [ ] Apply updated confidence tier thresholds:

  | Tier | Threshold | Rationale |
  |------|-----------|-----------|
  | High (green) | ≥ 0.85 | Model is confident, well-evidenced extraction |
  | Medium (amber) | ≥ 0.60 | Hedged or weak evidence |
  | Low (red) | < 0.60 | Unreliable, treat with caution |

  > Current `>= 0.99` threshold for "high" is unrealistically tight — almost nothing reaches it.

---

## 3. New Metrics: Span Recall

**Context:** Two metrics exist today:

| Metric | What it measures | Where it lives |
|--------|-----------------|----------------|
| Token F1 | Word overlap between extraction and full GT text. Partial credit for matching words. | Benchmarks page |
| LLM Judge Score | Correct / Partial / Incorrect vs GT | Benchmarks page |
| MQS | Trustworthiness of extraction (no GT needed) | Documents page |

**Missing metric — Span Recall:** Measures how many GT answer spans are fully covered by the extraction. Binary per span (covered or not).

Example — Audit Rights (6 GT spans, model extracted 1):

| Metric | Score | Why |
|--------|-------|-----|
| Token F1 | ~20–30% | Some word overlap but most words missing |
| Span Recall | 16.7% | 1 of 6 spans covered |

For legal contracts, Span Recall is more meaningful than Token F1 — a partially captured clause may have no legal value at all.

**Action Items:**
- [ ] Implement Span Recall metric in the benchmarking pipeline
- [ ] Add Span Recall column to the Benchmarks page per-field breakdown
- [ ] Decide whether Span Recall should also influence the document-level score

---

## 4. Two Scoring Systems — Clarify and Connect

**Context:** There are currently two completely separate scoring systems with no connection between them:

| System | Page | What it measures | Needs GT? |
|--------|------|-----------------|-----------|
| MQS | Documents page | Can we trust this extraction? (agreement, grounding, schema, history) | No |
| LLM Judge + Token F1 | Benchmarks page | Is this extraction actually correct vs CUAD GT? | Yes |

Span Recall doesn't exist anywhere yet — discussed as a potential improvement.

**Important nuance on document score:**
- **Benchmarks page** `document_score` = LLM correct rate (%) if available, otherwise Token F1 (%) — defined in `import_to_db.py`
- **Backend pipeline** `document_score` = average of `field_validation_score × (1 − min(field_penalty_total, 1))` across all fields — defined in `document_service.py`
- The Benchmarks.js UI describes DOC as "penalty-adjusted aggregate" but the import code uses LLM-correct% or F1% — **these are inconsistent and need to be aligned**

**Action Items:**
- [ ] Decide whether MQS and accuracy scores should be shown side-by-side in the UI, or kept separate
- [ ] Align the `document_score` formula between benchmark import and pipeline scoring — pick one definition and apply it consistently
- [ ] Add a tooltip or label in the UI making clear which score is MQS (trust) vs accuracy (vs GT)

---

## 5. Accuracy Prompt Changes

**Context:** The accuracy prompt change (collapsing whitespace, ignoring quotes) applies **only** to the LLM Judge score on the Benchmarks page. It has **no effect on MQS**.

**Action Items:**
- [ ] Verify the whitespace/quote normalisation changes are working correctly on the next benchmark run
- [ ] Document what normalisations are applied before LLM judge comparison so they're not accidentally changed later

---

## 6. False Positives — Model Over-Extracts

**Context:** The model is biased toward over-extraction. Example: NELNETINC document — model extracted `2020-03-27` as effective date from "Dated: March 27, 2020", but CUAD GT has no effective date for this document. The extraction may actually be correct, but it counts as a false positive against the benchmark.

**Rule:** If GT is absent but model predicted a value → scored as incorrect.

**Action Items:**
- [ ] Review all false positive cases in the current benchmark run
- [ ] Flag cases where CUAD GT may be incomplete/wrong vs cases where the model is genuinely hallucinating
- [ ] Consider adding a "model may be right, GT may be wrong" flag to debatable false positives

---

## 7. False Negatives — Model Misses GT Fields

**Context:** 5 cases across 4 fields where CUAD has a value but model returned nothing (as of last 5-doc run):

| Field | Missed in (docs) | Root Cause |
|-------|-----------------|------------|
| expiration_date | 2/5 | GT is a duration clause ("10 years"), model looks for a date |
| rofr_rofo_rofn | 1/5 | Field prompt/description too narrow |
| post_termination_services | 1/5 | Field prompt/description too narrow |
| warranty_duration | 1/5 | GT is a relative clause, model looks for a structured duration |

**Action Items:**
- [ ] Update field descriptions for `expiration_date`, `rofr_rofo_rofn`, `post_termination_services`, `warranty_duration` to accept duration expressions and full clause sentences, not just structured values
- [ ] Re-run benchmark after description changes and compare recall
- [ ] Watch FP rate alongside recall — broader descriptions can increase false positives

---

## 8. Metrics: Field-Level Recall and False Positive Rate

**Definitions:**
- **Field Recall %** — of all fields where the contract has a clause (GT non-empty), what % did the model extract? (Last run: 89.8%)
- **FP Rate %** — of all fields where the clause is absent (GT empty), what % did the model wrongly extract? (Last run: 5%)

**Also needed — raw counts metric:**
- [ ] Add a metric that shows: fields extracted / fields that should have been extracted (raw integers, not just %)
- [ ] Surface this per-document on the Documents page and per-field on the Benchmarks page

---

## 9. Fields to QA / Spot-Check

The following fields need manual spot-checking across multiple documents for extraction quality:

- [ ] Parties
- [ ] Termination for Convenience
- [ ] Affiliate License-Licensor
- [ ] Affiliate License-Licensee
- [ ] Governing Law
- [ ] Cap on Liability
- [ ] License Grant
- [ ] Audit Rights
- [ ] Anti-Assignment
- [ ] Exclusivity
- [ ] Minimum Commitment
- [ ] No-Solicit of Customers
- [ ] Non-Compete
- [ ] Revenue/Profit Sharing
- [ ] Renewal Term
- [ ] Rofr/Rofo/Rofn

---

## 10. Dashboard — Evidence and GT Display

**Current state:**
- Amber highlight = evidence (the grounding text from the contract)
- Blue highlight = first match / extracted field value
- Ground truth is **not** shown on the dashboard

**Action Items:**
- [ ] Decide whether GT should be surfaced on the Documents page (useful for debugging, but only relevant for benchmarked documents)
- [ ] If yes, add a GT column/panel that shows the CUAD ground truth span alongside the model extraction

---

## 11. Seed Data

- [ ] Review `seed_data.json` — check what documents and fields are seeded, whether they are representative, and whether any updates are needed

---

## 12. More Loan Agreement Documents

**Context:** Current benchmark runs are on a small set of documents. More loan-specific documents are needed for meaningful evaluation.

**Action Items:**
- [ ] Source additional loan agreement documents (note: CUAD skews toward tech/SaaS — loan-specific clauses like interest rate, collateral, default triggers are not well represented)
- [ ] Run extraction pipeline on the new documents
- [ ] Add them to the benchmark dataset with ground truth labels
- [ ] Check whether CUAD field descriptions map correctly to loan agreement language — some fields may need loan-specific wording

---

## 13. Expand Beyond 41 CUAD Fields

**Context:** CUAD defines 41 fields. The project may need additional fields not covered by CUAD, particularly for loan agreements.

**Action Items:**
- [ ] Audit the CUAD dataset for any unique fields not in the current 41 (check raw data for edge cases)
- [ ] Identify loan-specific fields missing from CUAD (e.g. interest rate, collateral, default triggers, prepayment penalty, covenant compliance)
- [ ] Define schema entries for any new fields: field name, description, answer format, expected type (string / array / date / yes-no)
- [ ] Discuss with Stephen which new fields are in scope for the next milestone
