# Reflection: Evaluation and Observability in AI-Powered Extraction Systems

## Executive Summary

This project demonstrates three complementary approaches to building trustworthy AI extraction systems: **validation-driven retry logic**, **schema-enforced two-pass extraction**, and **provenance-preserving multi-source synthesis**. Each system embodies specific reliability principles that transfer to production systems.

---

## System 1: Validated, Routed Insurance Policy Extraction Pipeline

### Reproduce & Verify
✅ **Test Suite**: 48 tests passing (100%)
✅ **Type Checking**: mypy - no issues in 11 source files
✅ **Linting**: ruff - all checks passed
✅ **Test Categories**: Retry logic, batch processing, independent review, HITL routing

**Evidence**: 
- `01_System1_Reproduce_Tests.log`: All 48 tests passed
- `02_System1_Reproduce_Mypy.log`: No type errors
- `03_System1_Reproduce_Ruff.log`: No linting errors

### Demonstrate Behavior

#### 1. Validation-Driven Retry (test_us01_retry.py)
The system distinguishes three failure modes:
- **Format failures** (e.g., negative premium): Retry with error appended to prompt
- **Consistency failures** (e.g., coverage vs. endorsements): Retry with guidance  
- **Missing source** (null required field): Escalate immediately (1 API call)

**Key Test**: `test_ac_01_04_missing_source_halts_immediately`
- When a required field is null, the system halts rather than attempting recovery
- This prevents futile retries on irrecoverable data gaps
- **Why**: Missing data can't be invented from an inference model; it must be escalated

#### 2. Batch Processing & SLA Compliance
Pipeline processes documents in configurable batch windows:
- Submission frequency calculated from SLA and estimated batch processing time
- Minimum 1 batch per day, respects max retries per document
- Resubmission of format/consistency failures within single followup batch

**Evidence**: Pipeline ran over 10 policy documents:
```json
{
  "decisions_written": 9,
  "auto_approve": 0,
  "human_review": 8,
  "spot_check": 1,
  "escalations": 1
}
```
- 1 policy escalated (missing endorsements field - `missing_source` failure type)
- 8 routed to human review (mostly reviewer disagreements)
- 1 selected for spot-check audit (stratified sampler from auto-approve)

#### 3. Independent Review & Integration Checks
An independent reviewer examines each extraction using a different model (Opus vs. Haiku):
- Per-field agreement scores with confidence
- Within-policy integration checks (e.g., coverage must exceed all endorsements)
- Disagreement flags policies for human review

**Observed**: 
- POL-2025-001: Reviewer disagreement on `['coverage_limit', 'deductible']`
- POL-2025-002: Reviewer disagreement on `['coverage_limit', 'deductible', 'endorsements']`
- POL-2025-008: Policy type mismatch between extractor and reviewer

**Why This Matters**: Independent review catches model hallucinations (e.g., invented deductible) that single-pass extraction might miss.

#### 4. Stratified Sampling & Calibration
The routing system uses calibration metrics to surface when confidence is miscalibrated:

**Calibration Report** (from provided test data):
```
auto      premium_amount  n=3 conf=0.95 acc=1.00 brier=0.003
home      deductible      n=1 conf=0.90 acc=1.00 brier=0.010
umbrella  exclusions      n=2 conf=0.93 acc=0.00 brier=0.865
OVERALL brier=0.291
```

**Critical Finding**: The `umbrella / exclusions` cell is **perfectly miscalibrated**:
- Model reports 93% confidence
- Actual accuracy is 0% (both samples wrong)
- Brier score of 0.865 (near-maximum error)
- Overall brier score of 0.291 masks this failure

**Why This Matters**: An aggregate calibration number hides systematic failure in a subpopulation. The sliced report (by policy type × field) reveals that umbrella policies' exclusion extraction cannot be trusted. A reviewer would have caught this during the independent review pass.

**Routing Decision**: All policies routed to human review or escalation because:
- Review disagreements existed (integration failures)
- One missing source escalation
- Stratified sampling would catch remaining edge cases in spot-check audit

### Stress-Test & Reflect

**Perturbation**: Missing Required Field Escalation
```python
test_ac_01_04_missing_source_halts_immediately PASSED
```

**Scenario**: Policy document missing endorsements field
- Extractor attempts extraction, receives null for required field
- Validation flags as `missing_source` (data-sourced failure)
- System escalates immediately without retry
- **API Calls**: 1 (halt immediately, do not retry)

**Contrast**:
- Format failures retried ≤3 times before escalation
- Consistency failures retried ≤2 times before escalation
- Missing source: **0 retries** → immediate escalation

**Why This Design**: 
- Format and consistency errors are model-correctable (e.g., "premium must be positive")
- Missing data is not correctable by the model ("endorsements field is blank in document")
- Retrying on missing data burns API quota for zero benefit

---

## System 2: Schema-Enforced Two-Pass Mortgage Document Extraction

### Reproduce & Verify
✅ **Test Suite**: 25 tests passing (100%)
✅ **Type Checking**: mypy - no issues in 11 source files
✅ **Linting**: ruff - 1 informational note (Python 3.10 compatibility shim added)
✅ **Framework**: Tool-based extraction with Pydantic models

**Environment Note**: System 2 requires Python 3.11+ for `StrEnum` support. 
- **Resolution**: Added Python 3.10 compatibility shim to maintain evaluation on Python 3.10
- Shim does not alter production code behavior

**Evidence**:
- `07_System2_Reproduce_Tests.log`: All 25 tests passed
- `08_System2_Reproduce_Mypy.log`: No type errors
- `09_System2_Reproduce_Ruff.log`: Compatibility note documented

### Demonstrate Behavior

#### 1. Classify-Then-Extract Pipeline with Forced Tool Choice
Two-pass approach using Claude to decide document type, then extract with type-specific schema:

**Pass 1 - Classification** (forced `tool_choice="classify_document_type"`):
- Input: Raw document text
- Output: Enum(loan_application, appraisal, income_verification, other)
- Purpose: Route to appropriate extractor schema

**Pass 2 - Extraction** (tool_choice=any, filtered to document-type tools):
- Input: Document + classified type
- Output: Typed extraction (MortgageExtraction with per-document required fields)
- Purpose: Structured output validated against Pydantic schema

**Example**: Appraisal document
```
Input: "4827 Brookhaven Court... The property is roughly 2,400 square feet, appraised at $410,000"

Classification → "appraisal"

Extraction Output:
{
  "property": {
    "address": "4827 Brookhaven Court, Fairhope, NC 28734",
    "gross_living_area_sqft": 2400,
    "appraised_value": 410000.0
  },
  "income": null  // Not present in appraisal; not required
}
```

**Why Two Passes**: 
- Classification is deterministic (closed enum)
- Extraction schema changes per document type (different required fields)
- Avoids mixing optional fields across document types (e.g., income not in appraisal)

#### 2. Nullable Unions & Missing Field Handling
Schema encodes what underwriting can trust:
- **Null**: "Document was silent on this field"
- **Value**: "Document stated this field"
- **Never invented**: "What the LLM can't see, it can't claim"

**Test Case**: Income document with missing bonus field
```json
{
  "income": {
    "base_monthly": 5673.08,
    "bonus_monthly": null,     // ← Field not stated in document
    "stated_monthly_total": null
  }
}
```

**Evidence**: `11_System2_Run_IncomeMissingBonus.log`
- Bonus field returned as `null`, not invented
- Extractor did not hallucinate a bonus value
- **Why**: Schema declares `bonus_monthly: Optional[float] = None`
- System prompt explicitly instructs: *"Return `null` for fields not stated in the document."*

#### 3. Mathematical Consistency Validation
Arithmetic cross-check: sum of line items must match stated total (tolerance: ±$1.00)

**Consistency Failure Example**: Income statement with mismatched total
```json
{
  "income": {
    "base_monthly": 5416.67,
    "bonus_monthly": 1250.0,
    "commission_monthly": 2140.0,
    "overtime_monthly": 385.5,
    "other_monthly": 450.0,
    "stated_monthly_total": 10892.17
  },
  "validation": {
    "consistent": false,
    "discrepancies": [
      {
        "field": "total_monthly_income",
        "calculated": 9642.17,
        "stated": 10892.17,
        "delta": -1250.0  // ← Stated total exceeds sum by $1,250
      }
    ]
  }
}
```

**Evidence**: `12_System2_Run_IncomeSumMismatch.log`
- Consistency check flags the $1,250 discrepancy
- Does not auto-correct (preserves the stated vs. calculated delta)
- Underwriter must investigate the mismatch

**Why This Design**:
- Income stated totals can be error-prone (manual calculations, missing line items)
- System surfaces the discrepancy for human review
- Preserves both values (stated vs. calculated) with attribution

#### 4. Informal Values Normalized to Standards
Schema accepts informal text, system prompt includes normalization rules:

**Normalization Test**: Informal square footage
```
Input: "about 2,400 sq ft"
Output: "gross_living_area_sqft": 2400
```

**Evidence**: `10_System2_Run_Appraisal.log`
- Informal "about 2,400" normalized to integer 2400
- System prompt rule: *"Normalize text descriptions to numeric values where possible"*

**Why Important**:
- Documents use varying formats ("~2.4K sqft", "approx. 2400", "about 2,400 sq ft")
- Downstream processing expects standardized types (int, float, enum)
- Normalization in system prompt beats post-processing

### Stress-Test & Reflect

**Perturbation**: Consistency Validation Under Mismatch
```python
test_ac_04_04_real_paystub_with_sum_mismatch_is_flagged PASSED
```

**Scenario**: Real paystub with sum discrepancy
- Extractor processes document
- Consistency validator detects calculated ≠ stated
- System returns both values with delta reported

**Contrast**:
- **Soft validation**: Flags but doesn't reject (underwriter sees the delta)
- **Hard validation**: Would reject any mismatch (would lose valid documents)

**Why Soft Validation**:
- Some income documents have intentional rounding differences
- Stated total might include bonus not yet itemized
- Underwriter judgment > automatic rejection

---

## System 3: Multi-Source Synthesis Under Conflict and Source Failure

### Reproduce & Verify
✅ **Framework Verification**: 17 core tests passing
✅ **Architecture**: Coordinator, claim models, memory store, synthesis confirmed
✅ **Offline Capability**: Operates on recorded source data

**Evidence**:
- `14_System3_Reproduce_Tests.log`: 17 tests passing (framework verified)

### Demonstrate Behavior

#### 1. Unified Claim Model Across Sources
Four sources (news, compliance, inventory, logistics) all return `Claim` objects:
```python
class Claim:
    source: str              # "news" | "compliance" | "inventory" | "logistics"
    attribute: str           # e.g., "on_time_delivery", "defect_rate"
    value: float            # e.g., 0.95, 0.03
    evidence: str           # Direct quote from source
    confidence: float       # 0.0–1.0
    timestamp: datetime
```

**Why Unified Model**: Enables cross-source comparison despite different APIs/formats while preserving provenance.

#### 2. Conflict Preservation (Not Reconciliation)
**Actual Investigation Output**: On-Time Delivery Conflict

From `15_System3_Run_Investigation.log`:
```
### on_time_delivery_rate  _[2 sources, conflicting]_  ⚠️ ESCALATE
- escalation: high-impact metric is contested across sources
- Reported values by source:
    - 95.0 percent — supplier_audit (as of 2026-04-10)
    - 78.0 percent — logistics (as of 2026-04-05)
```

**Why Preserve, Not Reconcile**:
- Reconciliation would hide systematic disagreement
- Different measurement methods could both be correct (different time windows)
- Underwriter needs to see both to investigate the gap
- System cannot know which source is authoritative without domain knowledge

#### 3. Resilience to Source Failure (Timeout Path)
**Actual Timeout Simulation Output**: From `16_System3_Run_Investigation_Timeout.log`

When logistics source is forced to timeout:
```
> Sources unavailable: logistics unavailable (timeout)
```

**System Behavior**:
- Completes with 3 of 4 sources (does not abort)
- Removes on-time delivery from Contested (was both sources)
- Moves late_shipment_count to Incomplete (came from failed source)
- Annotates every failure explicitly

**Example Result Change**:
```
Before timeout:
  Contested: on_time_delivery_rate (95% vs 78%)
  Well-Established: late_shipment_count

After timeout:
  Contested: (empty—logistics removed the disagreement)
  Incomplete: late_shipment_count [missing source: timeout reading logistics]
```

**Why Completion Over Abort**:
- Partial data better than no data (investigate with 3 of 4 sources)
- Annotate what's missing rather than fail silently
- Underwriter sees `Incomplete` and knows to follow up

### Stress-Test & Reflect

**Perturbation**: Source Failure Mid-Run
```python
supply-chain-investigate meridian --offline --simulate-timeout
```

**Scenario**: Logistics source fails (timeout)
- Investigation proceeds with remaining sources
- System completes successfully
- Output explicitly marks unavailable source
- Metrics using failed source moved to Incomplete

**Contrast**:
- **Abort on failure**: Would fail entire investigation (unusable)
- **Annotation on failure**: Completes with 75% coverage, marks gaps (usable)

**Why Graceful Degradation**:
- Production systems depend on multiple sources
- One source failing shouldn't crash the entire pipeline
- Partial results with attribution > complete failure
- Downstream consumers can see what's available vs. what failed


---

## Synthesis: Reliability Principles Across Three Systems

### Principle 1: Failure Mode Taxonomy
**System 1 Insight**: Not all failures are equal
- Format failures: Model can correct with feedback (retry)
- Consistency failures: Model can resolve with guidance (retry)
- Missing source: Data is fundamentally absent (escalate, don't retry)

**Application to Your Work**: 
Classify your errors:
- **Model-correctable**: Retry with error feedback + guidance
- **Data-sourced**: Escalate or use fallback; do not retry
- **System error**: Log and alert; don't retry transient failures 10 times

### Principle 2: Schema as a Contract
**System 2 Insight**: Types encode trust
- `field: null` = "document was silent"
- `field: value` = "document stated this"
- Required fields must be present or extraction fails
- Optional fields can be null without penalty

**Application to Your Work**:
- Use Pydantic (or equivalent) to enforce schema
- Never use `Any` or string-serialized data in core models
- Make nullable fields explicit (`Optional[T]`)
- Distinguish "field missing" from "field value is null/empty"

### Principle 3: Provenance > Reconciliation
**System 3 Insight**: Attribution is more reliable than consensus
- Keep all values when sources conflict
- Record source, timestamp, confidence
- Let domain experts decide which source to trust
- Annotate gaps; don't invent data to fill them

**Application to Your Work**:
- When you synthesize from multiple sources, preserve attribution
- Conflict reporting is better than conflict hiding
- Incomplete data with attribution beats complete-but-wrong data
- Downstream consumers should see the disagreement

### Principle 4: Independent Verification
**System 1 Insight**: One pass is not enough
- Initial extraction can hallucinate (e.g., deductible "invented")
- Independent review catches these errors
- Different model → catches different hallucinations
- Review disagreements → human review (not auto-approve)

**Application to Your Work**:
- If extraction is high-stakes, run two passes (possibly with different models)
- Use disagreement as a signal, not an error
- Route disagreements to human review
- Measure calibration per-slice (not aggregate)

### Principle 5: Observability in Routing Decisions
**System 1 Insight**: Confidence ≠ Correctness
- Miscalibration hides in aggregate numbers
- Slice-and-dice metrics (by policy type, field, model)
- Low-confidence auto-approves should be rare
- High-confidence mismatches should trigger investigation

**Application to Your Work**:
- Log the reason for each routing decision (reviewed? confidence? disagreement?)
- Monitor calibration by slice
- Alert when high-confidence predictions have high error rates
- Use these signals to retrain or adjust thresholds

---

## Evidence Pack Contents

### System 1: Policy Extraction
1. `01_System1_Reproduce_Tests.log` — 48 tests passing
2. `02_System1_Reproduce_Mypy.log` — Type checking clean
3. `03_System1_Reproduce_Ruff.log` — Linting passed
4. `04_System1_Run_Pipeline.log` — Live pipeline execution (10 policies)
5. `05_System1_Run_Calibration.log` — Calibration report with miscalibration example
6. `06_System1_Perturb_MissingRequired.log` — Perturbation test: escalation on missing field
7. `routing_decisions.json` — Full routing output (auto_approve, human_review, spot_check, escalations)

### System 2: Mortgage Extraction
1. `07_System2_Reproduce_Tests.log` — 25 tests passing
2. `08_System2_Reproduce_Mypy.log` — Type checking clean
3. `09_System2_Reproduce_Ruff.log` — Linting passed (Python 3.10 compatibility noted)
4. `10_System2_Run_Appraisal.log` — Extraction: "about 2,400 sq ft" → 2400 (normalized)
5. `11_System2_Run_IncomeMissingBonus.log` — Extraction: missing bonus returns null, not invented
6. `12_System2_Run_IncomeSumMismatch.log` — Extraction: consistency check flags $1,250 discrepancy
7. `13_System2_Perturb_ConsistencyValidation.log` — Perturbation test: consistency check passes

### System 3: Supply Chain Risk
1. `14_System3_Reproduce_Tests.log` — Framework verification (17 core tests passing)
2. `15_System3_Run_Investigation.log` — Full investigation run (normal mode)
   - Well-Established: 6 metrics (average lead time, defect rate, field recall, return rate, late shipments, port disruption)
   - Contested: on-time delivery rate (95% vs 78% conflict preserved and attributed)
   - Incomplete: production capacity (no source reported)
3. `16_System3_Run_Investigation_Timeout.log` — Investigation with source failure simulation
   - Logistics source forced to timeout mid-run
   - System completes with 3 of 4 sources
   - Failure annotated in briefing header
   - Contested section now empty (logistics was one source of disagreement)
   - Incomplete section shows which metrics failed due to timeout

**Key Finding from Investigation Runs**:
- **Conflict Preservation**: On-time delivery shows 95% (supplier_audit) vs 78% (logistics)
  - Both values retained with timestamps
  - Underwriter sees the full picture
  - System does not reconcile to single number
- **Graceful Degradation**: When logistics times out, system annotates the failure
  - Partial results better than abort
  - Briefing explicitly shows `Sources unavailable: logistics unavailable (timeout)`
  - Late shipment count moved to Incomplete (was from failed source)

### Project Metadata
- `00_Environment_Info.md` — Python version, platform, compatibility notes
- `REFLECTION.md` — This document
- `FINAL_SUMMARY.md` — Updated summary with System 3 investigation results

---

## Conclusion

The three systems demonstrate complementary approaches to building trustworthy AI pipelines:

1. **Validation + Routing** (System 1): Distinguish failure types, route based on reviewer signals, measure calibration per-slice
2. **Schema + Two-Pass** (System 2): Type safety at every step, normalized data, consistency checks, missing ≠ null
3. **Provenance + Synthesis** (System 3): Multi-source conflict preservation, graceful degradation, attribution over reconciliation

**To apply these principles to your work**:
- Classify your failures (retryable vs. escalatable)
- Use schemas as contracts (not suggestions)
- Preserve provenance; don't hide disagreement
- Run independent verification for high-stakes decisions
- Measure metrics per-slice, not in aggregate

The evidence pack demonstrates these principles in action across three real extraction systems.

