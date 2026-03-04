# CytoCARs — Predicting CAR-T Cytotoxicity from Protein Sequences

## Project Overview

CytoCARs is a machine learning project that predicts the **cytotoxic potential ("High" or "Low") of Chimeric Antigen Receptor (CAR) constructs** based solely on the amino acid sequences of their constituent protein domains. It uses ESM-2 protein language model embeddings combined with biophysical features, fed into an XGBoost classifier.

The project consists of two Google Colab notebooks:
- **`CytoCARs_Traning_Notebook_23_01_2026.ipynb`** — Full training pipeline (135 cells)
- **`CytoCARs_Inference_Notebook.ipynb`** — End-user prediction tool (20 cells)

## Architecture

### Data
- **Input:** Excel file (`Matrice_CAR_constructs_V2.xlsx`) with 101 CAR constructs, each described by 5 protein domain sequences
- **Domains:** `Peptide_Signal`, `scFv` (antigen binding), `Hinge`, `TM` (transmembrane), `Tail` (intracellular signaling)
- **Target:** Binary classification — `High` (1) vs `Low` (0) cytotoxicity (57 High, 44 Low)

### Feature Engineering
- **ESM-2 embeddings:** `facebook/esm2_t33_650M_UR50D` (650M params) generates 1280-dim embeddings per domain → 6400 features total
- **Biophysical features:** 6 properties per domain via BioPython (`length`, `mol_weight`, `pI`, `aromaticity`, `instability_idx`, `gravy`) → 30 features
- **Categorical:** `scFv_family` one-hot encoded (Short/Standard/Long based on scFv length) → 3 features
- **Total:** ~6433 features

### Model
- **Champion model:** XGBoost optimized via Optuna (100 trials)
- **Performance:** Test AUC-ROC = 0.838, High-class recall = 92%
- **Calibration:** Isotonic calibration via `CalibratedClassifierCV`
- **Comparison models tested:** Logistic Regression, Random Forest, MLP, k-NN

### Saved Artifacts (produced by training, consumed by inference)
| File | Content |
|------|---------|
| `final_xgb_model.joblib` | Trained XGBoost classifier |
| `data_scaler.joblib` | Fitted StandardScaler |
| `model_columns.joblib` | Ordered feature column names |
| `best_hyperparameters.joblib` | Optuna best hyperparameters |
| `processed_car_data_with_embeddings.pkl` | Pre-processed DataFrame (alternate entry point) |

## Training Pipeline (Training Notebook)

1. **Data loading** from Excel, column name cleaning
2. **Sequence validation** — valid amino acids: `ACDEFGHIKLMNPQRSTVWY*`
3. **Anomaly detection** — outlier sequence lengths via 3-sigma threshold
4. **Manual corrections** — known parsing error for `pRRL.SIN.EF1A.JAG1-F1.GS.CAR-3G` (TM/Tail concatenation)
5. **Biophysical feature extraction** — BioPython's `ProteinAnalysis`
6. **scFv family categorization** — Short (≤200), Standard (201-300), Long (>300)
7. **ESM-2 embedding generation** — batch_size=16, mean pooling over non-padding tokens
8. **Data split** — stratified 60/20/20 (train/val/test), `random_state=42`
9. **Scaling** — `StandardScaler` fit on train only
10. **Baseline models** — LogReg, XGBoost default, SMOTE augmentation, PCA reduction, Random Forest
11. **Optuna hyperparameter optimization** — 100 trials, 600s timeout
12. **Final model training** — XGBoost with best params, n_estimators=2000, early_stopping_rounds=30
13. **Calibration** — isotonic, 3-fold CV
14. **SHAP interpretation** — TreeExplainer, global/local feature importance
15. **Nested cross-validation** — 5-fold outer, 3-fold inner GridSearchCV

### Dual Execution Paths
- **Path A:** Full pipeline from raw Excel data (Cells 2–14a)
- **Path B:** Load `processed_car_data_with_embeddings.pkl` at Cell 14b, skip feature engineering

## Inference Pipeline (Inference Notebook)

### Single Prediction (Option A)
1. Enter 5 domain sequences (pre-configured examples or manual input via Colab forms)
2. Validate and clean sequences
3. Compute biophysical features + scFv family + ESM-2 embeddings
4. Align to `model_columns` (reindex with fill_value=0), scale, predict
5. Display styled HTML result (green = High, orange = Low) with confidence score

### Batch Prediction (Option B)
1. Upload Excel with columns: `Peptide_Signal_(Protein)`, `AntigenBindingDomain_(Protein)`, `Hinge_(Protein)`, `TM_(Protein)`, `Tail_(Protein)`, `Construct ID`
2. Same feature pipeline applied row-by-row
3. Output: styled HTML table + downloadable Excel file with predictions

## Naming Conventions

| Pattern | Example | Description |
|---------|---------|-------------|
| `biophys_{domain}_{property}` | `biophys_scFv_length` | Biophysical features |
| `emb_{domain}_{index}` | `emb_Hinge_927` | ESM-2 embedding dimensions |
| `family_{category}` | `family_Short` | One-hot encoded scFv family |

**Note:** The inference notebook uses `AntigenBindingDomain` in batch mode vs `scFv` in single mode. The `model_columns` + `reindex(fill_value=0)` pattern handles alignment.

## Key Dependencies

```
biopython          # Biophysical feature calculation
transformers[torch] # ESM-2 protein language model
xgboost            # Classifier
scikit-learn       # Preprocessing, metrics, cross-validation
imbalanced-learn   # SMOTE (training only)
optuna             # Hyperparameter optimization (training only)
shap               # Model interpretability (training only)
pandas, numpy      # Data manipulation
matplotlib, seaborn # Visualization (training only)
xlsxwriter         # Excel export (inference only)
torch              # PyTorch backend for ESM-2
```

## Conventions

- **`random_state=42`** used everywhere for reproducibility
- **Bilingual:** Code is primarily English with some French in print statements and variable names
- **Strict data hygiene:** Scaler/PCA fit on train only, SMOTE on train only, test set evaluated once
- **No PCA in final pipeline:** Explicitly removed — full embeddings perform better
- **Colab-specific:** Uses `google.colab.files` for upload/download, `@param` forms for UI
- **Defensive programming:** Variable existence checks, try/except, validation status tracking
- **Idempotent cells:** Cleanup logic before recreation (e.g., drop `biophys_*` columns before recomputing)

## Key Findings (from SHAP analysis)

- **Hinge** and **scFv** embedding dimensions are the most predictive features
- Classic biophysical features do not appear in top-20 importance
- The model prioritizes recall for "High" class (minimizing missed effective candidates)
