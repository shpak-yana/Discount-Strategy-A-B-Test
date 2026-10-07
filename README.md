# Discount Strategy A/B Testing — End-to-End Analytics Project

## 📌 Overview

This project evaluates discount strategies for an e-commerce retailer using two complementary experiments on the **Online Retail II** dataset:

1. **A/B test (2 groups)** — does a 5% discount increase revenue per customer, and is the effect statistically detectable?
2. **A/B/C/D test (4 groups)** — which discount level (5%, 10%, 15%) maximizes **net revenue** without deteriorating the guardrail metric (return rate)?

The project demonstrates a full production-grade A/B testing pipeline: data cleaning, heavy-tail diagnostics, stratified randomization, A/A validation, winsorization, hypothesis testing, bootstrap confidence intervals, power analysis, and multi-comparison correction.

---

## 🎯 Business Questions

### Experiment 1 — A/B Test
> Does a 5% discount increase revenue per customer, and is the effect statistically significant?

### Experiment 2 — A/B/C/D Test
> Which discount level (5%, 10%, 15%) maximizes net revenue per customer without deteriorating the guardrail metric (return rate)?

---

## 📂 Data

- **Source:** [Online Retail II](https://archive.ics.uci.edu/dataset/502/online+retail+ii) (UCI ML Repository), 2009–2011
- **Raw size:** 1,067,371 rows × 8 columns (two sheets combined)
- **After cleaning:** 776,577 rows, 5,852 unique customers with valid IDs
- **Granularity for analysis:** customer-level (5,852 customers)

### Data cleaning steps

1. Combined both sheets (`pd.read_excel(..., sheet_name=None)`).
2. Renamed columns to `snake_case`.
3. Removed **34,335 duplicate rows**.
4. Dropped **235,151 rows with missing `customer_id`** (guest purchases — cannot be tracked across the experiment).
5. Removed **cancellations** (`invoice` starting with `C`).
6. Removed **returns** (`quantity <= 0`) and **zero/negative prices**.
7. Removed **service stock codes** (`POST`, `BANK CHARGES`, `D`, `M`, `TEST001/002`, `PADS`, `ADJUST`, `SP1002`, `DOT`, `CRUK`, `C2`).
8. Created `revenue = quantity × price`.
9. Cast `customer_id` to string.

---

## 📊 Guardrail Metric — Return Rate (Baseline)

Return rate was measured **before** removing cancellations, at three levels:

| Metric | Value |
|---|---|
| Return rate (by invoices) | 16.60% |
| Return rate (by quantity) | 4.42% |
| Return rate (by revenue) | 4.16% |

The three metrics measure different things: **frequency** (invoices), **volume** (quantity), and **financial impact** (revenue). We use revenue-based return rate as the guardrail because it reflects the actual monetary loss.

---

## 🔬 Methodology

### 1. Distribution diagnostics

- Discovered a **heavy-tailed distribution**: only **4.41% of customers** generate **50% of total revenue**.
- Naive random split produced **unbalanced groups** — standard deviation differed by ~1.7× between A and B.
- Decile stratification alone did **not** fix the imbalance, because single "whales" (customers with £500k+ revenue) cannot be split across groups.

### 2. Winsorization + stratification

- **Winsorized** `total_revenue` at the **99th percentile** to cap the influence of extreme outliers (customers above the cap are pulled down to the cap value).
- **Stratified random split** by 50 quantiles of winsorized revenue.
- **A/A test passed** — groups statistically indistinguishable before intervention.

### 3. Experiment design

| Experiment | Groups | Intervention |
|---|---|---|
| A/B | A, B | +5% revenue for B |
| A/B/C/D | A, B, C, D | +5%, +9%, +12% (diminishing returns) for B, C, D |

**Guardrail simulation:** relative return-rate increase of +2%, +5%, +10% for B, C, D respectively.

### 4. Statistical testing

- **Welch's t-test** — does not assume equal variances.
- **Mann-Whitney U** — non-parametric, robust to outliers.
- **Bootstrap 95% CI** (10,000 iterations) — no normality assumption.
- **Power analysis** (statsmodels) — required sample size for detection.
- **Holm-Bonferroni correction** — controls family-wise error rate across multiple comparisons.

---

## 📈 Results — Experiment 1 (A/B Test)

### Descriptive statistics

| Group | n | Mean revenue | Median | Std |
|---|---|---|---|---|
| A | 2,949 | 2,267.08 | 856.03 | 4,147.87 |
| B | 2,903 | 2,385.54 | 856.01 | 4,199.64 |

### Point estimates

| Metric | Value |
|---|---|
| Absolute difference | **+118.46** |
| Relative uplift | **+5.23%** |
| Simulated effect | +5.00% |

### Statistical significance

| Test | Result | Significant? |
|---|---|---|
| Welch's t-test (gross revenue) | p = 0.2900 | ❌ No |
| Mann-Whitney U (gross revenue) | p = 0.1894 | ❌ No |
| Bootstrap 95% CI (net revenue) | [−99.96, 311.55] | ❌ Contains 0 |
| Welch's t-test (return rate) | p = 0.0868 | ❌ No |

### Power analysis

| Parameter | Value |
|---|---|
| Cohen's d | 0.0265 |
| Required n per group | **22,389** |
| Total required | **44,778** |
| Available | 5,852 |

### Interpretation

The simulated **+5% discount effect was not statistically detectable** at the available sample size. The observed uplift (+5.23%) is consistent with random noise. We would need **~7.7× more data** to reliably detect a 5% effect.

**Verdict:** do **not** roll out the 5% discount based on this experiment.

---

## 📈 Results — Experiment 2 (A/B/C/D Test)

### Point estimates (net revenue per customer)

| Group | n | Gross Mean | Return Rate | Net Mean | Uplift vs A |
|---|---|---|---|---|---|
| A (0%) | 1,450 | 2,269.97 | 6% | 2,118.66 | — |
| B (5%) | 1,453 | 2,387.61 | 7% | 2,229.39 | **+5.23%** |
| C (10%) | 1,450 | 2,464.23 | 7% | 2,297.91 | **+8.46%** |
| D (15%) | 1,499 | 2,545.98 | 7% | 2,370.00 | **+11.86%** |

### Statistical significance (Holm-Bonferroni adjusted)

| Comparison | Diff | p_raw | p_adj | Significant |
|---|---|---|---|---|
| A vs B | +110.73 | 0.4570 | 0.4685 | ❌ No |
| A vs C | +179.25 | 0.2342 | 0.4685 | ❌ No |
| A vs D | +251.34 | 0.0980 | 0.2940 | ❌ No |

### Bootstrap 95% CI (net revenue difference vs control)

| Comparison | Mean diff | 95% CI | Contains 0 |
|---|---|---|---|
| A vs B | +114.00 | [−177.03, 410.09] | ✅ Yes |
| A vs C | +176.99 | [−113.34, 478.98] | ✅ Yes |
| A vs D | +246.73 | [−52.57, 546.22] | ✅ Yes |

### Interpretation

- **Point estimates grow monotonically** with discount size (+5.23% → +8.46% → +11.86%), but **no discount is statistically significant** after Holm-Bonferroni correction.
- **All bootstrap 95% CIs include zero** — effects are indistinguishable from random noise at n ≈ 1,450 per group.
- **Group D (15%) is the most promising**: its CI lower bound (−52.57) is closest to zero, and its effect size is the largest.

**Verdict:** do **not** roll out any discount based on this experiment. If a follow-up experiment is run, **test D (15%) on the full customer base** — it has the largest effect size and therefore requires the smallest sample to detect.

---

## 💡 Key Findings

1. **Heavy-tailed revenue distributions break naive A/B testing.** Without winsorization and stratification, random splits produce unbalanced groups and invalid results.
2. **Statistical significance ≠ business significance.** A +5% uplift can look attractive but be undetectable at moderate sample sizes.
3. **Multi-comparison correction matters.** Running 3 pairwise tests without Holm-Bonferroni inflates the family-wise error rate from 5% to ~14%.
4. **Bootstrap and t-test agree** — both indicate the effects are not significant. This cross-validation increases confidence in the conclusion.
5. **Diminishing returns on discounts** are visible in the point estimates: 5% → 5.23%, 10% → 8.46%, 15% → 11.86%.

---

## 🧭 Recommendations

### For this dataset

1. **Do not roll out any discount** — the evidence is insufficient.
2. **If a follow-up experiment is budgeted**, test **D (15%)** on the full customer base. It requires ~4,300 customers per group (vs ~22,000 for B) to detect the effect.
3. **Collect real return-rate data** post-intervention instead of simulating it — return behavior is a critical guardrail.

### Methodological improvements

1. **Apply CUPED** (Controlled-experiment Using Pre-Experiment Data) to reduce variance and increase power without new data.
2. **Model margin impact**, not just revenue — high discounts may reduce contribution margin even when net revenue grows.
3. **Add sensitivity analysis** across multiple effect-size assumptions (linear, diminishing, pessimistic) to check robustness.

---

## ⚠️ Limitations

- Effects are **simulated** on historical data, not measured in a real randomized controlled trial.
- **Guest purchases** (no `customer_id`) were excluded — this may bias the sample toward registered customers.
- **Return rate was simulated** for the guardrail check rather than measured post-intervention.
- **No time-series or cohort effects** were modelled — customer behavior may drift over the two-year period.
- **Holm-Bonferroni is conservative** — it reduces power, so real effects may be missed.

---

## 🛠 Tech Stack

- **Language:** Python 3.14
- **Data:** pandas, NumPy, PyArrow (Parquet)
- **Statistics:** SciPy (`ttest_ind`, `mannwhitneyu`), statsmodels (`TTestIndPower`, `multipletests`)
- **Visualization:** Matplotlib, Seaborn
- **Environment:** JupyterLab
- **Data source:** Online Retail II (UCI ML Repository)

---

## 📁 Project Structure

```
discount-ab-test/
├── data/
│   ├── raw/
│   │   └── online_retail_II.xlsx
│   └── processed/
│       └── customer_df.parquet
├── discount-ab-test-evaluation.ipynb      # A/B test (2 groups)
├── A_B_C_D_test.ipynb                     # A/B/C/D test (4 groups)
├── images/
│   └── abcd_net_revenue.png
├── README.md
└── requirements.txt
```

---

## 🚀 How to Run

```bash
# Clone the repository
git clone https://github.com/shpak-yana/discount-ab-test.git
cd discount-ab-test

# Install dependencies
pip install -r requirements.txt

# Launch JupyterLab
jupyter lab
```

Open `discount-ab-test-evaluation.ipynb` first, then `A_B_C_D_test.ipynb`.

---

## 📚 What This Project Demonstrates

- **End-to-end analytics workflow** — from raw data to business recommendation.
- **Handling heavy-tailed distributions** — winsorization, stratification, robust statistics.
- **A/B testing rigor** — A/A validation, guardrail metrics, power analysis.
- **Multiple comparison correction** — Holm-Bonferroni for multi-armed experiments.
- **Bootstrap confidence intervals** — non-parametric inference.
- **Honest reporting** — when results are inconclusive, the recommendation is "do not act".

---

