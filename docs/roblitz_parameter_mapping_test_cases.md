# Manual Test Cases: Röblitz Parameter Mapping (Phase 7)
## Verification Protocol & Ablation Testing Suite

**Document Version:** 2.0.0 (Revised per Expert Biological & Numerical Review)  
**Target Milestone:** Phase 7 — Röblitz Parameter Mapping  
**Project:** OvaTwin (Hybrid AI + Physiological Digital Twin for PCOS)  
**Status:** Pre-implementation Test Specification (No Code Implemented Yet)  

---

## 1. Overview & Testing Strategy

This document establishes the **manual and automated test protocols** for validating Phase 7 before and during implementation.

### Key Testing Principles:
1. **Baseline Preservation Invariance (Zero-Perturbation Test):** When mapping is disabled or when clinical data is completely missing, the personalized digital twin simulation must match the generic Röblitz baseline simulation with **0.0 trajectory delta** across all 46 species.
2. **Single-Parameter Ablation First:** Rather than testing all parameters simultaneously, each of the 5 candidate parameters (`p1`, `p17`, `p49`, `p51`, `p173`) must first be validated individually in isolation before joint multi-parameter simulations are run.
3. **No Silent Fallback on Failure:** When RoadRunner/CVODE encounters numerical stiffness (e.g. Day 91), the failure must produce an explicit, transparent `SimulationResult(success=False, ...)` diagnostic object rather than silently altering patient parameters.
4. **Two-Layer Safety Verification:** Verify that values stay within the soft bounded perturbation window ($\pm 35\%$) and never violate the hard physiological safety clamps ($[0.4\times, 2.5\times]$).

---

## 2. Test Case Specifications

### Test Case TC-00: Baseline Preservation Invariance (Crucial Foundation Test)
* **Objective:** Ensure that the personalization pipeline does not inadvertently alter the published Röblitz baseline model when no patient data is applied.
* **Input Profile:**
  - Patient with all clinical observations set to `None` (or mapping disabled via flag `personalized=False`).
* **Execution:**
  1. Run generic simulation: $T_{generic} = \text{RoblitzModelRunner}().\text{simulate}(0, 120, 1201)$.
  2. Run personalized twin simulation: $T_{twin} = \text{PatientDigitalTwin}(\text{empty\_patient}).\text{run\_baseline}(0, 120, 1201)$.
* **Expected Result:**
  - $\max | T_{twin} - T_{generic} | = 0.0$ for all 46 species and all 1201 timepoints.
  - Zero warnings, status = `baseline_ready`.

---

### Test Case TC-01: Soft Bounded Scaling & Hard Clamp Unit Tests
* **Objective:** Verify the mathematical behavior of `bounded_scale()` across normal, extreme, and missing inputs.
* **Test Sub-cases:**
  | Case ID | Input Value $x$ | Reference $x_{ref}$ | $\Delta_{max}$ | $\alpha$ | Expected Output $S(x)$ |
  | :--- | :--- | :--- | :--- | :--- | :--- |
  | **TC-01a** | $5.0$ (equal) | $5.0$ | $0.35$ | $0.8$ | Exactly $1.000$ (no change) |
  | **TC-01b** | $15.0$ (3x ref) | $5.0$ | $0.35$ | $0.8$ | $1.0 + 0.35 \cdot \tanh(1.6) \approx 1.323$ (soft bounded) |
  | **TC-01c** | $500.0$ (extreme outlier) | $5.0$ | $0.35$ | $0.8$ | $\le 1.350$ (strictly bounded by $\Delta_{max}$) |
  | **TC-01d** | $0.1$ (near zero) | $5.0$ | $0.35$ | $0.8$ | $\ge 0.650$ (strictly positive, no negative output) |
  | **TC-01e** | `None` / `NaN` | $5.0$ | $0.35$ | $0.8$ | Exactly $1.000$ (graceful fallback) |
* **Hard Clamp Check:**
  - If a forced scale exceeds safety limits (e.g. $S = 3.5$), `clamp(P, 0.4*P_base, 2.0*P_base)` clamps strictly at $2.0 \times P_{base}$.

---

### Test Case TC-02: Missing Clinical Data & Audit Logging
* **Objective:** Verify that missing optional observations (e.g. `AMH = None`, `progesterone = NaN`, `follicle_count = None`) produce baseline defaults and clear diagnostic log entries.
* **Input Profile:**
  - `lh`: 8.5 mIU/mL
  - `fsh`: 4.2 mIU/mL
  - `amh`: `None`
  - `total_follicle_count`: `None`
* **Expected Result:**
  - `p1` (`b_syn_LH`) is personalized based on available `lh` and `fsh`.
  - `p49` (`k_AF1`) and `p173` (`b_IhB`) remain strictly at published baseline values ($3.661804$ and $89.493467$).
  - An audit log entry is generated for each missing marker:
    ```text
    [AUDIT] Patient #X: Parameter p173 assigned baseline (89.493) due to missing 'amh' / 'total_follicle_count'.
    ```
  - No `KeyError` or `TypeError` raised.

---

## 3. Incremental Ablation Test Suite (Core 5 Parameters)

Each candidate parameter must be simulated in isolation before combined execution:

### Test Case TC-A1: Single-Parameter Ablation — `p1` (`b_syn_LH`) Only
* **Patient Profile:** Classic elevated LH (e.g. LH = 14.5 mIU/mL, LH:FSH = 2.8).
* **Configuration:** Only `p1` is modified; `p17, p49, p51, p173` remain at baseline.
* **Expected Trajectory Impact:**
  - Pituitary LH reservoir (`LH_Pit`) expands.
  - Circulating blood LH (`LH_bld`) baseline level increases by $1.0$–$3.5\text{ mIU/mL}$.
  - Blood FSH (`FSH_bld`) experiences secondary mild suppression via estradiol/inhibin feedback loops.
  - RoadRunner simulation runs cleanly without CVODE solver failure.

### Test Case TC-A2: Single-Parameter Ablation — `p17` (`b_rel_FSH`) Only
* **Patient Profile:** Relative FSH deficiency (e.g. FSH = 3.2 mIU/mL).
* **Configuration:** Only `p17` is modified; all other parameters at baseline.
* **Expected Trajectory Impact:**
  - Circulating blood FSH (`FSH_bld`) baseline shifts downward.
  - Antral follicle growth rate ($AF1, AF2$) slows slightly in response to lower FSH stimulation.

### Test Case TC-A3: Single-Parameter Ablation — `p49` (`k_AF1`) Only
* **Patient Profile:** Polycystic ovarian morphology (Total Follicle Count = 32).
* **Configuration:** Only `p49` modified ($+25\%$ increase in initial recruitment rate).
* **Expected Trajectory Impact:**
  - State variable `AF1` (early antral follicles) elevates above baseline.
  - Modest downstream increase in $AF2$ pool.

### Test Case TC-A4: Single-Parameter Ablation — `p51` (`k_AF2_AF3`) Only
* **Patient Profile:** Oligo/amenorrhea with follicular arrest (Irregular cycle, small follicle size).
* **Configuration:** Only `p51` modified ($-25\%$ reduction in transition to dominant stage).
* **Expected Trajectory Impact:**
  - Accumulation in $AF2$ follicle stage.
  - Delayed or blunted pre-ovulatory follicle (`PrF`) emergence and lower peak Estradiol ($E_2$).

### Test Case TC-A5: Single-Parameter Ablation — `p173` (`b_IhB`) Only
* **Patient Profile:** High follicle count / high AMH.
* **Configuration:** Only `p173` modified ($+30\%$ increase in Inhibin B synthesis).
* **Expected Trajectory Impact:**
  - Circulating Inhibin B (`InhB`) shifts upward.
  - Feedback suppression on pituitary FSH synthesis/release.

### Test Case TC-A6: Combined Core 5-Parameter Personalized Baseline
* **Patient Profile:** Complete classic PCOS profile (Patient #1).
* **Configuration:** `p1, p17, p49, p51, p173` mapped simultaneously.
* **Expected Trajectory Impact:**
  - Concordant phenotype: elevated circulating LH, suppressed/blunted FSH, follicular arrest in $AF2$, elevated Inhibin B, delayed/attenuated ovulatory LH/E2 surge.
  - Simulation status: **PASS** (1201 points, shape `(1201, 46)`, 0 NaN, 0 Inf).

---

## 4. Solver Failure & Diagnostic Capture Test

### Test Case TC-03: Explicit Solver Error Capture (No Silent Masking)
* **Objective:** Verify that if an extreme parameter combination triggers a CVODE stiff integration issue (e.g. step size $h \to 0$ around Day 91), the failure is transparently captured.
* **Execution:**
  - Test runner deliberately injects a stress parameter set designed to trigger a solver halt.
* **Expected Result:**
  - System **DOES NOT** fall back to another parameter set or return false data.
  - System returns:
    ```python
    result.success == False
    result.error_message.contains("CVODE")
    result.parameters_used == {"p1": ..., ...}
    result.trajectory is None
    ```
  - An informative warning is logged with the parameter values that caused the issue for research investigation.

---

## 5. Cohort Batch Verification (All 541 Clinical Patients)

### Test Case TC-04: Batch Pipeline Robustness
* **Objective:** Execute the Core 5 mapping across all 541 patients in `PCOS_data_without_infertility-Full_new.csv`.
* **Verification Criteria:**
  - 541/541 patients successfully map to valid parameter sets (0 exceptions).
  - 100% of generated parameters satisfy:
    - $P_{final} > 0$
    - $0.4 \times P_{base} \le P_{final} \le 2.5 \times P_{base}$
  - Cohort summary table generated reporting: Mean, Std, Min, Max for `p1, p17, p49, p51, p173`.
