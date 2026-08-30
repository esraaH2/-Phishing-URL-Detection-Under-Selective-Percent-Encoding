# Leakage-Safe Phishing URL Detection Under Standards-Aware Transformations

A domain-held-out, multi-seed study of clean phishing-URL detection and
robustness to standards-aware URL representation changes.

This repository accompanies the study **“Leakage-Safe Phishing URL Detection
Under Standards-Aware Transformations: A Domain-Held-Out, Multi-Seed
Robustness Study.”** It evaluates four character-level neural networks and two
classical baselines using canonical deduplication, registrable-domain-held-out
splits, training-only preprocessing, validation-only threshold selection, and
paired robustness analysis.

> **Scope.** This is an offline defensive evaluation. The experiment does not
> visit URLs, resolve domains, retrieve page content, submit credentials, or
> interact with live services. The transformations test representation
> sensitivity; they do not prove destination-equivalent end-to-end attacks.

## Main findings

- Cleaning reduced 822,010 raw rows to 799,620 unique labeled URLs after
  removing contradictory labels and canonical duplicates.
- The split contains 471,429 training, 162,900 validation, and 165,291 test
  URLs, with zero canonical-URL and zero registrable-domain overlap.
- Eighteen runs were completed: six models × three seeds (42, 52, and 62).
- CharCNN achieved the best clean domain-held-out phishing F1:
  **0.9613 ± 0.0032**.
- Selective path/query percent-encoding changed 114,034 test URLs
  (**68.99% coverage**).
- Character 3–5-gram TF-IDF logistic regression was the most stable model under
  percent-encoding: mean full-test F1 decrease **0.0329**, changed-subset F1
  decrease **0.0699**, and conditional ASR **9.08%**.
- For all six models, primary-seed changed-subset F1 decreases had
  registrable-domain-cluster confidence intervals excluding zero; Holm-adjusted
  cluster sign-flip p-values were **0.0012**.

The central result is that the best model on clean URLs is not necessarily the
most robust model under representation changes.

## Experimental design

### Dataset

The experiment uses the public
[Phishing and Legitimate URLs](https://www.kaggle.com/datasets/harisudhan411/phishing-and-legitimate-urls)
dataset. The notebook expects `new_data_urls.csv` with `url` and `status`
columns. In the source dataset, `0` denotes phishing and `1` denotes
legitimate; the experiment remaps phishing to positive class `1` for all
metrics.

The raw dataset is not redistributed in this repository. Download or attach it
directly from Kaggle. The executed run records the source-file SHA-256 digest
in `CONFIG.json` and the artifact manifest.

### Leakage controls

1. Unicode NFC normalization and removal of surrounding whitespace/control
   characters.
2. Canonical URL keys used only for duplicate auditing—not as model input.
3. Complete removal of canonical URLs with contradictory labels.
4. Deterministic same-label canonical deduplication.
5. Five-fold `StratifiedGroupKFold` assignment by registrable domain: three
   folds for training, one for validation, and one for testing.
6. Executable assertions for zero URL and domain overlap between partitions.
7. Character vocabulary and all learned preprocessing fitted on training data
   only.
8. Decision thresholds selected on clean validation predictions and frozen
   before test evaluation.

### Models

| Family | Model | Main representation |
|---|---|---|
| Neural | LSTM | Training-only character vocabulary |
| Neural | BiLSTM | Training-only character vocabulary |
| Neural | CharCNN | Parallel character convolutions (3, 5, 7) |
| Neural | Transformer | Positional embeddings and masked self-attention |
| Classical | CharNgramLogistic | Character 3–5-gram TF-IDF + SGD logistic regression |
| Classical | LexicalHistGB | 21 lexical/structural features + histogram gradient boosting |

Neural models use a maximum URL length of 200, 96-dimensional embeddings,
class-balanced training, Adam optimization, early stopping, and a maximum of
15 epochs. Every model is evaluated with seeds 42, 52, and 62.

### Transformations

| Condition | Test URLs changed | Coverage | Newly over 200 characters |
|---|---:|---:|---:|
| Selective path/query percent-encoding | 114,034 | 68.99% | 2,454 |
| DNS host-case control | 163,848 | 99.13% | 0 |
| Explicit default-port control | 37,647 | 22.78% | 7 |
| IDNA host conversion | 4 | 0.0024% | 0 |

Coverage is computed from actual string inequality. Unchanged URLs are not
reported as transformed examples. The adaptive best-of-k condition is retained
as supplementary analysis rather than treated as the primary threat model.

## Results

### Clean domain-held-out performance

Values are means over three seeds.

| Model | Phishing F1 (mean ± SD) | PR-AUC | Recall | MCC |
|---|---:|---:|---:|---:|
| CharCNN | **0.9613 ± 0.0032** | **0.9935** | 0.9608 | **0.9311** |
| LSTM | 0.9605 ± 0.0012 | 0.9923 | 0.9489 | 0.9305 |
| BiLSTM | 0.9579 ± 0.0029 | 0.9923 | 0.9555 | 0.9251 |
| CharNgramLogistic | 0.9431 ± 0.00002 | 0.9858 | 0.9440 | 0.8986 |
| Transformer | 0.9346 ± 0.0038 | 0.9838 | 0.9176 | 0.8856 |
| LexicalHistGB | 0.9007 ± 0.0008 | 0.9711 | 0.8516 | 0.8348 |

### Selective percent-encoding robustness

Values are means over three seeds. “Changed drop” evaluates exactly the URLs
whose string representation changed. Conditional ASR is restricted to clean-
detected phishing URLs that were actually transformed.

| Model | Clean F1 | Transformed F1 | Full F1 drop | Changed F1 drop | Conditional ASR |
|---|---:|---:|---:|---:|---:|
| CharNgramLogistic | 0.9431 | **0.9102** | **0.0329** | **0.0699** | **9.08%** |
| BiLSTM | 0.9579 | 0.9025 | 0.0553 | 0.1246 | 22.50% |
| CharCNN | 0.9613 | 0.8936 | 0.0677 | 0.1486 | 22.22% |
| LSTM | 0.9605 | 0.8826 | 0.0779 | 0.1827 | 31.86% |
| LexicalHistGB | 0.9007 | 0.8110 | 0.0897 | 0.2340 | 41.67% |
| Transformer | 0.9346 | 0.8303 | 0.1043 | 0.2550 | 41.14% |

Exact per-seed values, confidence intervals, raw discordance counts, subgroup
results, calibration metrics, and adjusted p-values are provided in the CSV
tables and full artifact bundle. The tables—not rounded README values—are the
authoritative numerical record.

## Statistical evaluation

The notebook exports:

- accuracy, balanced accuracy, precision, recall, specificity, F1, PR-AUC,
  ROC-AUC, MCC, Brier score, 15-bin ECE, and confusion counts;
- full-test and exactly changed-subset paired effects;
- conditional and unconditional attack success rate with Wilson intervals;
- 5,000-draw row-paired bootstrap intervals;
- 1,000-draw registrable-domain-cluster bootstrap intervals;
- McNemar discordant counts as supplementary row-level evidence;
- 5,000-permutation registrable-domain-cluster sign-flip tests;
- Holm correction across the six models within each transformation;
- URL-length and domain-multiplicity subgroup diagnostics; and
- supplementary paired clean-model comparisons.

Primary inferential analysis uses seed 42. Seeds 52 and 62 quantify training
variability and are not treated as independent datasets.

## Repository layout

```text
.
├── README.md
├── notebooks/
│   └── phishing_url_robustness.ipynb
├   figures/
│   ├── FIG01_Split_Diagnostics.png
│   ├── FIG05_Robustness_Heatmap_ASR.png
│   └── FIG06_Cluster_Bootstrap_F1_Drop.png

```

`LICENSE` and `CITATION.cff` should be added after selecting the intended
license and confirming the final publication metadata. Do not add placeholder
DOIs.

## Reproducing the experiment

### Recommended: Kaggle

1. Create a Kaggle notebook and attach the
   [dataset](https://www.kaggle.com/datasets/harisudhan411/phishing-and-legitimate-urls).
2. Import `notebooks/phishing_url_robustness.ipynb`.
3. Enable a GPU accelerator. The completed reference run used two Tesla T4
   GPUs; a single supported GPU also works but takes longer.
4. Select **Restart & Run All**. Do not execute training cells out of order.
5. Confirm that `REVIEWER_CHECKLIST.json` contains only `true` or zero-valued
   overlap checks.
6. Download `/kaggle/working/phishing_robustness_v3_complete.zip` immediately,
   or use **Save Version** with output files enabled.

The final reference bundle is approximately 282.4 MB. Kaggle's
`/kaggle/working` directory is not durable across all new sessions. Crash-safe
`DONE.json` markers can resume completed model/seed runs only while their run
directories still exist.

### Local execution

The completed reference run used:

- Python 3.12.13
- NumPy 2.0.2
- pandas 2.3.3
- SciPy 1.16.3
- scikit-learn 1.6.1
- TensorFlow 2.20.0

For a local run, place the Kaggle dataset as `new_data_urls.csv` in the working
directory or edit `DATASET_CANDIDATES` in configuration cell 1. Use a
TensorFlow-compatible GPU environment for the deep models. The exact executed
environment is recorded in `results/requirements_frozen.txt`.


## Limitations

- Evaluation uses one public dataset with unknown collection dates and source
  mixture; domain-held-out evaluation is not temporal or cross-source testing.
- Registrable-domain grouping does not identify shared infrastructure or
  campaign ownership across different domains.
- Destination equivalence and browser/server behavior were not verified.
- Percent-encoding newly pushes 2,454 test inputs beyond the 200-character
  model limit, so truncation is a documented co-mechanism.
- The IDNA condition changes only four URLs and does not support a
  population-level conclusion.
- Hyperparameters were fixed rather than selected by nested group
  cross-validation.


