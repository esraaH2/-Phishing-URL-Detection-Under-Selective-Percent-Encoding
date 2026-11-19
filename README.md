# CANOE: A Canonicalization-Aware Out-of-Fold Ensemble for Robust Phishing URL Detection

A domain-held-out study of clean phishing-URL detection and robustness to
standards-aware (RFC 3986) URL representation changes, with a canonicalization
-aware stacking ensemble, **CANOE**.

This repository accompanies the paper **"CANOE: A Canonicalization-Aware
Out-of-Fold Ensemble for Robust Phishing URL Detection."** It evaluates five
single character-level deep models, four end-to-end CNN–recurrent hybrids, a
character *n*-gram baseline, and three prediction-level ensembles under a
leakage-safe protocol: canonical deduplication, a registrable-domain-held-out
split, training-only preprocessing, validation-only thresholds, and paired,
dependence-aware robustness analysis.

> **Scope.** This is an offline, defensive evaluation. The experiment does not
> visit URLs, resolve domains, retrieve page content, submit credentials, or
> interact with live services. The transformations test representation
> sensitivity under RFC 3986 equivalences; they do not prove
> destination-equivalent end-to-end attacks.

## Main findings

- Cleaning reduced 822,010 raw rows to **799,620 unique labeled URLs** (removing
  24 contradictory-label rows and 22,366 canonical duplicates) in **316,684
  registrable-domain groups**.
- The registrable-domain-held-out split contains **489,542 training**,
  **146,845 validation**, and **163,233 test** URLs, with **zero** canonical-URL
  and **zero** registrable-domain overlap between partitions.
- On clean held-out domains, the three ensembles beat every individual
  detector; **CANOE and Stacking reach the best F1 (0.9755)**, the best hybrid
  (CNN+BiLSTM) reaches 0.9715, and the best single models (CNN, GRU) reach
  0.9644.
- **End-to-end CNN–recurrent hybrids do not reliably beat a plain CNN:** the
  clean gain is small (+0.0021 to +0.0071 F1) and the advantage under
  percent-encoding changes sign across runs.
- Selective path/query percent-encoding changed **116,955 test URLs (71.65%
  coverage)**. Every deep model loses **0.0731–0.1131 full-test F1** under
  encoding and lets 18.6–46.3% of detected phishing URLs evade.
- **CANOE, scoring an RFC 3986 canonical view, reduces the percent-encoding
  loss to 0.0000** and leaves its confusion matrix unchanged. The ablation
  shows the robustness comes from **canonicalization**, not from architectural
  or ensemble combination; on a held-out dot-segment equivalence outside the
  canonicalizer's scope, CANOE still loses 0.0635 F1.
- Integrated-Gradients attributions explain the failure: after encoding, the
  inserted `%XX` escapes capture 27–31% of the attribution and the path/query
  evidence flips from "phishing" to "legitimate".

The central result is that the best model on clean URLs is not necessarily the
most robust under representation changes, and that standards-aware
canonicalization, not architecture, supplies the robustness on the covered
equivalences.

## Experimental design

### Dataset

The experiment uses the public
[Phishing and Legitimate URLs](https://www.kaggle.com/datasets/harisudhan411/phishing-and-legitimate-urls)
dataset. The notebook expects `new_data_urls.csv` with `url` and `status`
columns. In the source dataset, `0` denotes phishing and `1` denotes
legitimate; the experiment uses phishing as the positive class for all metrics.

The raw dataset is **not** redistributed here. Download or attach it directly
from Kaggle.

### Leakage controls

1. Unicode NFC normalization and removal of control/whitespace characters.
2. Canonical URL keys used only for duplicate auditing — never as model input.
3. Removal of canonical URLs with contradictory labels.
4. Deterministic same-label canonical deduplication.
5. Five-fold `StratifiedGroupKFold` by registrable domain: three folds train,
   one validation, one test (≈60/20/20).
6. Executable assertions for zero URL and zero domain overlap.
7. Character vocabulary and all learned preprocessing fitted on training only.
8. Decision thresholds selected on clean validation predictions and frozen
   before test evaluation.

### Models

| Family | Models |
|---|---|
| Single deep | CNN, BiLSTM, GRU, LSTM, Transformer |
| Hybrid | CNN+BiLSTM, CNN+LSTM, CNN+GRU, CNN+BiLSTM+Attention |
| Classical | Character 3–5-gram TF-IDF + logistic regression |
| Ensemble | SoftVote, Stacking, **CANOE** (canonical-view stacking) |

Deep models use a maximum URL length of 200, **64-dimensional** character
embeddings, dropout 0.3, Adam (`1e-3`, clipnorm 1.0), batch 512, mixed
precision, at most 10 epochs, early stopping (patience 2), and
validation-F1-maximizing thresholds. The pipeline is run three times with the
same configuration (seed 42); because GPU kernels are not bit-for-bit
deterministic, the runs differ, and conclusions are based only on results that
hold in all three runs.

### CANOE

CANOE applies an RFC 3986 canonicalizer `C(·)` (lowercase scheme/host, IDNA
A-labels, no default port, single-pass decoding of percent-encoded *unreserved*
characters only), scores the canonical view with ten detectors, and combines
their logits with an ℓ2-regularized logistic-regression meta-learner
cross-fitted over validation domains (out-of-fold `StratifiedGroupKFold`). It
differs from plain Stacking only by `C(·)`.

## Notebook

`notebooks/phishing_url_robustness.ipynb` is the end-to-end, self-documenting
pipeline. It is designed to run on **Kaggle with a GPU image** (the raw dataset
attached as input). It builds the split, trains every model, runs the
ensembles, computes paired statistics and tables, and produces the confusion
matrices and the Integrated-Gradients explainability figures. Figures are saved
under the run's `figures/` directory and are also embedded in the notebook
outputs.

To report seed-to-seed variability, set `SEEDS = [42, 52, 62]` in the first
configuration cell and re-run.

## Citation

If you use this code or the evaluation protocol, please cite the paper:

> E. Halboup and M. Halboup, "CANOE: A Canonicalization-Aware Out-of-Fold
> Ensemble for Robust Phishing URL Detection," 2026.
