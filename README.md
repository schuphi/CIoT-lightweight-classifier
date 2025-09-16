# Lightweight Multiclass Intrusion Detection on IoT (CIC-IoT2023)

This project trains and evaluates **lightweight tree-based models** to classify **33 cyberattack types (+ benign)** from IoT network traffic. Achived similar discrimination as DNNs with tree ensembles for strong accuracy–efficiency trade-offs on edge devices, that cannot handle heavy deep nets.

## Overview
- **Problem:** Multiclass intrusion detection on IoT traffic (packet/window features).
- **Dataset:** Balanced subset of CIC-IoT2023 (≈ **100k** rows; ~3k samples per class; two minority classes completed with **SMOTE**).
- **Models:** Logistic Regression (baseline), **Decision Tree**, **Random Forest**, **XGBoost**.
- **Split:** **70/15/15** train/validation/test with stratification (no leakage).
- **Primary metric:** **Macro F1** (class-agnostic), plus accuracy, precision, recall.

## Results (test set, macro averages)
- **Logistic Regression:** F1 ≈ **0.58**
- **Decision Tree:** F1 ≈ **0.66**
- **Random Forest:** F1 ≈ **0.69–0.70**
- **XGBoost:** F1 ≈ **0.71** (best)

## Methodology & Pipeline

### 1) Preprocessing
- **Duplicates:** Remove exact duplicates (1,363 rows) **before** any split.
- **Missing values:** Median imputation on training statistics (few rows).
- **Feature selection:** 
  - **VIF** analysis to drop near-collinear features (e.g., Tot size, AVG, LLC, IPv with infinite VIF).
  - **Random Forest feature importance** to retain high-signal, high-VIF features (e.g., Header_length, IAT, Rate) and drop low-importance, high-VIF ones.
- **Transforms & scaling:** 
  - Log1p for features with |skew| > 0.75.
  - **StandardScaler** for all numeric features (fit on train only).
- **Outliers:** **Isolation Forest ∩ Local Outlier Factor** on top-importance features; drop points flagged by **both** (~1.95% of train).
- **Categoricals:** One-hot encode `Protocol Type`.

### 2) Imbalance handling
- Targeted **SMOTE** on **train only** for underrepresented classes (e.g., `Recon-PingSweep`, `Uploading_Attack`). Validation/test remain untouched.

### 3) Modeling & tuning
- **Baseline:** Logistic Regression.
- **Tree-based models:** Decision Tree, Random Forest, XGBoost.
- **Search:** `RandomizedSearchCV`, **5-fold stratified** CV, maximize **macro F1**.
- **Selected configs (best found):**
  - **Decision Tree:** `max_depth=20`, `min_samples_split=8`, `min_samples_leaf=1`, `max_features≈0.86`
  - **Random Forest:** `n_estimators=200`, `max_depth=20`, `min_samples_split=5`, `min_samples_leaf=1`, `max_features≈0.34`
  - **XGBoost:** `n_estimators=200`, `max_depth=7`, `min_child_weight=3`, `learning_rate=0.05`, `subsample=0.8`, `colsample_bytree=0.6`

## Reproducing

### Environment
Python **3.10+** recommended.

```bash
python -m venv .venv && source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -U pandas numpy scikit-learn xgboost imbalanced-learn matplotlib
```

## Data 
The original CIC-IoT2023 files are very large (PCAP-derived).
This repo relies on a balanced CSV subset built via stratified sampling + SMOTE (not included due to size).
If you rebuild:
Extract features from PCAPs into tabular form (see paper for the 39-feature schema).
Stratified sample ~3k rows per class (or as available).
Apply the preprocessing steps above strictly inside a train/val/test split to avoid leakage.

## Limitations
Residual confusions between attacks with similar network signatures; synthetic samples may introduce noise; XGBoost’s training time is higher.

