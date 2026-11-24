# Credit Risk Portfolio Monitoring, Early Warning Signals & Policy Simulation

An end-to-end credit risk analytics project covering data engineering, exploratory risk segmentation, vintage & roll-rate tracking, behavioral Early Warning Signals (EWS), and underwriting policy cut-off simulation on a **135,000-loan ($7.27B) retail lending portfolio** spanning **2022–2024**.
 
> 📄 **[Download Executive PDF Report](./Credit_Risk_Analysis.pdf)**  
> 📓 **[Explore Full SQL & Python Analysis Notebook](./notebooks/Credit_Risk_Analysis.ipynb)**

---

## 1. Dashboard Overview

### Page 1: Portfolio & Risk Monitoring
*Tracks active exposure, 90+ DPD NPL trajectory against the <5.0% risk appetite threshold, Credit Score × DTI risk matrix, vintage curves by Months on Book (MOB), and monthly roll-forward rates.*

### Page 2: Early Warning Signals & Policy Simulation
*Monitors the high-risk watchlist (21,895 loans / $776.4M exposure), pre-default behavioral deterioration (T-6 to T-0 months), and underwriting cut-off trade-offs between volume retention and loss avoidance.*

---

## 2. Dataset & Data Quality Engineering

Data cleaning, validation, and aggregation were executed in **Python** and **DuckDB (SQL)** across two relational tables:
* **`origination_data`:** 135,000 retail loans disbursed from `2022-01` to `2024-12` (~$7.27B total principal; ~45K loans/year).
* **`monthly_performance`:** 2,355,761 loan-month observations covering 131,136 active borrowers.

**Key Data Governance & Audit Checks:**
* **Integrity & Completeness:** Verified **0 duplicate keys** and **0 null values** across all core credit attributes (`loan_amount`, `dti`, `credit_score`, `dpd_current`, `months_on_book`).
* **Reconciliation of 3,864 Unmatched Loans:** Confirmed that 100% of loans without performance records belong to the `2024-12` origination cohort (first EMI due in `2025-01`), ensuring no orphan records.
* **Data Leakage Prevention:** Explicitly excluded synthetic ground-truth/leakage columns (`deterioration_pattern`, `will_default_original`) prior to constructing clean analytical views (`clean_origination`, `clean_performance`).
* **Delinquency Structure Handling:** Audited DPD buckets and identified direct transitions from `0 DPD` (2,246,724 rows) to `90+ DPD` (109,037 rows) with no intermediate 30/60 DPD delinquency flags. Consequently, **3-month behavioral deltas** (`credit_score_delta_3m`, `revolving_util_delta_3m`) were engineered as primary Early Warning Signals.

---

## 3. Key Analytical Findings

### 1. "L-Shape" Concentration of Credit Risk (Score × DTI Matrix)
* **Safe Core:** Borrowers with `Credit Score ≥ 620` and `DTI ≤ 45%` maintain a highly stable 12-month default rate of **6.89% – 7.42%**.
* **L-Shape Red Zone:** Risk surges sharply along both boundaries:
  * **Subprime (`<620`)** breaches **10.10% – 12.55%** regardless of DTI.
  * **High DTI (`>45%`)** pushes even Prime borrowers (`680–739`) up to **10.45%**.
  * The intersection (**Subprime + High DTI**) hits **12.55%**, roughly **1.8×** the safe core default rate.
* **Weak Discriminators at Origination:** Employment status (`7.37%–7.76%`), loan term (`7.56%–7.76%`), origination utilization (`7.31%–7.76%`), and 12M inquiries (`7.54%–7.81%`) show flat default profiles and do not meaningfully separate risk.

### 2. Absence of Risk-Based Pricing
* Interest rate distributions (~12.0% average APR across all vintages) overlap nearly 100% between defaulting and performing borrowers. High-risk segments are underpriced relative to their expected credit loss (ECL).

### 3. Critical Seasoning Window (Vintage Analysis)
* Cumulative defaults accelerate steeply between **MOB 6 and MOB 12** across all three vintages (`2022`, `2023`, `2024`) before plateauing after MOB 12.
* The **2023 vintage** exhibits slightly elevated risk (**7.88%** 12M default) compared to 2022 (**7.59%**) and 2024 (**7.57%**).
* Portfolio-level NPL remains above the **<5.0% Risk Appetite Target**, though monthly 0-to-90+ DPD roll-forward rates have trended downward from a ~0.7% peak (early 2023) to ~0.23% (late 2024).

### 4. Pre-Default Behavioral EWS (T-6 to T-0 Months)
* **Credit Score Deterioration:** Average credit scores of defaulters erode from **~680 (T-6)** to **~660 (T-0)**, with the steepest 3-month drop (**>10 points**) peaking at **T-2 months before default**.
* **Utilization Spike:** Average revolving utilization climbs from **~30% to ~35%**, with the 3-month utilization jump peaking above **+3.0%** at **T-2 months**.
* Currently, **21,895 active loans (16.7% of portfolio)** representing **$776.4M Exposure at Risk** sit in the high-risk L-Shape watchlist.

---

## 4. Underwriting Policy Simulation

Simulated tightening strategies against the **7.68% Baseline NPL** to evaluate the trade-off between approval volume sacrificed and credit loss avoided:

| Policy Scenario | Volume Rejected (%) | Volume Retained (%) | Simulated NPL (%) | Loss Avoided (%) |
| :--- | :---: | :---: | :---: | :---: |
| **0. Baseline (Current Policy)** | 0% | 100% | 7.68% | 0.00% |
| **A. Cut Subprime (<620) + High DTI (>45%)** | 1% | 99% | 7.64% | 1.27% |
| **B. Cut Full L-Shape Red Zone (<620 OR >45%)** | 17% | 83% | **7.16%** | **22.34%** |

* **Key Takeaway:** Cutting only the extreme corner (Scenario A) saves just **1.27%** of losses. Cutting the full L-Shape Red Zone (Scenario B) avoids **22.34%** of credit losses while retaining **83%** of loan volume (exceeding the `>80%` retention target) and reducing NPL by **52 bps** (`7.68% → 7.16%`).

---

## 5. Strategic Recommendations

1. **Adopt Tiered Underwriting & Risk-Based Pricing:** Implement **Scenario B** cut-offs for auto-rejection of high-risk L-Shape profiles, or apply risk-adjusted APR tiers (+200–400 bps) for borderline segments (`Score 620–679` with elevated DTI) to align yield with expected loss.
2. **Deploy Behavioral EWS Triggers at T-3 / T-2:** Because origination profiles overlap heavily across the mid-tier portfolio and 30/60 DPD flags are absent, integrate automated monthly monitoring for **`3M Score Drop > 8–10 pts`** and **`3M Utilization Jump > 2.5%–3.0%`** to initiate proactive collection/restructuring **2–3 months ahead of default**.
3. **Intensive Monitoring During MOB 5–12:** Concentrate early-bucket account management and limit controls during the critical MOB 6–12 seasoning window where the majority of cumulative defaults materialize.

---

## 6. Repository Structure & Tech Stack

```text
├── Credit_Risk_Analysis.pbix         # Interactive 2-page Power BI report & DAX model
├── Credit_Risk_Analysis.pdf          # Static 2-page executive dashboard export
├── notebooks/
│   └── Credit_Risk_Analysis.ipynb    # DuckDB SQL & Python EDA, Vintage, EWS & Simulation
├── data/
│   ├── 01_portfolio_summary.csv      # Monthly portfolio KPIs & Score × DTI matrix
│   ├── 02_vintage_analysis.csv       # Cumulative default curves by vintage & MOB
│   ├── 03_roll_rate.csv              # Monthly 0-to-90+ DPD roll-forward trend
│   ├── 04_ews_signals.csv            # Pre-default behavioral metrics (T-6 to T-0)
│   └── 05_policy_simulation.csv      # Underwriting cut-off scenario outputs
└── images/
    ├── page1_portfolio_monitoring.png
    └── page2_ews_policy.png
