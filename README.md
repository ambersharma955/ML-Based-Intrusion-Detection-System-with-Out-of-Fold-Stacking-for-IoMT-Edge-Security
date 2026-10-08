## Introduction

Intrusion detection on network traffic is a classification problem: given a flow's features, decide whether it is benign or one of several attack types. The Tree-based IDS paper tackles this on the CICIDS2017 dataset with four tree-structured classifiers — Decision Tree (DT), Random Forest (RF), Extra Trees (ET), and XGBoost — combined through a **stacking ensemble**, where an XGBoost meta-learner is trained on the four base learners' predictions to produce a final, hopefully stronger, classification.

Stacking is only as sound as the data its meta-learner is trained on. If the meta-learner ever sees a base learner's prediction on a row that base learner was also trained on, it is not learning to combine models — it is learning to read off whichever base learner memorized that row best. We re-implemented the authors' pipeline faithfully and audited it end-to-end before extending it, which is how we found the issues below.

## Research Gap

Reproducing the original notebook exposed two concrete problems, one data-integrity bug and one methodological gap in the core contribution:

1. **The notebook silently corrupted its own input data.** A `to_csv(...)` call re-wrote the authors' shipped `data/CICIDS2017_sample.csv` every time the notebook ran. A second cell further downstream sampled the data using raw (non-merged) CICIDS labels that don't match this file's merged labels, silently dropping the DoS, BruteForce, and WebAttack classes (2,626 rows, 4 of 7 classes) from that run without raising an error. Any numbers produced by a second run of the notebook, or by that mis-labeled sampling cell, are not valid.

2. **The stacking ensemble is trained in-sample, so it cannot be shown to add anything over the best single base learner.** The meta-learner is fit on the base learners' predictions of the rows those same base learners were just trained on. Tree models are close to perfect on their own training data (DT reaches 99.985% training accuracy), so the meta-learner mostly learns to copy the best-memorizing base learner rather than to arbitrate between them. This is visible directly in the notebook's own output: the stacking accuracy is identical to the decision tree's to 16 decimal places. Under cross-validation, the original stacking result is statistically indistinguishable from plain DT (corrected t-test p = 0.28). The paper also evaluates on a single 80/20 split, with scaling fit before the split, which cannot separate a genuine ensembling gain from an artifact of that one split. In short: stacking was never shown to actually out-perform its best base learner under a fair test — that is the gap this work addresses.

## Our Improvement

We appended a self-contained extension to the end of `Tree-based_IDS_GlobeCom19.ipynb` (it reads the notebook's earlier variables but does not modify them, so every cell above it still produces its original output) that replaces in-sample stacking with **out-of-fold (OOF) probability stacking**:

- **OOF meta-features.** Each base learner's meta-feature for a training row comes from a copy of that model trained on an inner 5-fold split that never saw that row (SMOTE is re-applied inside each inner training fold only, so no synthetic row leaks into a validation fold either). This is the standard formulation of stacking (Wolpert, 1992), which the original notebook did not implement.
- **Probability meta-features.** The meta-learner receives the four base learners' full class-probability vectors (4 models x 7 classes) instead of 4 hard labels, so it can learn which base learner to trust for which class rather than just taking a vote.
- **Leakage-free protocol.** Min-max scaling and SMOTE are fit inside each training fold only, never on data the test fold can see.
- **Fair evaluation.** All methods are compared on the same repeated stratified 5-fold cross-validation (5 folds x 3 repeats = 15 test folds, fixed seeds), plus the original notebook's 80/20 split for continuity, with a corrected resampled t-test (Nadeau & Bengio, 2003) and a Wilcoxon signed-rank test for significance.
- **Nothing else changed.** The data, the SMOTE target, the four base learners, and their hyperparameters are exactly as in the original notebook — only the meta-learner's training data differs.
- **Verified non-destructive.** The SHA-256 hash of every file in `./data` is checked before and after the extension runs; the original `to_csv` write is disabled so this can never regress again.

## Results

Evaluated on the authors' 56,661-flow CICIDS2017 sample, with identical data and folds, original stacking vs. our OOF-probability stacking:

| Metric | Original stacking | Our improvement | Change |
|---|---|---|---|
| Accuracy | 99.535 ± 0.063% | **99.691 ± 0.068%** | better on 15/15 folds |
| False alarm rate | 0.623 ± 0.129% | **0.342 ± 0.127%** | **~45% lower**, better on 15/15 folds |
| Attack detection rate | 99.707% | **99.755%** | better on 12/15 folds |
| Macro-F1 | 0.9636 | **0.9727** | better on 11/15 folds |

Pooled over one full CV repeat (56,661 flows): false alarms fell from **152 to 82** and missed attacks from **102 to 76**. An ablation that applies only the out-of-fold change (hard labels, not probabilities) captures most of the gain by itself, which confirms the in-sample meta-learner was the main cause of the original limitation; probability meta-features add the remaining gain in attack detection rate and per-class F1.

## Limitations

- **Cost.** The fix needs ~5x more training time (an inner 5-fold CV per outer fold); inference time is essentially unchanged (+6%).
- **Infiltration is still unsolved.** Its recall stays at ~77% in both the original and improved stacking — only 36 Infiltration flows exist in the entire 56,661-flow sample, too few for any version of this method to learn from reliably.
- **Scale.** All results use the authors' 2% CICIDS2017 sample, not the full 2.83M-flow dataset used for the paper's headline numbers, so our numbers and the paper's are not directly comparable to each other.
- **Duplicate flows.** The sample contains an estimated ~22% duplicate flows, which can appear in both a train and test fold. This affects the original and improved stacking equally, but inflates both methods' absolute scores.

## Where to Look

Everything — the extension's code, the full results tables, the statistical tests, and a detailed research log with these same points and suggested future work — is appended as the final cells of `Tree-based_IDS_GlobeCom19.ipynb`, after a markdown cell titled "PE1 Extension: leakage-free out-of-fold (OOF) stacking".
