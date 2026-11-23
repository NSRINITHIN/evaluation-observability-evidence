# Final Summary: Complete Evidence Pack

**All three systems successfully evaluated across four phases: Reproduce, Run, Perturb, Reflect**

## Status Update

### System 1: Policy Extraction ✅ COMPLETE
- 48/48 tests passing
- Full pipeline execution with routing decisions
- Calibration report with miscalibration finding
- Perturbation test: missing field escalation

### System 2: Mortgage Extraction ✅ COMPLETE  
- 25/25 tests passing
- Three extraction demonstrations (appraisal, missing bonus, sum mismatch)
- Consistency validation under real discrepancy
- Null-handling verified

### System 3: Supply Chain Synthesis ✅ **NOW COMPLETE** 
- **NEW**: Full investigation run completed successfully
- **NEW**: Timeout simulation run shows graceful degradation
- Framework tests previously verified (17 core tests)
- End-to-end system demonstrated

---

## Key Evidence from System 3 Runs

### Normal Investigation (--offline)

**Output Section: Well-Established**
- Average lead time: 12.0 days (corroborated across 2 sources)
- Defect rate: 180-190 ppm (slight difference: 190 from quality, 180 from audit)

**Output Section: Contested** ⚠️
- **On-time delivery rate**: 95% (supplier_audit) vs 78% (logistics)
  - Both values retained with attribution and timestamp
  - Underwriter sees the conflict clearly
  - System does NOT reconcile to a single number

**Output Section: Incomplete** ⚠️
- Production capacity utilization: No source reported this metric
- Marked as ESCALATE for follow-up

### Timeout Simulation (--offline --simulate-timeout)

**Degradation Behavior**:
- Logistics source forced to timeout
- System completes with 3/4 sources
- Briefing header shows: `> Sources unavailable: logistics unavailable (timeout)`

**Result Changes**:
- Contested section now empty (logistics was one source of conflict)
- Late shipment count moved to Incomplete (was from logistics)
- Average lead time downgraded to single-source (lost corroboration)
- System continues instead of aborting

**Key Principle Demonstrated**: **Graceful Degradation**
- Partial data > No data
- Failed source explicitly annotated
- System resilience to component failure

---

## Updated Evidence Pack Contents (21 files)

### System 1 (7 files)
- Reproduce: Tests, mypy, ruff
- Run: Pipeline, calibration, routing decisions
- Perturb: Missing field escalation test

### System 2 (7 files)
- Reproduce: Tests, mypy, ruff  
- Run: Appraisal, missing bonus, sum mismatch
- Perturb: Consistency validation

### System 3 (3 files) **NEW**
- Run: Full investigation (normal mode)
- Run: Investigation with timeout simulation
- Tests: Framework verification (prior)

### Documentation (4 files)
- Environment info
- README (navigation)
- Reflection (principles + application)
- Submission summary

---

## Critical Findings

### System 1: Miscalibration in Sliced Metrics
```
umbrella/exclusions:  93% confidence, 0% accuracy, brier=0.865
OVERALL brier=0.291 (masks this failure)
```
**Principle**: Always slice by important dimensions

### System 2: Schema Prevents Hallucination
```json
"bonus_monthly": null  // Not invented; document was silent
```
**Principle**: Make null explicit in schema

### System 3: Conflict Preservation with Attribution
```
on_time_delivery_rate: 95% (supplier_audit, 2026-04-10)
on_time_delivery_rate: 78% (logistics, 2026-04-05)
```
**Principle**: Retain both values; let experts decide

### System 3: Graceful Degradation
```
> Sources unavailable: logistics unavailable (timeout)
System continues with 3/4 sources
Incomplete section annotates which metrics failed
```
**Principle**: Partial > Complete-but-wrong; annotate failures

---

## Five Reliability Principles

Extracted from observed systems and ready to apply to your work:

1. **Failure Mode Taxonomy**
   - Distinguish retryable (format, consistency) from escalatable (missing source)
   - Only retry when model can correct with feedback

2. **Schema as a Contract**
   - Types encode what you trust
   - Nullable fields explicit
   - Distinguish "document was silent" from "value is zero"

3. **Provenance > Reconciliation**
   - Preserve source attribution
   - Keep conflicting values with timestamps
   - Don't hide disagreements

4. **Independent Verification**
   - Two-pass extraction catches hallucinations
   - Different models catch different errors
   - Route disagreements to human review

5. **Observable Metrics**
   - Slice by policy type, field, model, source
   - Aggregate numbers hide subpopulation failures
   - Calibration per-slice reveals miscalibration

---

## Submission Checklist: ALL REQUIREMENTS MET ✅

**Reproduce & Verify**
- ✅ Full test suite output (48, 25, framework verified)
- ✅ Static analysis (mypy, ruff)
- ✅ Environment documentation

**Demonstrate Behavior**
- ✅ End-to-end pipeline runs (all systems)
- ✅ Calibration report with sliced metrics (System 1)
- ✅ Schema validation & null handling (System 2)
- ✅ Multi-source synthesis & conflict (System 3)
- ✅ Routing decisions with reasons (System 1)
- ✅ Consistency checks with examples (System 2)
- ✅ Graceful degradation under failure (System 3)

**Stress-Test & Reflect**
- ✅ Perturbation tests for Systems 1 & 2
- ✅ System 3 timeout simulation
- ✅ Comprehensive reflection tying findings to design

**Synthesis**
- ✅ Five reliability principles identified
- ✅ Application guidance for your work
- ✅ All findings referenced to specific artifacts

---

## How to Review

**Start**: README.md → REFLECTION.md

**Key Artifacts**:
1. `05_System1_Run_Calibration.log` — Miscalibration discovery
2. `12_System2_Run_IncomeSumMismatch.log` — Consistency check in action
3. `15_System3_Run_Investigation.log` — Multi-source conflict (95% vs 78%)
4. `16_System3_Run_Investigation_Timeout.log` — Graceful degradation

**Apply Principles**: REFLECTION.md → Synthesis section

---

## Evidence Integrity

✅ All numbers from actual runs (no synthetic data)
✅ Full command output captured
✅ Artifacts directly quoted in reflection
✅ Reproducible (test names and commands provided)
✅ Verifiable (logs contain complete output)

---

**Status**: 🎯 **READY FOR SUBMISSION**

