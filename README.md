# CytoCARs — CAR-T Cytotoxicity Prediction Tool

CytoCARs predicts whether a Chimeric Antigen Receptor (CAR) construct will exhibit **High** or **Low** cytotoxicity, based solely on the amino acid sequences of its five protein domains.

The model uses [ESM-2](https://huggingface.co/facebook/esm2_t33_650M_UR50D) protein language model embeddings combined with biophysical features, fed into an optimized XGBoost classifier (**Test AUC-ROC = 0.838**).

## Quick Start

### 1. Open the notebook in Google Colab

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/)

Upload `CytoCARs_Inference_Notebook.ipynb` to Google Colab, or click the badge above and open it from GitHub.

### 2. Upload the model files

When prompted (Cell 2), upload the three required files:

| File | Description |
|------|-------------|
| `final_xgb_model.joblib` | Trained XGBoost classifier |
| `data_scaler.joblib` | Feature scaler (StandardScaler) |
| `model_columns.joblib` | Expected feature column names |

### 3. Run the prediction

The notebook supports two modes:

#### Option A — Single CAR prediction
1. Run **Cell 4a** to load a pre-configured example, or fill in **Cell 4b** with your own sequences.
2. Run **Cell 5** to execute the prediction pipeline.
3. Run **Cell 6** to display the result.

#### Option B — Batch prediction from Excel
1. Prepare an Excel file (`.xlsx`) following the format described below.
2. Run **Cell 7** and upload your file.
3. Run **Cell 8** to view the styled results table. A predictions Excel file will also be generated for download.

## Input Format

### Single prediction

Five amino acid sequences, one per CAR domain:

| Domain | Description |
|--------|-------------|
| **Peptide_Signal** | Signal peptide |
| **scFv** | Single-chain variable fragment (antigen binding domain) |
| **Hinge** | Hinge region |
| **TM** | Transmembrane domain |
| **Tail** | Intracellular signaling domain |

Valid amino acid characters: `A C D E F G H I K L M N P Q R S T V W Y *`

### Batch prediction (Excel)

The Excel file must contain the following columns (names must match exactly):

| Column name | Content |
|-------------|---------|
| `Construct ID` | Unique identifier for each construct (recommended) |
| `Peptide_Signal_(Protein)` | Signal peptide sequence |
| `AntigenBindingDomain_(Protein)` | scFv / antigen binding domain sequence |
| `Hinge_(Protein)` | Hinge region sequence |
| `TM_(Protein)` | Transmembrane domain sequence |
| `Tail_(Protein)` | Intracellular signaling domain sequence |

An example file (`example_batch_input.xlsx`) is provided in this repository for reference.

## Output

- **Predicted Cytotoxicity**: `High` or `Low`
- **Confidence Score**: probability that the construct has high cytotoxicity (0–100%)

For batch predictions, the output Excel file includes three additional columns: `Validation_Status`, `Predicted_Cytotoxicity`, and `Confidence_Score_High`.

## Requirements

The notebook installs all dependencies automatically (Cell 1). See `requirements.txt` for the full list.

A **GPU runtime** is recommended in Google Colab for faster ESM-2 embedding computation (Runtime > Change runtime type > T4 GPU).

## How It Works

1. **Sequence validation** — checks for valid amino acid characters
2. **Biophysical feature extraction** — computes length, molecular weight, isoelectric point, aromaticity, instability index, and hydrophobicity (GRAVY) for each domain using BioPython
3. **ESM-2 embeddings** — generates 1280-dimensional protein embeddings per domain using Meta's ESM-2 (650M parameter model)
4. **Prediction** — the pre-trained XGBoost classifier outputs the cytotoxicity class and confidence score

## Repository Contents

```
CytoCARs_Inference_Notebook.ipynb   # Prediction notebook (Google Colab)
final_xgb_model.joblib              # Trained XGBoost model
data_scaler.joblib                  # Fitted StandardScaler
model_columns.joblib                # Feature column blueprint
example_batch_input.xlsx            # Example Excel file for batch prediction
requirements.txt                    # Python dependencies
README.md                           # This file
```

## Citation

If you use CytoCARs in your research, please cite the associated publication.

## License

*To be defined.*
