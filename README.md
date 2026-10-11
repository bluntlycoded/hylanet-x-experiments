# HyLANET-X experiments

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.23293486.svg)](https://doi.org/10.5281/zenodo.23293486)

Code and results for an evaluation of HyLANET-X, a hybrid intrusion detection model that combines LightGBM with a neural branch, on four public datasets: CICIDS2017, TON_IoT, IoTID20 and SCADANet.

The question behind the experiments is simple: does adding a neural branch to a tuned gradient-boosting model actually help? To answer it fairly, every model goes through the same leakage-controlled protocol, and the results include tuned tree baselines, five seeds and a time-ordered split.

## What is in this repository

```
notebooks/
  HyLANET_X_experiments.ipynb            the experiment notebook (no outputs)
  HyLANET_X_experiments_executed.ipynb   the notebook as it was run, with all outputs
results/
  tables/        every results table as CSV (and LaTeX)
  figures/       ROC/PR curves, alpha curves, training curves, feature shift; confusion/ has all confusion matrices
  runs/          one JSON per run (dataset, split scheme, seed) with metrics, per-class scores, alpha, timings
  tuning/        hyperparameter search results and all Optuna trials
  transfer/      cross-dataset transfer runs (CICIDS2017 <-> IoTID20)
  splits/        train/validation/test indices for every split, so runs can be repeated exactly
  preds/         test-set predictions for every run
  probs/         test-set class probabilities for the seed-42 runs
  environment.json   software versions and hardware of the run
```

The cleaned datasets themselves are not included. The notebook downloads the original data and rebuilds them.

## Data

| Dataset | Source used | Original release |
|---|---|---|
| CICIDS2017 | Kaggle `dhoogla/cicids2017` (cleaned Parquet release) | https://www.unb.ca/cic/datasets/ids-2017.html |
| TON_IoT | Kaggle `arnobbhowmik/ton-iot-network-dataset` (`train_test_network.csv`) | https://research.unsw.edu.au/projects/toniot-datasets |
| IoTID20 | Kaggle `rohulaminlabid/iotid20-dataset` | https://sites.google.com/view/iot-network-intrusion-dataset |
| SCADANet | Kaggle `ealgul/scada-dataset-v01` | Algül et al., Computer Networks 278 (2026) 112087 |

The notebook downloads all four with `kagglehub`. Public datasets download without a Kaggle key; if Kaggle asks for one, set `KAGGLE_USERNAME` and `KAGGLE_KEY`.

## Protocol

- Each dataset is cleaned once before any split. Identifier columns (IPs, ports, flow IDs, absolute timestamps) are set aside, free-text columns with more than 1,000 values are dropped, and exact duplicates, conflicting labels and classes with fewer than 50 rows are removed. Large classes are capped at 100,000 rows with a fixed seed. Every step is logged in `results/tables/data_construction.csv`.
- Splits are 60% train, 15% validation and 25% test. The random split is stratified and repeated with five seeds (42, 123, 2024, 3407, 7777). For the two datasets with timestamps (IoTID20 and SCADANet) there is also a time-ordered split within each class.
- Preprocessing is fitted on the training set only. Hyperparameters (Optuna), early stopping, the learning-rate schedule and the fusion weight are chosen on the validation set. The test set is scored once.
- Ten models are compared: Random Forest, LightGBM, linear SVM, MLP, CNN-BiLSTM-Tr (the earlier neural stage, on PCA inputs), FTA (the new order-free feature-token attention branch), HyLANET-X (LightGBM + FTA, soft vote), HyLANET-X (stacked), HyLANET-X (CNN design) and RF + LightGBM (a tree-only fusion used as a control).

## Main results

Macro-F1 (%) under the random split, mean ± standard deviation over five seeds (`results/tables/macro_f1_all_datasets_random.csv`):

| Model | CICIDS2017 | TON_IoT | IoTID20 | SCADANet |
|---|---|---|---|---|
| Random Forest | 98.88 ± 0.06 | 95.78 ± 0.11 | 85.29 ± 0.34 | 92.05 ± 0.09 |
| LightGBM | 98.91 ± 0.24 | 95.93 ± 0.11 | 85.28 ± 0.25 | 91.97 ± 0.09 |
| Linear SVM | 90.28 ± 0.31 | 62.16 ± 2.41 | 71.42 ± 0.28 | 88.59 ± 1.00 |
| MLP | 84.41 ± 1.20 | 86.07 ± 0.58 | 79.62 ± 0.08 | 91.63 ± 0.20 |
| CNN-BiLSTM-Tr | 79.32 ± 1.86 | 80.45 ± 1.14 | 73.46 ± 0.92 | 88.07 ± 0.32 |
| FTA | 83.11 ± 1.03 | 87.45 ± 1.61 | 79.77 ± 0.32 | 91.17 ± 1.10 |
| HyLANET-X | 98.93 ± 0.25 | 95.92 ± 0.12 | 85.25 ± 0.28 | 92.05 ± 0.08 |
| HyLANET-X (stacked) | 97.13 ± 0.98 | 95.25 ± 0.27 | 83.91 ± 0.08 | 91.47 ± 0.12 |
| HyLANET-X (CNN design) | 95.49 ± 0.27 | 94.79 ± 0.31 | 82.11 ± 0.23 | 91.66 ± 0.15 |
| RF + LightGBM | 98.96 ± 0.24 | 95.92 ± 0.15 | 85.51 ± 0.39 | 92.15 ± 0.08 |

Macro-F1 (%) under the time-ordered split, five model seeds on one fixed split (`results/tables/macro_f1_all_datasets_temporal.csv`):

| Model | IoTID20 | SCADANet |
|---|---|---|
| Random Forest | 49.34 ± 0.25 | 82.36 ± 0.08 |
| LightGBM | 47.94 ± 0.65 | 85.07 ± 2.93 |
| Linear SVM | 45.94 ± 0.00 | 94.96 ± 0.00 |
| MLP | 45.88 ± 0.58 | 95.98 ± 0.11 |
| CNN-BiLSTM-Tr | 40.03 ± 2.48 | 91.93 ± 0.18 |
| FTA | 46.20 ± 0.85 | 89.30 ± 4.75 |
| HyLANET-X | 46.38 ± 0.70 | 88.94 ± 5.45 |
| HyLANET-X (stacked) | 51.81 ± 1.31 | 88.94 ± 3.61 |
| HyLANET-X (CNN design) | 42.98 ± 0.33 | 93.63 ± 0.68 |
| RF + LightGBM | 49.34 ± 0.25 | 85.53 ± 3.21 |

In short, under random splits HyLANET-X, LightGBM and Random Forest are statistically tied (HyLANET-X minus LightGBM: +0.02 points, Wilcoxon p = 0.40). Under the time-ordered split, the stacked HyLANET-X beats LightGBM in 9 of 10 runs (+3.9 points, p = 0.004). The paired tests are in `results/tables/significance_wilcoxon.csv`. The other analyses, all in `results/tables/`, are:

- complementarity of the two branches
- random vs time-ordered splits
- raw vs PCA inputs
- PCA component order
- cross-dataset transfer
- ROC/PR AUC
- per-class scores

## Running the notebook

1. Open `notebooks/HyLANET_X_experiments.ipynb` in Jupyter on a machine with a GPU. CPU also works, just slower.
2. Run the cells from top to bottom. The first cell installs any missing packages. Every cell prints a line when it finishes.
3. Set `FAST = True` in the configuration cell for a quick end-to-end check. Set it to `False` for the full experiment.

Every run is cached in `hylanet_outputs_v2/`, so the notebook can be restarted after a disconnect and picks up where it stopped. `PARALLEL_WORKERS` (default 1) can be raised to the number of GPUs to train several runs at once.

## Reproducibility notes

- The results here come from a server with two NVIDIA L4 GPUs and 256 CPU threads (Python 3.10, PyTorch 2.11, LightGBM 4.7, scikit-learn 1.7.2, Optuna 5.0; see `results/environment.json`). The executed notebook ran with `PARALLEL_WORKERS = 2`.
- Model size and latency come from `results/tables/efficiency_clean.csv`. To produce it, the seed-42 models were retrained with 16 CPU threads and timed on one CPU thread with nothing else running (section 7b of the notebook). `efficiency.csv` holds the timings measured inside the main runs. Those were taken while other work was running on the machine, so they should not be used.
- The CICIDS2017 release used here has no timestamps and the TON_IoT file has no `ts` column, so these two datasets only have random-split results.

## Citation

The code and results are archived on Zenodo: https://doi.org/10.5281/zenodo.23293486

```bibtex
@software{nayak_reddy_hylanetx_2026,
  author    = {Nayak, Rahul and Reddy, G. Navabharat},
  title     = {{HyLANET-X} experiments: notebooks and results},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.23293486},
  url       = {https://doi.org/10.5281/zenodo.23293486}
}
```

