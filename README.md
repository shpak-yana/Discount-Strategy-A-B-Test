# Discount Strategy A/B Test — Online Retail II

## Business Question

Does a discount strategy increase revenue per customer, or does it merely shift revenue without creating real value? This project evaluates a simulated discount experiment on the Online Retail II 
dataset using a full A/B testing pipeline.

## Data

- **Source:** Online Retail II (UCI / Kaggle), 2009–2011
- **Raw size:** 1,067,371 rows × 8 columns (two sheets combined)
- **After cleaning:** 779,425 rows, 5,878 unique customers
- **Granularity for analysis:** customer level (5,852 customers)

### Data cleaning steps

1. Combined both sheets (`pd.read_excel(..., sheet_name=None)`).
2. Renamed columns to `snake_case`.
3. Removed 34,335 duplicate rows.
4. Dropped 235,151 rows with missing `customer_id` (guest purchases).
5. Removed cancellations (`invoice` starting with `C`).
6. Removed returns (`quantity <= 0`) and zero/negative prices.
7. Created `revenue = quantity × price`.
8. Cast `customer_id` to string.

### Guardrail metric: return rate (baseline)

| Metric | Value |
|---|---|
| Return rate (by invoices) | 17.61% |
| Return rate (by quantity) | 4.49% |
| Return rate (by revenue) | 6.24% |

## Methodology

### 1. Distribution diagnostics
- Discovered a heavy-tailed distribution: **4.41% of customers generate 50% of revenue**.
- Random split produced unbalanced groups (std differed by ~1.7×).
- Simple decile stratification did not fix the imbalance because single "whales" cannot be split across groups.

### 2. Winsorization + stratification
- Capped `total_revenue` at the 99th percentile (winsorization) to reduce the influence of extreme outliers.
- Stratified random split by 50 quantiles of winsorized revenue.
- **A/A test passed:** Mann-Whitney p = 0.97, Welch's t-test p = 0.96.

### 3. Discount simulation
- Applied a simulated **+5% revenue lift** to group B.
- Simulated a **+2% relative increase in return rate** for group B as a guardrail stress test.

### 4. Statistical testing
- **Welch's t-test** (does not assume equal variances).
- **Mann-Whitney U** (non-parametric, robust to outliers).
- **Bootstrap** (10,000 iterations) for 95% confidence intervals.
- **Power analysis** (statsmodels) to estimate required sample size.

## Results

| Metric | Group A | Group B | Diff | Uplift |
|---|---|---|---|---|
| Gross revenue (mean) | 2267.08 | 2385.54 | +118.46 | **+5.23%** |
| Return rate (mean) | 6.19% | 6.38% | +0.19 pp | +3.00% |
| Net revenue (mean) | — | — | +106.39 | — |

### Statistical significance

| Test | p-value | Significant? |
|---|---|---|
| Welch's t-test (gross revenue) | 0.2900 | ❌ No |
| Mann-Whitney (gross revenue) | 0.1894 | ❌ No |
| Welch's t-test (return rate) | 0.0868 | ❌ No |
| Bootstrap 95% CI (net revenue) | [−99.96, 311.55] | ❌ Contains 0 |

### Power analysis

| Parameter | Value |
|---|---|
| Cohen's d | 0.0265 |
| Required n per group | **22,389** |
| Total required | **44,778** |
| Available | 5,852 |

## Conclusions

1. **The simulated +5% discount effect was not statistically detectable** at the current sample size. The observed uplift (+5.23%) is consistent with random noise.
2. **Bootstrap 95% CI for net revenue includes zero** ([−99.96, 311.55]), meaning we cannot rule out a null or even negative effect.
3. **Power analysis shows we would need ~7.7× more data** (≈22,389 customers per group) to reliably detect a 5% effect.
4. **Guardrail metric (return rate) did not deteriorate significantly** (p = 0.0868), but the point estimate suggests a possible +3% relative increase.

## Recommendation

**Do not roll out the 5% discount based on this experiment.** 
The evidence is insufficient: the confidence interval is wide, the effect size is tiny (Cohen's d = 0.0265), and we cannot rule out a null effect.

### Recommended next steps

1. **Increase sample size** — run the experiment on the full customer base for a longer period.
2. **Test a stronger intervention** — e.g. 10% or 15% discount, where the effect may be detectable with the current sample.
3. **Apply CUPED** (Controlled-experiment Using Pre-Experiment Data) to reduce variance and increase power without new data.
4. **Run an A/B/C/D test** to compare multiple discount levels simultaneously.

## Limitations

- The experiment is **simulated** on historical data, not a real randomized controlled trial. Real-world behavior may differ.
- Guest purchases (no `customer_id`) were excluded, which may bias the sample toward registered customers.
- Return rate was simulated for the guardrail check rather than measured post-intervention.
- Only customer-level aggregation was used; no time-series or cohort effects were modelled.

## Tech Stack

- Python 3.14 (pandas, NumPy, SciPy, statsmodels)
- Jupyter Notebook
- Data source: Online Retail II (UCI ML Repository)

## How to run

```bash
pip install -r requirements.txt
jupyter notebook discount-ab-test-evaluation.ipynb
```
