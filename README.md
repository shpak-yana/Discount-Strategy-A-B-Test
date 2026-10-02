# Discount Strategy A/B Test

End-to-end evaluation of a **simulated discount strategy** on the Online Retail II dataset.  
Customer-level randomization → causal inference with proper statistical testing → segment-level analysis with multiple-testing correction.

> **Important:** Treatment effects are **simulated** for demonstration and learning purposes. The pipeline, statistical methods and decision framework are production-ready.

---

## Business Question

Would offering a moderate discount increase **revenue per customer** enough to justify the potential rise in return rate?

We evaluate three scenarios (weak / medium / strong) and answer:
- Does the discount move the primary metric (RPC)?
- What happens to conversion and AOV?
- Does the guardrail (return rate) stay within acceptable limits?
- Which customer segments benefit most / least?

---

## Key Results (Medium Scenario)

| Metric                    | Control     | Treatment   | Absolute Diff | Relative Uplift | Significant? |
|---------------------------|-------------|-------------|---------------|-----------------|--------------|
| **Revenue per customer**  | —           | —           | small         | ≈ 0%            | No           |
| **Conversion Rate**       | —           | —           | +             | **+12%**        | Yes          |
| **AOV (converted)**       | —           | —           | ≈ 0           | ≈ 0%            | No           |
| **Return rate (guardrail)** | —         | —           | **+1.0 pp**   | —               | Guardrail risk |

**Main takeaway:**  
A moderate discount **lifts conversion** but does **not** produce a statistically significant increase in revenue per customer. At the same time the return rate rises by ~1 percentage point — a clear guardrail risk.  
→ Discounting may be useful for acquisition / reactivation campaigns, but is not an efficient lever for pure revenue growth.

**Segment insights:**
- Effect is heterogeneous across UK vs non-UK, new vs returning and value tiers.
- Multiple testing controlled with **Holm-Bonferroni** correction.

*(Exact numbers are available in the notebooks and `reports/` folder.)*

---

## Experiment Design

| Component              | Choice                                                                 |
|------------------------|------------------------------------------------------------------------|
| **Randomization unit** | Customer (`customer_id`)                                               |
| **Split**              | 50/50 deterministic hash (`md5(SALT + customer_id)`)                   |
| **Primary metric**     | Revenue per customer (RPC)                                             |
| **Secondary metrics**  | Conversion Rate, Average Order Value (AOV of converters)               |
| **Guardrail**          | Return rate (threshold: +1 pp)                                         |
| **Scenarios**          | weak (+5% conv, −2% AOV), medium (+12% conv, 0% AOV), strong (+20% conv, +3% AOV) |

---

## Statistical Methods

- **Welch’s t-test** — primary metric (RPC) and AOV (unequal variances)
- **Two-proportion z-test** — conversion rate
- **Bootstrap confidence intervals** (10 000 resamples)
- **Post-hoc power analysis** + Minimum Detectable Effect (MDE)
- **Holm-Bonferroni** correction for segment-level multiple testing

All tests are two-sided, α = 0.05.

---

## Project Structure

```
Discount-Strategy-A-B-Test/
├── data/
│   ├── raw/                          # Place online_retail_II.xlsx here
│   └── processed/                    # Clean transactions, customers, experiment tables (parquet)
├── notebooks/
│   ├── 01_data_cleaning_and_experiment_design.ipynb
│   ├── 02_ab_test_analysis.ipynb
│   └── 03_segmentation_and_impact.ipynb
├── reports/
│   ├── ab_test_results_medium.csv
│   ├── segments_summary.csv
│   └── segments_for_dashboard.csv
├── requirements.txt
└── README.md
```

---

## How to Reproduce

1. **Clone the repository**
   ```bash
   git clone https://github.com/shpak-yana/Discount-Strategy-A-B-Test.git
   cd Discount-Strategy-A-B-Test
   ```

2. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

3. **Get the data**  
   Download [Online Retail II](https://archive.ics.uci.edu/dataset/502/online+retail+ii) (or the Kaggle version) and place `online_retail_II.xlsx` into `data/raw/`.

4. **Run the notebooks in order**
   ```bash
   jupyter notebook notebooks/01_data_cleaning_and_experiment_design.ipynb
   jupyter notebook notebooks/02_ab_test_analysis.ipynb
   jupyter notebook notebooks/03_segmentation_and_impact.ipynb
   ```

The first notebook produces three experiment files (`experiment_weak/medium/strong.parquet`).  
Notebooks 02 and 03 focus on the **medium** scenario by default (the most realistic one).

---

## Tech Stack

- **Python 3.10+**
- pandas, NumPy
- SciPy, statsmodels (Welch t-test, z-test, power analysis)
- seaborn / matplotlib
- Jupyter, Parquet, openpyxl

---

## Dataset

**Online Retail II** (UCI / Kaggle)  
~1 million transactions from a UK-based online gift retailer (2009–2011).  
After cleaning: cancellations removed, guest customers dropped, service codes excluded, top 1% of largest orders filtered out → **~785k clean transactions**, customer-level table used for the experiment.

---

## What This Project Demonstrates

- Proper experimental design (customer-level randomization, pre-registered metrics & guardrails)
- Full causal analysis pipeline from raw data to decision recommendation
- Correct handling of multiple testing when looking at segments
- Clear communication of statistical results to business stakeholders
- Reproducible research (deterministic split, fixed seeds, parquet outputs)
