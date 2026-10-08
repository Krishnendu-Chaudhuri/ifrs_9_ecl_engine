# IFRS 9 Expected Credit Loss (ECL) — Lending Club Analysis

End-to-end **IFRS 9 Expected Credit Loss** engine on public Lending Club–style loan data.  
The notebook walks through the full practitioner workflow: single-loan 12-month ECL → PD term structures → SICR staging → macro scenario weighting → one-factor Vasicek point-in-time (PIT) PDs.

---

## Overview

| Item | Detail |
|------|--------|
| **Scope** | Retail unsecured consumer loans (Lending Club accepted loans 2007–2018) |
| **Standard** | IFRS 9 §5.5 / B5.5 (staging, SICR, probability-weighted scenarios) |
| **Output** | Stage 1 / 2 / 3 ECL, scenario-weighted portfolio ECL, coverage ratios |
| **Language** | Python 3 (pandas, numpy, matplotlib, seaborn) |

**Illustrative results (full tape, ~2.26 M loans):**

| Metric | Value |
|--------|-------|
| Total EAD | ≈ €13.7 bn |
| Stage 1 share of EAD | ~69.5 % |
| Stage 3 share of EAD | ~30.5 % |
| Baseline ECL | ≈ €152 M |
| Probability-weighted ECL | ≈ €315 M |
| Forward-looking uplift | ~107 % |
| Overall coverage (weighted) | ~2.3 % |

---

## Regulatory & methodological anchors

- **TTC PDs** — S&P Global 2024 Default Study (Tables 25/26), mapped from Lending Club grades A–G onto the agency scale.
- **LGD** — CRR Art. 161 senior unsecured floor (45 %).
- **Macro paths** — EBA 2025 EU-wide stress-test style scenarios (baseline / optimistic / adverse).
- **Staging** — IFRS 9 §5.5 / B5.5 SICR waterfall (relative + absolute PD change + 30-DPD backstop).
- **Scenario weighting** — IFRS 9 B5.5.42 probability-weighted expectation.
- **PIT conversion** — Single-factor Vasicek / ASRF:

$$
\mathrm{PD}(z)=\Phi\left(\frac{\Phi^{-1}(\mathrm{PD}_{\mathrm{TTC}})-\sqrt{\rho}\,z}{\sqrt{1-\rho}}\right)
$$

---

## Repository / notebook structure

```
IFRS9_ECL_LendingClub_Complete_Analysis.ipynb
│
├── 1. Setup
│     Paths, RNG, plotting defaults
├── 2. Data — Lending Club portfolio
│     Load accepted_2007-18.csv, clean & map fields
├── 3. Exploratory data analysis
│     Status / grade distributions, basic stats
├── 4. Grade → rating map & TTC PDs
│     S&P-style TTC PDs, LGD, EAD construction
├── 5. 12-month ECL (Phase 1)
│     Single-loan & portfolio 12-month ECL
├── 6. PD term structure (Phase 2)
│     Constant-hazard cumulative PDs by remaining term
├── 7. SICR & staging (Phase 3)
│     Stage 1 / 2 / 3 assignment
├── 8. Macro scenarios (Phase 4)
│     Notch-shift / scenario ECL + probability weighting
├── 9. Vasicek PIT PD (Phase 5)
│     Closed-form normal CDF / PPF, illustrative PIT PDs
├── 10. Portfolio results summary
└── 11. Key take-aways & limitations
```

Figures are written to `figures/` (e.g. `03_scenario_ecl.png`).

---

## Data

**Source:** [Kaggle Lending Club Accepted Loans 2007–2018](https://www.kaggle.com/datasets/wordsforthewise/lending-club)

Place the file as:

```
~/Downloads/lending club data/accepted_2007-18.csv
```

(or adjust `DATA_DIR` in the first code cell).

**Fields used (illustrative):**

`id`, `grade`, `sub_grade`, `fico_range_low`, `dti`, `loan_amnt`, `term`, `int_rate`, `loan_status`, `delinq_2yrs`, `revol_util`, `out_prncp`, `total_rec_late_fee`

After cleaning the notebook retains ~2.26 M rows.

---

## Quick start

```bash
# 1. Clone / download the notebook
# 2. Install dependencies
pip install pandas numpy matplotlib seaborn

# 3. Put accepted_2007-18.csv in the path expected by DATA_DIR
# 4. Open the notebook and run all cells
jupyter notebook IFRS9_ECL_LendingClub_Complete_Analysis.ipynb
```

No external credit-risk libraries are required; normal CDF / inverse-CDF are implemented with Abramowitz–Stegun / Acklam approximations so SciPy is optional.

---

## Core methodology (short)

### 1. Mapping & risk parameters
Lending Club grades **A–G** are mapped onto an S&P-style rating scale.  
Each rating receives a **through-the-cycle (TTC) PD** from the S&P default study.  
**LGD** is held constant at 45 % (senior unsecured).  
**EAD** is taken as outstanding principal (`out_prncp`) or original loan amount where appropriate.

### 2. 12-month ECL
$$
\mathrm{ECL}_{12\mathrm{m}} = \mathrm{PD}_{12\mathrm{m}} \times \mathrm{LGD} \times \mathrm{EAD}
$$

### 3. Lifetime / term-structure PDs
A constant-hazard approximation converts the 12-month PD into a cumulative PD over the remaining contractual term.

### 4. SICR & staging
- **Stage 1** — no significant increase in credit risk → 12-month ECL  
- **Stage 2** — SICR (relative / absolute PD thresholds or 30-DPD) → lifetime ECL  
- **Stage 3** — credit-impaired / default → lifetime ECL at 100 % PD (or observed default status)

### 5. Macro scenarios
Three scenarios (baseline, optimistic, adverse) shift PDs (notch or multiplicative).  
Portfolio ECL is the probability-weighted average of the three scenario ECLs.

### 6. Vasicek PIT overlay
Systematic factor \(z\) implied by the macro scenario converts TTC PDs into forward-looking PIT PDs (illustrative \(\rho = 0.15\)).

---

## Key take-aways

- **Stage 3 dominates ECL** even when it is a minority of EAD — the classic IFRS 9 pattern.
- **Forward-looking scenario weighting** can lift ECL substantially above a pure baseline (exact uplift depends on adverse probability and stress severity).
- The **SICR waterfall** moves a material share of the book into lifetime measurement without waiting for default.
- Mapping consumer grades onto an agency scale allows reuse of published TTC matrices while still producing a coherent IFRS 9 engine.

---

## Limitations (validator-relevant)

| Area | Limitation |
|------|------------|
| Data | Public tape approximates marginals; full joint distribution / vintage effects are not reproduced. |
| PD term structure | Constant-hazard approximation; production models use full cumulative transition matrices. |
| LGD | Static 45 %; no cure rates, collateral or recovery-timing model. |
| Macro → \(z\) | Illustrative mapping; production calibration uses bank-specific default history or official EBA paths more rigorously. |
| Behaviour | No prepayment / revolving CCF modelling. |
| Scope | Unsecured consumer loans only; no wholesale, SME or secured portfolios. |

---

## References

1. IFRS 9 *Financial Instruments* — §5.5 Impairment, Application Guidance B5.5.  
2. S&P Global Ratings — *2024 Annual Global Corporate Default And Rating Transition Study*.  
3. CRR (Regulation (EU) No 575/2013) — Art. 161 LGD.  
4. EBA — EU-wide stress-test methodology (macroeconomic scenarios).  
5. Basel Committee — IRB / ASRF single-factor model (foundation of the Vasicek PIT formula).
