# Which Degrees Still Pay?

**A program-level decision-support tool for university program portfolio review**


Pragun Kalra ·  Adonay Kfleyesus · Akshaya Jonnalagadda · Santiago · Chelsea Chen

---

## Overview

Universities usually decide which degree programs to **open, expand, hold, or restructure** based on enrolment demand and departmental advocacy. Neither tells you how graduates actually do.

This project builds an outcome-based alternative. It joins three federal data sources and compares programs at the **field of study × credential level** grain across four dimensions:

- **Earnings** — median earnings 4 years after completion
- **Student debt** — median federal debt and debt-to-earnings
- **Occupational growth** — BLS 2024–34 projections for the destination occupation
- **AI exposure** — how far AI tools have already entered the destination occupation's work (O\*NET)

**The central obstacle:** about 75% of the earnings column is privacy-suppressed by the Department of Education for small programs. Rather than drop or impute those rows, we train a model on the reported programs and **estimate earnings for the suppressed two-thirds of the catalogue**.

## Key findings

1. **Credential level is the biggest lever, but its value depends on the field.** One extra credential level is worth about **+$12,341** in STEM, **+$9,185** in business/health/trades, and **+$8,146** in the humanities and social sciences.
2. **Field matters far more than institution.** Median earnings differ by **$68.4k** between the top field (Engineering) and the bottom one (Culinary & Personal Services). Program and occupation inputs account for **66.8%** of random-forest importance, against **21.6%** for institution inputs.
3. **AI exposure goes with growth, not risk.** Among programs with early-career pay above $70k, high-exposure programs show **29%** median 1→5-year earnings growth against **10%** for low-exposure ones. Low-exposure programs are **3.2× more likely** to be on a declining trajectory (9.3% vs 2.9%). *This is a descriptive association, not a causal estimate.*
4. **The hidden two-thirds is now comparable.** **100,336** suppressed programs were scored, so coverage goes from **33.4% to 100%**. **31** field × credential cells had no reported program at all.

## Data

**Source:** [College Majors 2026: Earnings, Debt, Jobs & AI](https://www.kaggle.com/datasets/kylefengkfeng209/college-majors-2026-earnings-debt-jobs-ai) on Kaggle (`college_majors_2026.csv`, 227,980 rows × 72 columns).

One row = one institution × one field of study (4-digit CIP) × one credential level.

| Source | Contributes |
|---|---|
| [College Scorecard](https://collegescorecard.ed.gov/data/) (U.S. Dept. of Education) | Graduate earnings at 1, 4 and 5 years; federal debt; monthly payment; employment |
| [BLS Employment Projections](https://www.bls.gov/emp/) / [OEWS](https://www.bls.gov/oes/) | Occupation median wage, 2024–34 growth, annual openings |
| [O\*NET](https://www.onetonline.org/) | Software tools used per occupation, including AI tools and expert systems |

### Cleaning funnel (227,980 → 150,565 rows, 66%)

| Step | Rows removed | Why |
|---|---|---|
| Drop duplicate `program_id` | 0 | Checks that the file is clean instead of assuming it |
| U.S. institutions only | 4,662 | Foreign programs aren't a U.S. university's decision and would skew benchmarks |
| Keep 6 standard credential levels | 16,782 | Non-credential and post-bacc certificates aren't "open a program" decisions |
| Must map to a BLS/O\*NET occupation | 4,765 | Without a mapping there's no demand or AI signal |
| Credentials awarded > 0 | 51,206 | Dormant or discontinued programs |
| Drop impossible values | 0 | Cheap sanity check |

After cleaning, **50,229** programs have reported earnings (the training set) and **100,336** are privacy-suppressed (the prediction set).

## Methodology

- **Feature/target split by suppression.** Each column was tested on one question: *does it have a value for the suppressed programs?* Outcome columns are 94–100% missing there, while descriptive columns (institution, program, occupation, AI) are about 7% missing. Only never-suppressed columns are used as model inputs.
- **No imputation on the target.** Suppression depends on cohort size (median 35 graduates for reported programs vs 6 for suppressed ones), so filling in the target would fabricate data where it differs most.
- **Outliers kept.** We tested this: field rankings computed with and without IQR outliers have a Spearman ρ of 0.949. High-earning medical, dental and law programs are signal, not noise.
- **Engineered metrics:** `ai_exposure_index` (four O\*NET signals, min-max scaled across the whole frame, then averaged), `wage_catchup_pct`, `pay_vs_national_pct`, `annual_debt_burden`, and `roi_tier` (debt-to-earnings banded into Strong / Moderate / Stretched / At risk).
- **AI analysis.** Programs are compared within bands of 1-year earnings, then the top and bottom AI-exposure quartiles are compared on 1→5-year growth. Matched field × credential pairs (within 5% of early pay) are shown as illustrations.

## Modelling results

Target: `median_earnings_4yr_usd`. Sample: 40,000 programs, 80/20 train/test split, `random_state = 42`.

| Model | R² | RMSE | MAE | vs. benchmark MAE |
|---|---|---|---|---|
| 0 · National median for same field + credential (no model) | 0.696 | $15,087 | $9,528 | — |
| 1 · Linear regression | 0.690 | $15,215 | $10,387 | −9.0% |
| 2 · Linear regression (log target) | 0.704 | $14,873 | $9,884 | −3.7% |
| 3 · Decision tree (depth 4) | 0.520 | $18,934 | $13,383 | −40.5% |
| **4 · Random forest (200 trees)** | **0.826** | **$11,418** | **$7,368** | **+22.7%** |

Only the random forest beats the free benchmark. Its median absolute error is about $5k, and 76.8% of test programs fall within ±$10k.

**Ablation (random forest, adding one source block at a time):**

| Inputs | R² | MAE |
|---|---|---|
| Institution only | 0.313 | $16,628 |
| + Program | 0.706 | $9,746 (still worse than the benchmark) |
| + Occupation (BLS) | 0.821 | $7,468 |
| + AI exposure (O\*NET) | 0.826 | $7,368 |

The model beats doing nothing only after the Dept. of Education data is joined to Dept. of Labor projections. That join is the project's core contribution.

Linear regression is kept to make the results interpretable. It is refit on decorrelated inputs because several pairs correlate above 0.99. It is also fit separately by field group (STEM, humanities & social sciences, business/health/trades) to show that the value of a credential differs by field.

## Recommendations

| Action | Target |
|---|---|
| **Expand** | High-pay, high-AI-exposure fields (Computer Science, Engineering, Mathematics, Business), plus stackable credential pathways |
| **Hold** | Programs in the middle ROI tier, while adding AI tooling to their curriculum |
| **Restructure** | Programs failing the floor test (fewer than 60% of graduates out-earn a high-school graduate), especially those with low AI exposure. Six field × credential cells covering about 215k credentials a year |

These are screening and prioritisation signals, not causal claims.

## Repository contents

```
WhatDegreesStillPay.ipynb   # full analysis: cleaning, EDA, AI analysis, models, estimates, recommendations
README.md
```

### Notebook structure

| § | Section |
|---|---|
| 1–2 | Business context, decision thresholds, setup |
| 3 | Data understanding and missingness |
| 4 | Six-step cleaning funnel |
| 5 | Feature/target split and types of missingness |
| 6 | Outlier test |
| 7 | Feature engineering (including two documented bug fixes) |
| 8 | EDA: earnings distribution, credential ladder, field spread, field × credential interaction |
| 9 | AI exposure: definition, pay-banded growth test, matched pairs |
| 10 | Model ladder, AI sensitivity check, ablation, linear and grouped regression, decision tree, feature importance, error analysis |
| 11 | Scoring the suppressed programs and building the decision table |
| 12–13 | Business insights and recommendations |
| 14–16 | Limitations, future work, reflection on AI tools |

## How to run

The notebook was built in **Google Colab** and is easiest to run there.

1. Open `WhatDegreesStillPay.ipynb` in Colab (or Jupyter with Python 3).
2. Run all cells. The loader installs `kagglehub` if needed and downloads the dataset automatically. Public datasets usually don't need Kaggle credentials.
3. If the Kaggle download fails, upload `college_majors_2026.csv` to the runtime (`/content/` or the working directory) and rerun the loading cell.

**Dependencies:** `pandas`, `numpy`, `matplotlib` (3.10), `seaborn` (0.13), `scikit-learn`, `kagglehub`

```bash
pip install pandas numpy matplotlib seaborn scikit-learn kagglehub
```

The full run is memory-heavy: the raw file is about 360 MB in memory, and several 200-tree random forests are trained. A standard Colab runtime is enough.

## Limitations

- **Association, not causation.** Students self-select into fields, and ability and preparation aren't observed.
- **Missing-not-at-random target.** The model is trained on larger programs and extrapolated to smaller ones. The inputs of the two groups are similar (destination wage and growth are close), but the estimates are least reliable where they are most novel. Use ROI tiers, not point estimates.
- **Single snapshot.** A structural decline can't be told apart from a cyclical dip.
- **AI exposure is a proxy.** O\*NET records whether tools are present in an occupation, not displacement risk.
- **Median of medians.** Field-level figures understate the spread within a field.
- **ROI-tier denominator.** Debt-to-earnings exists for only 38,460 programs (25.5% of the frame). "2.9% at risk" means 2.9% *of programs with debt and earnings data*.

## Future work

- Validate the hidden-program estimates against state unemployment-insurance wage records
- Add gradient boosting and quantile regression for real prediction intervals
- Model completion rate together with earnings
- Rebuild the AI index on task-level exposure instead of tool counts
- Reweight the sample to correct for size-driven suppression
