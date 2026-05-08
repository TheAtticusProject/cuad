# Meeting Prep — Field Extraction Quality

## The Two Root Causes of Poor Scores

All the fields with bad scores trace back to exactly two problems:

---

### Problem 1 — Schema Type Mismatch (Multi-Span Fields)

The model is designed to extract **one value per field**. But many fields in CUAD have **multiple valid answer spans per document** — sometimes up to 55. When the model extracts 1 and the GT has 6, scoring it as 50% or even 16.7% misrepresents what actually happened. The model found the right thing, it just didn't find all of them.

This is a **schema architecture issue**, not a model quality issue. The fix is to change these fields from `string` → `array` type in the schema, so the model is prompted to return a list and the evaluator knows to score against each span individually.

**Fields that must become array type (sorted by severity):**

| Field | % of Docs with GT | Docs with Multiple Spans | Max Spans in One Doc |
|-------|-------------------|--------------------------|----------------------|
| Parties | 99.8% | 508 / 509 | **55** |
| Post-Termination Services | 35.7% | 91 / 182 | **23** |
| Rofr/Rofo/Rofn | 16.7% | 62 / 85 | **22** |
| Audit Rights | 42.0% | 136 / 214 | **19** |
| Insurance | 32.5% | 115 / 166 | **17** |
| Cap on Liability | 53.9% | 161 / 275 | **16** |
| License Grant | 50.0% | 164 / 255 | **16** |
| IP Ownership Assignment | 24.3% | 79 / 124 | **15** |
| Minimum Commitment | 32.4% | 98 / 165 | **14** |
| Non-Compete | 23.3% | 61 / 119 | **12** |
| Exclusivity | 35.3% | 104 / 180 | **12** |
| Non-Transferable License | 27.1% | 68 / 138 | **12** |
| Change of Control | 23.7% | 72 / 121 | **11** |
| Joint IP Ownership | 9.0% | 21 / 46 | **11** |
| Affiliate License-Licensor | 4.5% | 15 / 23 | **10** |
| Affiliate License-Licensee | 11.6% | 26 / 59 | **10** |
| Liquidated Damages | 12.0% | 27 / 61 | **10** |
| Irrevocable or Perpetual License | 13.7% | 36 / 70 | **10** |
| Warranty Duration | 14.7% | 37 / 75 | **10** |

**Concrete example — Audit Rights (PACIRA doc):**
- GT has **6 spans**: each covering a different obligation (right to inspect, timing, cost allocation, frequency limit, scope, etc.)
- Model extracts **1 span** (the main right-to-audit sentence)
- Current score: **~17%** (but displayed as 50% due to UI bucketing)
- Correct framing: model found the primary clause correctly, but missed 5 supporting clauses
- Fix: schema expects array → model returns array → each span scored independently → Span Recall = 1/6 = 16.7%, which is honest

---

### Problem 2 — Field Descriptions Too Narrow (False Negatives)

The model is looking for structured, clean values (a date, a duration, a yes/no flag) but CUAD's ground truth for several fields is the **full clause text** — a complete sentence or paragraph. The model returns null because it doesn't find a clean value, even though the clause is right there in the document.

This is a **field description issue**. The fix is to broaden the extraction prompt to accept clause-level text, not just structured values.

**Affected fields with GT examples:**

#### `expiration_date`
The model looks for a date like `2024-12-31`. But CUAD GT is:
- `"The term of this Agreement shall be ten (10) years which shall commence on the date..."`
- `"The Contract is valid for 5 years, beginning from and ended on ."`
- `"This Agreement shall commence on the Effective Date and shall continue for a period of six (6) months..."`

The contract describes the term in prose — there is no clean date to extract.

**Current description (too narrow):**
> Extract the date on which the contract's initial term expires.

**Improved description:**
> Extract the expiration date or term duration of the contract. This may be a specific date (e.g. "December 31, 2025"), a duration from the effective date (e.g. "two years from the Effective Date"), or a full clause describing the term length (e.g. "The term of this Agreement shall be ten (10) years commencing on..."). If no explicit expiration is stated but the contract term is described, extract the full term description clause.

---

#### `warranty_duration`
The model looks for a duration like `24 months`. But CUAD GT is:
- `"If, within the twenty-four (24) month warranty period set forth above..."` (full sentence)
- `"Within 7 days after the arrival of the goods at destination, should the quality... be found not in conformity..."` (a relative timeframe clause, not a clean duration)
- `"Google warrants that the Distribution Products will for a period of [*] from the date of supply..."` (redacted period, full warranty clause)
- Up to **8 spans** in a single document covering different warranty obligations

**Current description (too narrow):**
> Extract the duration of any warranty against defects or errors.

**Improved description:**
> Extract all clauses that describe warranty duration or warranty obligations related to defects, errors, or product performance. This includes specific durations (e.g. "24 months"), relative timeframes (e.g. "within 7 days of delivery"), and full warranty clauses that define the warranty period and its conditions. Extract each distinct warranty clause as a separate item if multiple warranties apply.

---

#### `post_termination_services`
The model may look for a simple yes/no or a short description. But CUAD GT is full clause text describing obligations that survive termination — IP handover, wind-down, continued performance, transition assistance. Up to **23 spans** in a single document.

**Current description (too narrow):**
> Identify if a party has obligations after the contract terminates or expires.

**Improved description:**
> Extract all clauses describing obligations that apply after the contract terminates or expires. This includes: transition assistance, IP or data transfer on termination, wind-down commitments, last-buy rights, continued performance during notice periods, payment obligations surviving termination, and any other post-termination duties explicitly stated. Extract the full clause text for each distinct obligation, not just a yes/no flag.

---

#### `rofr_rofo_rofn`
CUAD GT for this field is always multi-span (62 of 85 docs with GT have multiple spans, max 22). The full mechanism of a right of first refusal requires multiple clauses to be legally meaningful — the trigger event, the notice period, the response window, what happens if the right is not exercised.

**Current description (too narrow):**
> Does the contract grant a right of first refusal, right of first offer, or right of first negotiation?

**Improved description:**
> Extract all clauses related to any right of first refusal (ROFR), right of first offer (ROFO), or right of first negotiation (ROFN). This includes: the clause granting the right, the triggering event or condition, the notice and response procedures, the time window to exercise the right, what happens if the right is not exercised, and any carve-outs or exceptions. Each component clause should be extracted separately.

---

## What This Means for Scores

Before any fixes, here is the honest state of scoring:

| Issue | What the score shows | What it actually means |
|-------|---------------------|----------------------|
| Parties extracted 2 of 5 spans | 50% (UI bucket) | Model found 2 of 5 named parties — partial, not failed |
| Audit Rights extracted 1 of 6 | 50% (UI bucket) | 1/6 = 16.7% span recall — model found the right clause but missed supporting ones |
| Expiration Date returned null | 0% | Field description mismatch — the clause exists but model couldn't parse it as a date |
| Warranty Duration returned null | 0% | Same — GT is a full clause, model expected a clean duration |

---

## Prioritised Fix List for Tomorrow

### Quick wins (field description changes only — no code changes):
1. [ ] Update `expiration_date` description → accept term duration clauses
2. [ ] Update `warranty_duration` description → accept full warranty clauses, expect array
3. [ ] Update `post_termination_services` description → full clause text, expect array
4. [ ] Update `rofr_rofo_rofn` description → multi-clause extraction, expect array

### Schema changes (require schema + evaluator update):
5. [ ] Change `parties` to array type — this is the most impactful single change (99.8% of docs, up to 55 spans)
6. [ ] Change `audit_rights` to array type
7. [ ] Change `cap_on_liability` to array type
8. [ ] Change `license_grant` to array type
9. [ ] Change `post_termination_services` to array type

### Metric changes (require benchmarking pipeline update):
10. [ ] Add **Span Recall** metric — number of GT spans covered / total GT spans
11. [ ] Add raw field counts — "extracted X of Y expected fields" per document
12. [ ] Fix document score formula inconsistency between `import_to_db.py` and `document_service.py`

---

## What to Show in the Meeting

1. **The multi-span table above** — shows clearly that the scoring problem is architectural, not model quality
2. **The Parties example** — 508 out of 509 documents have multiple party spans (up to 55). Extracting only the first party and scoring it as a failure is wrong by design.
3. **The Audit Rights example** — PACIRA doc has 6 GT spans. Model extracted 1. True Span Recall = 16.7%, not 50% or 0%.
4. **The 4 improved field descriptions above** — concrete, implementable changes that will immediately improve false negative rates on `expiration_date`, `warranty_duration`, `post_termination_services`, `rofr_rofo_rofn`.
5. **The confidence display fix** — already done. Fields now show real values (e.g. 88%) instead of snapping to 50%. New thresholds pending: ≥0.85 green, ≥0.60 amber, <0.60 red.
