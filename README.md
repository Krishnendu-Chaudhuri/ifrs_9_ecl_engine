# IFRS 9 Expected Credit Loss (ECL) Engine

**Lending Club Accepted Loans (2007–2018)**  
Research / validation prototype implementing the IFRS 9 three-stage impairment model on public consumer unsecured loan data.

> **Status:** Reproducible research notebook — **not** a bank-production approval package.  
> Recovery timing, macro calibration, representativeness, independent monitoring and governance sign-off remain necessary before any production use.

---

## Overview

This notebook builds an end-to-end IFRS 9 Expected Credit Loss engine on the Lending Club accepted tape (~2.26 million loans). It covers:

| Stage | Content |
|-------|---------|
| **1** | Setup, single-pass data load, data quality, status mapping, development vs live-book split |
| **2** | Origination PD model, empirical hazard term structure, out-of-time metrics |
| **3** | Recovery-based LGD, EAD path (contractual annuity), SICR staging |
| **4** | Loan-level ECL measurement + one-factor Vasicek scenario overlay |
| **5** | Sensitivity, held-out vintage back-test, feature drift, reasonableness checks, executive summary, assumptions & model-risk register |

**Snapshot date:** 2018-12-31  
**Live book (reporting date):** ~912.6 k loans | Total EAD ≈ $9.56 bn  
**Probability-weighted ECL:** ≈ $632 m (coverage ≈ 6.61%)

---

## Regulatory anchors (IFRS 9)

- §5.5.1–5.5.5 / B5.5.1–B5.5.20 — scope and three-stage model  
- B5.5.22–B5.5.24 — definition of credit-impaired (Stage 3)  
- B5.5.15–B5.5.17 — significant increase in credit risk (SICR) and 30-DPD rebuttable presumption  
- B5.5.44–B5.5.48 — discount rate (EIR approximated by contractual interest rate)

---

## Data

**Source:** Lending Club accepted loans CSV (`accepted_2007-18.csv`)

**Key design choices**

- Single full-data read of all pipeline columns (including recovery fields).  
- Footer / non-numeric `id` rows dropped.  
- Required recovery columns: `funded_amnt`, `total_rec_prncp`, `recoveries`, `collection_recovery_fee`.  
  **No constant-LGD fallback is permitted.**  
- Dates parsed as `%b-%Y`. Interest rate and revol utilisation converted from percent strings.

**Portfolio split**

| Dataset | Definition | Purpose |
|---------|------------|---------|
| `df_dev` | Resolved-Good + Resolved-Bad | PD / LGD development |
| `df_train` | Issue 2009–2011 (maturity-filtered) | Model fitting |
| Hold-out 2012 | 36-month loans, excluded from fitting | Independent historical validation |
| OOT test | Issue 2013–2014 | Discrimination & calibration metrics |
| `df_live` | Live-* statuses at 2018-12-31 | ECL measurement |

**Maturity rules (right-censoring control)**  
- 36-month loans: issue ≤ 2014-12  
- 60-month loans: issue ≤ 2013-12  

---

## Methodology summary

### PD

- **Main model:** Logistic regression (sklearn Pipeline) with median imputation + standard scaling.  
- Features exclude grade and interest rate to reduce circularity with underwriting.  
- Empirical grade-only benchmark retained for reference.  
- Lifetime PD + monthly conditional hazard term structure derived from observed default timing.  
- Default month ≈ `last_pymnt_d` + charge-off lag (default 4 months).  
- Current PD on the live book updated via last-FICO substitution only.

### LGD

- Recovery-based: net recoveries = recoveries − collection recovery fee, discounted by an assumed recovery lag.  
- Segment estimates by grade × term; portfolio mean used only for thin cells.  
- Bounded to [0, 1]; 45 % retained solely as an unsecured benchmark, never as a fallback.

### EAD

- Contractual annuity amortisation from remaining term.  
- No prepayment, cure, redraw or payment-irregularity model (upper-bound path).

### Staging (SICR)

| Stage | Rule |
|-------|------|
| **3** | Credit-impaired proxy: Late (31–120) or Default status |
| **2** | SICR: lifetime PD / origination PD ≥ 2× **or** absolute PD increase ≥ 50 bp, **or** 30-DPD backstop (Grace / Late 16–30) |
| **1** | All other performing exposures |

### ECL & scenarios

- Stage 1: 12-month ECL  
- Stage 2 / 3: lifetime ECL  
- Discounting at contractual interest rate  
- One-factor Vasicek transform applied to **monthly conditional hazards** (not directly to lifetime PD)  
- Illustrative scenarios (CONFIG): Baseline (z = 0), Optimistic, Adverse / Severe  
- Probability-weighted ECL = Σ scenario probability × scenario ECL  
- Asset correlation ρ = 0.15 (illustrative; Basel retail range 0.03–0.16)

---

## Key results (live book at 2018-12-31)

| Metric | Value |
|--------|-------|
| Live loans | 912,609 |
| Total EAD | $9,562,889,216 |
| Mean current lifetime PD | 16.08 % |
| Mean LGD | 92.9 % |
| Stage 1 share of EAD | 94.1 % |
| Stage 2 share of EAD | 3.3 % |
| Stage 3 share of EAD | 2.6 % |
| Baseline scenario ECL | $399 m (coverage 4.17 %) |
| **Probability-weighted ECL** | **$632 m (coverage 6.61 %)** |
| Forward-looking uplift | +58.4 % |
| Stage 1 coverage | 4.05 % |
| Stage 2 coverage | 11.29 % |
| Stage 3 coverage | 93.92 % |
| OOT AUC (2013–2014) | 0.649 |
| OOT calibration slope | 0.896 |

Stage coverage ordering is respected (Stage 1 < Stage 2 < Stage 3).

---

## Repository layout

```
.
├── ifrs9_ecl_final.ipynb      # Main notebook (this engine)
├── data/
│   └── accepted_2007-18.csv  # Lending Club accepted tape (user-supplied)
├── figures/                   # Exploratory & diagnostic plots
├── output/                    # Drift report, tables, etc.
└── README.md
```

Configure paths in the single `CONFIG` dictionary at the top of the notebook:

```python
CONFIG = {
    "DATA_PATH": Path("data/accepted_2007-18.csv"),
    "FIG_DIR": Path("figures"),
    "OUTPUT_DIR": Path("output"),
    "SNAPSHOT_DATE": pd.Timestamp("2018-12-31"),
    "RANDOM_SEED": 42,
    "RHO": 0.15,
    # ... maturity rules, SICR thresholds, LGD bounds, scenarios, etc.
}
```

---

## How to run

1. Place `accepted_2007-18.csv` under `data/` (or update `CONFIG["DATA_PATH"]`).  
2. Open `ifrs9_ecl_final.ipynb` in Jupyter / VS Code.  
3. Run all cells top-to-bottom.  
4. Outputs (figures, drift CSV, executive summary) are written to `figures/` and `output/`.

**Requirements (typical):**  
`pandas`, `numpy`, `scikit-learn`, `scipy`, `matplotlib`, `seaborn`

The notebook uses a memory-aware single-pass load and chunked ECL calculation (`CHUNK_SIZE = 200_000`) so it can run on a laptop with ~16 GB RAM.

---

## Validation highlights

- **Held-out 2012 vintage** (36-month) excluded from PD fitting and hazard-shape estimation; used for independent discrimination / calibration evidence.  
- **Out-of-time test** on 2013–2014 issues.  
- **Live-vintage feature drift** (2015–2018) via PSI and absolute standardized mean difference; PSI ≥ 0.25 flagged for review (not automatic model failure).  
- **Sensitivity** on LGD (±10 %), ρ, SICR thresholds, and Late-31 staging treatment.  
- **Reasonableness checks:** coverage by grade, stage-ordering, status × stage cross-tab.  
- **Assertions** at the end of Stage 5 reconcile executive-summary numbers with underlying tables and enforce key design invariants (no 2012 leakage, recovery columns present, LGD in [0, 1], etc.).

---

## Known limitations & model risk

1. Public Lending Club tape — not a regulated bank’s internal data; underwriting / collection / recovery practices differ.  
2. No continuous DPD; stage assignment relies on status-label proxies.  
3. No external macro time series; scenario z-values are transparent stress anchors, not GDP/unemployment-calibrated forecasts.  
4. Recovery cash-flow dates unavailable → single assumed recovery lag; recoveries may be incomplete at cut-off.  
5. Static / segment LGD; no cure model, collateral, or downturn LGD estimated from data.  
6. Contractual amortisation overstates future EAD for loans that prepay.  
7. Current PD updates FICO only; other risk drivers frozen at origination.  
8. Single-country, consumer unsecured book — no multi-product or sovereign correlation structure.

A full **Assumptions Register** and **Self-Assessment** table are produced inside the notebook (Stage 5e–5f).

---

## Citation / academic use

This is a teaching and research prototype suitable for:

- IFRS 9 / expected-credit-loss methodology illustration  
- Empirical finance / credit-risk coursework  
- Validation and model-risk case studies  

It is **not** intended for regulatory capital, financial reporting, or production impairment calculations without substantial additional work on data quality, macro linkage, governance and independent review.

---

## License & data notice

Lending Club data is subject to the terms of the original data release.  
This notebook is provided for educational and research purposes.
