# Methodology and Pipeline Changelog

This document records the debugging and methodology-rewrite history behind the results reported in the paper. It is provided as evidence that the reported findings reflect a genuine, iteratively corrected experimental process rather than a single unverified run.

## 1. Initial pipeline audit

The original notebook (`AML_FullPlan.ipynb`) was audited end to end against the results it produced. Two structural issues were identified before any results could be trusted.

**Issue 1, checkpoint selection criterion.** `run_experiment`, `fine_tune_on_aml`, and `train_aml_from_scratch` all selected the "best" epoch checkpoint using validation Dice as the primary criterion, with N:C ratio error only used to break an exact Dice tie. Since Dice differences between epochs were often in the fourth decimal place while ratio error varied substantially epoch to epoch, this meant the reported "best ratio error" for a run was effectively whichever epoch happened to have peak Dice, not the model's actual best ratio reliability. This was verified directly against a saved per-epoch history file, where the epoch with the single highest Dice (0.9899) had a ratio error of 0.0719, while an earlier epoch with only marginally lower Dice (0.9898) had a ratio error of 0.0576.

**Issue 2, unsupervised RFG probe.** The Ratio-Feedback Gate module's probe head, which is supposed to estimate the network's own N:C ratio, received no direct training signal tying its estimate to the true ratio. The ratio-consistency loss was applied only to the final segmentation output, not to the RFG probe's own preliminary prediction. A diagnostic check confirmed this: the correlation between the probe's ratio estimate and ground truth was near zero or negative across the RFG-enabled configurations (-0.13, 0.20, -0.45 on the original, buggy runs).

## 2. Fixes applied

- Changed the checkpoint selection rule in all three training functions to prioritize the lowest validation N:C ratio error, with Dice used only as a tiebreaker.
- Added an auxiliary supervision term (`lambda3`) to `TotalLoss`, applied whenever the RFG module is active, that directly supervises the RFG probe's own ratio estimate against the ground truth ratio.
- Made the `lambda3` weighting explicit at every `TotalLoss` construction site instead of relying on an implicit default.
- Fixed a separate bug in the AML pseudo-label QC pipeline where images flagged as having an implausible cell fraction were still saved to disk and left in the training subset instead of being excluded, matching the handling already used for the nucleus-fragmentation QC check.
- Applied the same corrected, ratio-error-first selection rule to the cell that picks the best Phase I configuration to transfer into Phase II, which had the same Dice-first bias, and made the downstream Phase II cells reference that selection dynamically instead of a hardcoded configuration.
- Removed three cells that were fully superseded by later cells in the notebook, and de-duplicated a `per_class_dice` function that had been defined twice.

## 3. Re-validation after fixes

- Re-ran the full 18-run Phase I ablation grid (2 backbones, ratio-consistency loss on/off, RFG module on/off, 3 seeds each) under the corrected selection rule.
- Expanded the RFG correlation diagnostic from 3 representative runs to all 9 RFG-enabled runs (3 configuration types, 3 seeds each), confirming a consistent correlation of approximately 0.98 to 0.99 between the RFG probe's estimate and ground truth on the full validation set, verifying that the supervision fix worked as intended.
- Added a paired t-test (matched by seed) for every ablated configuration against baseline, rather than comparing raw means, and found no statistically significant improvement in ratio error from either the ratio-consistency loss or the RFG module under this corrected pipeline.

## 4. Phase II robustness pass

- Identified that the original Phase II comparison (training from scratch vs. fine-tuning the transferred Phase I checkpoint) relied on a single train/validation/test split of the 352-image AML subset, an unreliable basis for a statistical claim given the subset size.
- Added a 5-independent-split evaluation loop with a paired t-test matched by split, replacing the single-split comparison.
- This produced a statistically significant result (scratch outperforming fine-tuning on both Dice, p = 0.0050, and ratio error, p = 0.0015), consistent across every one of the 5 splits.
- Identified a learning-rate mismatch between the two regimes being compared (scratch at 1e-4, fine-tuning at the function's unstated default of 2.5e-5) as a possible confound, and ran a controlled follow-up with both regimes at a matched 1e-4. The Dice result held under the matched rate (p = 0.0047); the ratio-error gap narrowed and lost significance (p = 0.065), indicating part, but not all, of the original ratio-error difference was attributable to the learning-rate mismatch rather than the domain gap alone.

## 5. Supporting evidence available in this repository

- `AML_ratio_project/` contains the per-run result CSVs (`all_results.csv`, `phase2_multiseed_results.csv`, `phase2_matched_lr_results.csv`) and per-epoch training history files referenced above.
- Model checkpoints (`.pt` files) for every reported configuration are retained under `AML_ratio_project/checkpoints_v2/`.
- The notebook `AML_V2.ipynb` is pushed with all outputs rendered, so every number in this changelog and in the paper can be traced back to the specific cell that produced it.
