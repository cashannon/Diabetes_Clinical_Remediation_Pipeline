# Diabetes Clinical Data Remediation Pipeline  
## Team Project: Automated Data Quality Enhancement for Readmission Risk Modeling

**Team Members:** Christian Shannon, Ashley Love, Mugtaba Awad, and Kristian Livingston

**Presentation Link:**  
https://github.com/cashannon/Diabetes_Clinical_Remediation_Pipeline/blob/main/reports/Diabetes%20Clinical%20Remediation%20Pipeline%20Presentation.mp4
---

## 1. Project Overview

This project delivers a modular Python pipeline that audits, remediates, and validates the Diabetes 130-US Hospitals dataset (101,766 encounters) to create a high-fidelity asset for predicting hospital readmission risk.  
The pipeline targets systemic data quality issues—sentinel nulls, non-standard ICD-9 codes, and inconsistent encodings—using automated auditing, MICE-based imputation, and regex normalization, improving the overall Data Quality Index (DQI) while preserving clinically realistic distributions.  

Key dataset notes:  
- 101,766 rows, 53+ columns after enrichment, with extreme missingness in weight and payer code.  
- Diabetes 130-US Hospitals dataset (UCI), representing a decade of inpatient encounters.  
- Includes demographics, utilization metrics (e.g., time in hospital, procedures, medications), admission/discharge codes, and readmission outcomes.  

---

## 2. Clinical Problem & Impact

### Clinical Problem

Clinical data often contain high rates of missing values, placeholder codes (e.g., `?`), and non-standard diagnostic encodings, which degrade the validity of readmission models.  
The team hypothesized that a reproducible remediation pipeline would outperform manual cleaning by systematically improving completeness, coded consistency, and duplicate handling before any predictive modeling.  

### Impact on Readmission Analytics

Using the remediated dataset as a modeling-ready asset:  
- Baseline DQI ≈ **0.93**, Final DQI ≈ **1.00** in the integrated pipeline notebook (≈7.35 point gain).  
- Example DQI component lift (from validation notebook): completeness from **0.72 → 0.90**, coded consistency from **0.88 → 0.94**, duplicates from **0.95 → 0.98**.  
- Downstream models on remediated data show improved discrimination; e.g., XGBoost ROC-AUC rises to **≈0.74** with balanced performance across classes.  

These improvements make readmission risk scores more trustworthy for targeting high-risk patients while avoiding artifacts driven by missing or malformed records.  

---

## 3. Data Quality Pipeline

The pipeline is organized into three core phases, each with dedicated Python components.

### 3.1 Phase 1 – Clinical Audit & Baseline DQI

Implemented in `wrangler_baseline_audit_RAW.py` via the `DataAuditor` class.  

Key steps:  
- **Load & De-duplicate**  
  - Load raw CSV, track raw duplicate count, and remove exact duplicate encounters.  

- **Sentinel & Unknown Handling**  
  - Convert `?`, `Unknown`, `Not Available`, `NULL`, and similar variants to proper `NaN`.  
  - Clean age/weight “bracket” encodings into simpler strings for downstream processing.  

- **Clinical ID Enrichment**  
  - Map numeric `admissiontypeid`, `dischargedispositionid`, and `admissionsourceid` to human-readable descriptions (e.g., “Emergency”, “Discharged to home”).  

- **Baseline Clinical Profile & DQI**  
  - Generate demographics, utilization, and readmission distributions.  
  - Compute DQI as an average of:
    - Completeness (mean non-null ratio across columns).  
    - Coded consistency (valid categorical values for gender, readmitted, ID fields).  
    - Duplicates penalty (share of duplicate rows).  

### 3.2 Phase 2 – Advanced Remediation (MICE & ICD-9 Normalization)

Implemented via `DataRemediator` (in `remediator.py`) and orchestrated in:  
- `scientist_mice_remediation_RAW.py` (`DataRemediatorPipeline`)  
- `wrangling_feature_engineering_v1_RAW.ipynb`  

Core logic:  
- **Numeric Imputation with MICE/KNN**  
  - Exclude identifiers (`encounterid`, `patientnbr`) from any imputation.  
  - Standardize numeric features with `StandardScaler`, apply MICE-style imputation, then inverse transform to original units.  
  - Preserve extremely sparse fields (e.g., weight, payer code, medical specialty) to avoid bias from guesswork.  

- **ICD-9 Code Normalization (Regex)**  
  - Normalize `diag1`, `diag2`, and `diag3` using regex rules to standardize ICD-9 formats and reduce spurious category explosion.  
  - Example: reduction in unique diagnosis codes (e.g., `diag1` from 700+ unique codes down to a smaller, standardized set).  

- **Target Engineering for Readmission**  
  - Map `readmitted` string labels (`NO`, `>30`, `<30`) to numeric (0/1).  
  - Create `readmit30days` binary indicator for 30-day readmission.  

### 3.3 Phase 3 – Visual & Statistical Validation

Implemented in `wrangling_feature_engineering_v1_RAW.ipynb` and described in the whitepaper.  

Validation steps:  
- Side-by-side descriptive statistics (count, mean, median, min, max) before vs. after imputation for key numeric features (e.g., `nummedications`, `numlabprocedures`, `numberinpatient`).  
- Overlaid KDE plots showing pre- vs. post-remediation distributions closely aligned, confirming that MICE preserved medical plausibility.  
- DQI “before vs. after” bar chart showing consistent gains across completeness, coded consistency, and duplicate handling.  

---

## 4. Modeling Approach (Downstream Example)

While this repository focuses on remediation, the notebooks provide a reference pipeline for predictive modeling using the remediated data.  

### 4.1 Problem Framing & Splits

- Binary classification: 30-day readmission (`readmit30days` = 1 vs. 0).  
- Stratified train–test split (80/20) to preserve class imbalance.  

### 4.2 Feature Engineering & Encoding

- **Numeric features:**  
  - Admissions/utilization metrics (time in hospital, procedures, medications, counts of inpatient/outpatient/emergency visits, number of diagnoses, etc.).  

- **Categorical features:**  
  - Race, gender, age bracket, weight, payer code, medical specialty, ICD-9 codes, medication families, and enriched admission/discharge/source descriptions.  

- **Encoding:**  
  - `ColumnTransformer` + `StandardScaler` for numeric features.  
  - `OneHotEncoder(handle_unknown='ignore')` for categorical features.  

### 4.3 Models & Performance (Illustrative)

From `wrangling_feature_engineering_v1_RAW.ipynb`:  

- **Logistic Regression – Baseline**  
  - Accuracy ≈ 0.56, ROC-AUC ≈ 0.57.  

- **Logistic Regression – Class-Balanced + One-Hot**  
  - Accuracy ≈ 0.64, ROC-AUC ≈ 0.70.  

- **Random Forest**  
  - Accuracy ≈ 0.63, ROC-AUC ≈ 0.69.  

- **XGBoost (tuned)**  
  - Accuracy up to ≈ 0.68, ROC-AUC up to ≈ 0.74, with more balanced precision/recall across classes.  

These experiments demonstrate that systematic remediation yields more stable and informative models compared to training on the raw, noisy dataset.  

---

## 5. Technical Stack

Core libraries and tools:  

- **Data & Pipelines**  
  - `pandas`, `numpy` for data shaping and profiling.  
  - Custom `DataAuditor` and `DataRemediator` classes for modular reuse.  

- **Imputation & Scaling**  
  - `sklearn.impute` (MICE-style via `IterativeImputer`), `StandardScaler`.  

- **Modeling & Evaluation**  
  - `scikit-learn`: `LogisticRegression`, `RandomForestClassifier`, `train_test_split`, ROC-AUC, confusion matrices, classification reports.  
  - `xgboost.XGBClassifier` for gradient-boosted trees.  

- **Visualization**  
  - `matplotlib`, `seaborn` for KDEs, histograms, correlation heatmaps, and DQI component plots.  

---

## 6. Ethical & Governance Considerations

Summarized from the whitepaper and appendix:  

- **Data Privacy & PII**  
  - Uses a de-identified public clinical dataset; no direct patient identifiers are exposed in the pipeline.  

- **Clinical Validity & Safety**  
  - Imputation constrained to medically plausible ranges; KDE overlays verify distributions are not distorted by remediation.  
  - High-missingness features (e.g., weight) are preserved rather than aggressively imputed to avoid misleading clinical inference.  

- **Transparency & Reproducibility**  
  - Clear DQI framework (completeness, coded consistency, duplicates) and explicit mapping of each remediation step to its impact on data quality.  
  - Pipeline is modular and auditable, supporting external review and extension to other specialties (e.g., cardiology, oncology).  

- **Decision Support, Not Automation**  
  - The remediated dataset is intended to power risk stratification and decision-support, with clinicians retaining final judgment on interventions.  

---

## 7. Repository Structure

A representative layout based on the project files:

```text
.
├─ README.md                          # This file
├─ data/
│  ├─ diabeticdata.csv                # Diabetes 130-US Hospitals dataset
│  ├─ IDS_mapping.csv
│  └─ diabetic_data_cleaned.csv       # Clean Diabetes 130-US Hospitals dataset
├─ notebooks/
│  ├─ wrangling_feature_engineering_v1_RAW.ipynb   # Full E2E remediation + modeling
│  ├─ Integrated_Remediation_Pipeline.ipynb        # Single-call pipeline & DQI lift
│  ├─ wrangler_baseline_audit_RAW.py      # DataAuditor: baseline audit & DQI
│  ├─ remediator.py                       # DataRemediator: MICE + ICD-9 rules
│  ├─ scientist_mice_remediation_RAW.py   # DataRemediatorPipeline wrapper
│  ├─ DataAuditor.py                      # Packaged auditor class
│  └─ visualizer.py                       # Plot utilities for DQI & distributions
├─ reports/
│  ├─ Diabetes Clinical Remediation Pipeline Presentation.mp4 # Recorded Presentation
│  └─ Diabetes-Clinical-Remediation-Pipeline.pdf # Presentation
└─ config/
   └─ _config.yml                      # Path and parameter settings
