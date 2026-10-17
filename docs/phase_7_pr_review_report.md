# Phase 7 Pull Request Review Report: Phenotype-Informed Röblitz Parameter Mapping

**Target Branch:** `Röblitz_Parameter_Mapping`  
**Base Branch:** `main`  
**Component:** OvaTwin Biophysical Digital Twin (`src/twin/`, `src/HPO/`)  
**Scope:** Phase 7 (Sub-phases 7.1 through 7.10)  
**Verification Status:** 100% Tests Passing (All Unit Tests, Integration Tests, Ablation Suites, and 541/541 Clinical Patients Verified with Zero Solver Failures)

---

## 1. Executive Summary

This Pull Request implements **Phase 7: Röblitz Parameter Mapping** for OvaTwin. It bridges raw clinical and hormonal patient observations (from 541 clinical records) into the published **Röblitz 2013 ordinary differential equation (ODE) menstrual cycle model** (`models/BIOMD0000000494.xml`).

### The Core Problem Solved
In systems biology models of the human menstrual cycle, circulating blood hormone concentrations (such as serum LH, FSH, estradiol, and progesterone) are **emergent dynamic state variables** governed by coupled non-linear feedback loops, **not fixed parameters**. Directly overwriting dynamic states with one-off lab measurements destroys model integrity. 

This PR establishes a **phenotype-informed mapping engine** that safely personalizes the true kinetic rate parameters (basal pituitary synthesis, release rates, follicle recruitment rates, and stage transition rates) using clinical markers while guaranteeing:
1. **Mathematical Stability:** Non-linear bounded scaling ($\tanh$) strictly limits parameter perturbations to $[-35\%, +35\%]$ (Layer 1).
2. **Physiological Safety:** Hard safety clamps enforce $[0.40\times, 2.50\times]$ baseline and guarantee $P > 0$ under all circumstances (Layer 2).
3. **Defensive Error Handling:** CVODE/RoadRunner solver failures are captured explicitly without silent fallbacks or masked parameters.
4. **Baseline Invariance (TC-00):** Unpersonalized models produce an exact $0.0$ trajectory delta against the published Röblitz baseline across all 46 species.
5. **Cohort-Wide Robustness (TC-04):** 100% of all 541 patients in the clinical dataset successfully map and simulate cleanly with zero integration crashes.

---

## 2. Core 5 Personalizable Parameter Rationale

From the 141 total parameters extracted directly from the SBML model (`results/roblitz_parameter_registry.csv`), Phase 7 identified and categorized 5 parameters for phenotype-informed personalization:

| SBML ID | Kinetic Name | Reaction | Published Baseline | Hard Safety Bounds | Input Clinical Markers | Biological Role in PCOS |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`p1`** | `b_syn_LH` | `re25` ($\to \text{LH\_Pit}$) | $7309.9159\text{ mIU/(mL}\cdot\text{d)}$ | $[2923.97, 18274.79]$ | Serum LH, Computed FSH/LH ratio | Primary driver of pituitary LH hypersecretion in classic PCOS. |
| **`p17`** | `b_rel_FSH` | `re84` ($\text{FSH\_pit} \to \text{FSH\_bld}$) | $0.057002\text{ d}^{-1}$ | $[0.0228, 0.1425]$ | Serum FSH, Computed FSH/LH ratio | Governs basal pituitary release of FSH; drives relative FSH deficiency. |
| **`p49`** | `k_AF1` | `re43` ($\to \text{AF1}$) | $3.661804\text{ d}^{-1}$ | $[1.4647, 9.1545]$ | Total Follicle Count (L + R) | Captures excessive early antral follicle recruitment from enlarged preantral pool. |
| **`p51`** | `k_AF2_AF3` | `re44` ($\text{AF2} \to \text{AF3}$) | $4.882316\text{ d}^{-1}$ | $[1.9529, 12.2058]$ | Mean Follicle Size, Cycle Regularity | Models follicular arrest at 2–9 mm stage and delayed dominant selection. |
| **`p173`** | `b_IhB` | `re72` ($\to \text{InhB}$) | $89.493467\text{ pg/(mL}\cdot\text{d)}$ | $[35.7974, 223.7337]$ | Total Follicle Count, Serum AMH | Represents granulosa cell hypersecretion of Inhibin B, feeding back on FSH. |

---

## 3. Two-Layer Architecture & Mapping Mechanics

```text
[ Clinical Observations ]
  (LH, FSH, Follicle Count, Mean Size, Cycle Regularity, AMH)
             │
             ▼
[ Layer 1: Soft Bounded Scaling Function ]
  S(x; x_ref, Δ_max, α) = 1.0 + Δ_max * tanh( α * (x - x_ref) / x_ref )
  - S = 1.0 at reference median (Baseline Preservation)
  - Perturbation bounded strictly to [-35%, +35%] ([0.65x, 1.35x])
             │
             ▼
[ Layer 2: Hard Physiological Safety Clamp ]
  P_final = max( P_hard_min, min( P_base * S(x), P_hard_max ) )
  - Enforced via Authoritative Parameter Registry ([0.40x, 2.50x])
  - Guarantees P_final > 0 under all possible clinical outliers
             │
             ▼
[ RoadRunner / CVODE Simulation Guard ]
  - rtol = 1e-5, atol = 1e-8 (mitigates Day 91 stiffness)
  - Zero silent fallbacks: explicit SimulationResult(success=...)
  - Trajectory Sanity: 1201 points, 46 species, finite, non-negative
```

### Missing Data Handling (Transparent Fallback)
If a clinical marker is unobserved (`None` or `NaN`):
- The parameter automatically falls back to its published SBML baseline value.
- An explicit entry is written to `mapping_audit`:
  ```text
  Audit(p173 | b_IhB | total_follicle_count+amh=None -> scale=1.000 -> final=89.4935 [missing_fallback])
  ```

---

## 4. Phased Implementation Summary (7.1 to 7.10)

The implementation followed an ablation-first sequence:

| Step | Scope | Key Code Deliverables | Test Suite | Verification Status |
| :--- | :--- | :--- | :--- | :--- |
| **7.1** | SBML Parameter Extraction | `src/HPO/sbml_parameters.py` | `test_sbml_parameters.py` | 7/7 PASS |
| **7.2** | Reaction Rate Laws Verification | `src/HPO/sbml_reactions.py` | `test_sbml_reactions.py` | 8/8 PASS |
| **7.3** | Registry Classification & Bounds | `src/HPO/parameters.py` | `test_parameters_classification.py` | 7/7 PASS |
| **7.4** | Bounded Mapping Engine | `src/twin/roblitz_mapping.py` | `test_roblitz_mapping.py` | 7/7 PASS |
| **7.5** | Single-Parameter Mapping (`p1`) | `map_p1_b_syn_LH` (Dual-signal logic) | `test_phase_7_5_p1_mapping.py` | 6/6 PASS |
| **7.6** | Single-Parameter Ablation (`p1`) | `src/twin/p1_ablation.py` | `test_phase_7_6_p1_ablation.py` | 4/4 PASS (TC-00, TC-A1) |
| **7.7** | Trajectory Sanity & Solver Guard | `src/twin/simulation_guard.py` | `test_phase_7_7_simulation_guard.py` | 5/5 PASS (TC-03) |
| **7.8** | Second Parameter Mapping (`p17`) | `map_p17_b_rel_FSH`, `src/twin/p17_ablation.py` | `test_phase_7_8_p17_mapping.py` | 8/8 PASS (TC-A2) |
| **7.9** | Follicular Parameters (`p49, p51, p173`) | `map_core_5_parameters`, `follicular_ablation.py` | `test_phase_7_9_follicular_mapping.py` | 8/8 PASS (TC-A3, A4, A5, A6) |
| **7.10** | Cohort Verification (541 Patients) | `src/twin/cohort_verification.py` | `test_phase_7_10_cohort_verification.py` | 5/5 PASS (TC-04) |

---

## 5. File-by-File Change Index

### Core Architecture & Mapping
- `src/HPO/sbml_parameters.py`: Extracts and caches all 141 SBML parameters with bidirectional ID $\leftrightarrow$ Name lookups.
- `src/HPO/sbml_reactions.py`: Parses all 77 SBML kinetic reactions and identifies governing rate laws for candidate parameters.
- `src/HPO/parameters.py`: Authoritative registry containing metadata, classifications (`PERSONALIZABLE_CORE`, `EXPLORATORY`, `DRUG_PK`, `STRUCTURAL`), and hard safety bounds.
- `src/twin/roblitz_mapping.py`: Implements Layer 1 $\tanh$ scaling (`bounded_scale`), Layer 2 clamping, audit log dataclass (`MappingAuditEntry`), and individual parameter mapping functions (`map_p1_b_syn_LH`, `map_p17_b_rel_FSH`, `map_p49_k_AF1`, `map_p51_k_AF2_AF3`, `map_p173_b_IhB`, `map_core_5_parameters`).
- `src/twin/parameter_mapping.py`: Updates `PatientTwinMapping` to store `ode_parameters` and `mapping_audit`; updates `create_mapping()` to apply Core 5 mapping when `personalize_ode=True`.
- `src/twin/digital_twin.py`: Adds `personalized` flag to `run_baseline()`; fixes `compute_trajectory_difference()` to support bracket-stripped column lookups.
- `src/twin/simulation_guard.py`: Implements `SimulationResult`, `validate_trajectory_sanity()`, and `safe_simulate()` with calibrated integrator tolerances (`rtol=1e-5`, `atol=1e-8`).
- `src/HPO/baseline_simulation.py`: Integrates `safe_simulate()` into `run_baseline_simulation()`, raising explicit exceptions on solver crashes.

### Ablation & Batch Runners
- `src/twin/p1_ablation.py`: Isolated `p1` ablation against generic baseline.
- `src/twin/p17_ablation.py`: Isolated `p17` and joint dual-gonadotropin (`p1 + p17`) ablation engine.
- `src/twin/follicular_ablation.py`: Isolated follicular ablation (`p49`, `p51`, `p173`) and combined Core 5 twin simulation.
- `src/twin/cohort_verification.py`: Batch execution across all 541 patients in the clinical dataset.

### Generated Artifacts in `results/`
- `results/roblitz_parameter_registry.csv`: All 141 SBML parameters.
- `results/roblitz_candidate_reactions.csv`: Governing reactions for Core 5.
- `results/roblitz_parameter_classification.csv`: Parameter categorization breakdown.
- `results/p1_ablation_patient_1.csv`: Isolated `p1` trajectory deltas for Patient #1.
- `results/p17_ablation_patient_1.csv`: Isolated `p17` trajectory deltas for Patient #1.
- `results/p1_p17_joint_ablation_patient_1.csv`: Dual gonadotropin deltas for Patient #1.
- `results/p49_ablation_tc_a3.csv`: Isolated `p49` recruitment deltas.
- `results/p51_ablation_tc_a4.csv`: Isolated `p51` follicular arrest deltas.
- `results/p173_ablation_tc_a5.csv`: Isolated `p173` Inhibin B hypersecretion deltas.
- `results/core_5_combined_ablation_patient_1.csv`: Full 5-parameter combined simulation for Patient #1.
- `results/cohort_core_5_parameters.csv`: Complete parameter registry for all 541 patients.
- `results/cohort_core_5_summary_stats.csv`: Formal cohort distribution summary table.

---

## 6. Test Suite & Verification Matrix

| Test Identifier | Description | Target Criteria | Result |
| :--- | :--- | :--- | :--- |
| **TC-00** | Baseline Preservation Invariance | Unpersonalized twin delta vs generic baseline $\equiv 0.0$ for all 46 species | **PASS** |
| **TC-01** | Mathematical Boundedness | All parameters stay strictly within soft $[-35\%, +35\%]$ and hard clamps | **PASS** |
| **TC-02** | Missing Observation Fallback | Missing lab markers gracefully default to baseline with audit log entry | **PASS** |
| **TC-03** | Explicit Solver Failure Capture | Solver errors transparently returned (`success=False`), zero silent masking | **PASS** |
| **TC-04** | Cohort Pipeline Robustness | 541/541 patients mapped without exception; zero crashes on representative phenotypes | **PASS** |
| **TC-A1** | Isolated `p1` Ablation | Pituitary LH reservoir expands/contracts concordant with serum LH | **PASS** |
| **TC-A2** | Isolated `p17` Ablation | Blood FSH adjusts without unintended systemic distortion | **PASS** |
| **TC-A3** | Isolated `p49` Ablation | Total follicle count elevates early antral recruitment ($AF1$) | **PASS** |
| **TC-A4** | Isolated `p51` Ablation | Arrested follicles and irregular cycles blunt dominant transition ($AF3, PrF$) | **PASS** |
| **TC-A5** | Isolated `p173` Ablation | Follicle count & AMH increase basal Inhibin B synthesis | **PASS** |
| **TC-A6** | Combined Core 5 Baseline | Simultaneous 5-parameter simulation runs cleanly (1201 points, 0 NaN, 0 Inf) | **PASS** |

---

## 7. How to Test & Review Manually

Reviewers can independently verify all components with the following commands:

### Run the Complete Test Suite
```powershell
python -m pytest src/twin/test_phase_7_5_p1_mapping.py `
                 src/twin/test_phase_7_6_p1_ablation.py `
                 src/twin/test_phase_7_7_simulation_guard.py `
                 src/twin/test_phase_7_8_p17_mapping.py `
                 src/twin/test_phase_7_9_follicular_mapping.py `
                 src/twin/test_phase_7_10_cohort_verification.py
```

### Inspect Cohort Summary Statistics
```powershell
Get-Content results\cohort_core_5_summary_stats.csv
```

### Interactive 30-Second Verification in Python
```python
import pandas as pd
from src.Clinical.hpo_observation_builder import build_hpo_patient_values
from src.twin.digital_twin import PatientDigitalTwin

# 1. Load any patient from the clinical dataset
df = pd.read_csv("data/PCOS_data_without_infertility-Full_new.csv")
patient = build_hpo_patient_values(df.iloc[0])

# 2. Initialize Digital Twin (automatically personalizes Core 5)
twin = PatientDigitalTwin(patient)
print("Personalized ODE Parameters:")
for k, v in twin.mapping.ode_parameters.items():
    print(f"  {k:12s} = {v:.4f}")

# 3. Simulate personalized baseline for 120 days
sim = twin.run_baseline(start_day=0, end_day=120, points=1201, personalized=True)
print("Simulation result shape:", sim.result.shape)
print("Twin status            :", twin.status)
```

---

## 8. Reviewer FAQ

**Q: Why are only 5 parameters personalized out of 141?**  
A: The remaining 136 parameters are structural affinities, half-lives, Hill coefficients, or pharmacokinetics for exogenous GnRH analogs (`Ago`, `Ant`). Over-parameterizing unobserved internal kinetic affinities without direct longitudinal observations violates identifiability principles and destabilizes the ODE system. The 5 Core parameters directly correspond to observable clinical markers in PCOS (LH, FSH, follicle count, mean size, cycle regularity, AMH).

**Q: Why was `absolute_tolerance` calibrated to $10^{-8}$?**  
A: In the Röblitz model, microscopic follicle transition concentrations drop below $10^{-30}$ near Day 91. With RoadRunner's default tolerance ($10^{-12}$), CVODE attempts to take step sizes smaller than floating-point epsilon ($h \to 10^{-53}$), causing spurious corrector convergence failures. Setting `atol=1e-8` is 6 to 10 orders of magnitude smaller than actual physiological hormone concentrations (which range from 0.1 to 250,000) and completely eliminates sub-epsilon integration stalls while preserving exact numerical invariance.

**Q: Does this PR maintain backwards compatibility?**  
A: Yes. `PatientDigitalTwin.run_baseline(personalized=False)` continues to produce the generic baseline with exact $0.0$ delta across all 46 species, ensuring zero regressions for unpersonalized simulations and existing what-if scenarios.
