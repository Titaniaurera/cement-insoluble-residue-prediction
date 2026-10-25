# Insol Prediction for PCC Cement (Mill CM01)

Predicts the daily **Insol** (insoluble residue, %) of Portland Composite Cement from its chemical
composition (SiO2, Al2O3, Fe2O3, CaO, MgO, SO3, Na2O, K2O, P2O5), residue and fineness (Blaine),
plus lagged values. The whole workflow is time-aware: no random splits, no leakage from test into training.

## Approach

1. **Data prep**: rebuild real dates from the sheet's month/day columns, convert `--` to missing, set
   physically impossible values (CaO < 40, Al2O3 < 1) to missing.
2. **Chronological 80/20 split.** KNN imputation and scaling are fitted on train only.
3. **EDA** on train: distributions, Shapiro-Wilk, correlation (levels and differenced), VIF, seasonality, ACF.
4. **Features**: lags (1, 2, 3, 5, 10 days) of the 8 core oxides.
5. **Model selection** with `TimeSeriesSplit` on train across 9 regressors; the winner is compared with naive baselines on test.
6. **EWMA residual correction**: recent residuals correct the next prediction; span chosen by CV on train.
7. **Improved correction**: tuned Ridge/Lasso/ElasticNet, out-of-fold residuals, shrinkage factor, Wilcoxon + block-bootstrap significance, Ljung-Box.
8. **Experiments**: Insol lags as features, periodic retraining, Huber regression.
9. **Inference**: next-day prediction and per-row prediction from an hourly QC CSV.

In the original run the Lasso + EWMA(7) model reached MAE ~ 0.56 and R2 ~ 0.79 on the test period, and
Huber + EWMA(7) reached MAE ~ 0.55 (not a statistically significant difference). Re-run the notebook to reproduce.

## Repository layout

```
insol_prediction.ipynb   the full pipeline (sections 1 to 13)
requirements.txt
data/                    put the input files here (git-ignored)
outputs/                 figures, predictions, saved model (git-ignored)
```

## Run it

```bash
pip install -r requirements.txt
jupyter notebook insol_prediction.ipynb
```

Input files (not included, git-ignored):

| File | Description |
|------|-------------|
| `data/Kualitas semen 2021-2026.xlsx`, sheet `CM01PCC` | daily lab data: `BULAN`, `Tgl.`, the 11 feature columns, `Insol` |
| `data/QC_hourly.csv` *(optional)* | hourly QC export with `SamplingLocalTime` and the oxide columns |
| `data/data_baru.xlsx` *(optional)* | days after the last date in the main sheet, with actual `Insol` |

Different location? Set `INSOL_DATA_DIR` (and optionally `INSOL_OUTPUT_DIR`) before launching Jupyter.
On Google Colab, uncomment the Drive lines in the configuration cell.

## Important caveats

- EWMA correction needs the previous days' actual Insol at prediction time. If the lab reports with a delay, set `LAB_DELAY_DAYS`.
- Predictions are only as good as the history: if the last known day is more than ~3 days before the target day, lags and correction are stale (the notebook warns you).
- Per-row (hourly) values mean "the daily Insol to expect if the whole day looked like this hour". Compare with the lab value only through their mean.



# Insol Prediction for PCC Cement (Insilo 2, sheet CM02PCC)

Predicts the daily **Insol** (insoluble residue, %) of Portland Composite Cement from its chemical
composition (SiO2, Al2O3, Fe2O3, CaO, MgO, SO3, Na2O, K2O, P2O5), residue and fineness (Blaine),
plus lagged values. The whole workflow is time-aware: no random splits, no leakage from test into training.

## Approach

1. **Data prep**: rebuild real dates from the sheet's month/day columns, convert `--` to missing. Optionally set
   physically impossible values (CaO < 40, Al2O3 < 1) to missing (`APPLY_DOMAIN_RULES`).
2. **Chronological 80/20 split.** KNN imputation and scaling are fitted on train only.
3. **EDA** on train: distributions, Shapiro-Wilk, correlation (levels and differenced).
4. **Features**: lags (1, 2, 3, 5, 10 days) of the 8 core oxides, giving 51 features.
5. **Model selection** with `TimeSeriesSplit` on train across 9 regressors; the winner is compared with naive baselines on test.
6. **EWMA residual correction**: recent residuals correct the next prediction; span chosen by CV on train.
7. **Improved correction**: tuned Ridge/Lasso/ElasticNet, out-of-fold residuals, shrinkage factor, Wilcoxon + block-bootstrap significance, Ljung-Box.
8. **Final model**: Huber regression + EWMA(7), measured once on the test set (MAE, RMSE, R2, % days with |error| <= 1).
9. **Inference**: next-day prediction and per-row prediction from an hourly QC CSV.

Results depend on your data: run the notebook and read the table in section 10.

## Repository layout

```
insol_prediction.ipynb   the full pipeline (sections 1 to 13)
requirements.txt
data/                    put the input files here (git-ignored)
outputs/                 figures, predictions, saved model (git-ignored)
```

## Run it

```bash
pip install -r requirements.txt
jupyter notebook insol_prediction.ipynb
```

Input files (not included, git-ignored):

| File | Description |
|------|-------------|
| `data/new data semen.xlsx`, sheet `CM02PCC` | daily lab data: `BULAN`, `Tgl.`, the 11 feature columns, `Insol` |
| `data/QC_hourly.csv` *(optional)* | hourly QC export with `SamplingLocalTime` and the oxide columns |
| `data/data_baru.xlsx` *(optional)* | days after the last date in the main sheet, with actual `Insol` |

Different location? Set `INSOL_DATA_DIR` (and optionally `INSOL_OUTPUT_DIR`) before launching Jupyter.
On Google Colab, uncomment the Drive lines in the configuration cell.

## Running Insilo 2

Change only these three lines in the configuration cell:

```python
SILO = "insilo2"
EXCEL_FILE = DATA_DIR / "<your insilo 2 file>.xlsx"
SHEET_NAME = "<sheet name>"
```

Saved models and figures are named after `SILO`, so the two silos never overwrite each other.

## Using the saved model

Section 13 writes `outputs/model_<silo>_huber_ewma7.joblib`. It holds the model, the imputer, the column order and the daily history:

```python
import joblib
bundle = joblib.load("outputs/model_insilo1_huber_ewma7.joblib")
model, columns = bundle["model"], bundle["feature_columns"]
```

## Important caveats

- EWMA correction needs the previous days' actual Insol at prediction time. If the lab reports with a delay, set `LAB_DELAY_DAYS` (section 9).
- Predictions are only as good as the history: if the last known day is more than ~3 days before the target day, lags and correction are stale (the notebook warns you).
- Per-row (hourly) values mean "the daily Insol to expect if the whole day looked like this hour". Compare with the lab value only through their mean.
- The baseline correction (sections 7, 8, 10) feeds back the residual of the *corrected* prediction, so it only closes about half of a constant bias. Section 9 uses raw residuals and closes all of it. Inference (section 12) uses raw residuals.


