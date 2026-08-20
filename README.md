# Ratio-Consistency Loss and a Feedback Gate for AML Blast Cell Segmentation: An Ablation Study of Nucleus-to-Cytoplasm Ratio Reliability

Code and experiment notebook for the paper *Ratio-Consistency Loss and a Feedback Gate for AML Blast Cell Segmentation: An Ablation Study of Nucleus-to-Cytoplasm Ratio Reliability*. This repository contains the full research workflow used to produce every number, table, and figure in the manuscript, data loading, model definition, training, the Phase I ablation study, the Phase II target-domain transfer study, and all statistical tests.

## Environment

- Python 3.11
- PyTorch (CUDA-enabled)
- segmentation-models-pytorch
- Albumentations
- OpenCV (opencv-python-headless)
- scikit-image
- SciPy
- pandas
- statsmodels
- scikit-learn
- matplotlib

Training was performed on a single Google Colab-assigned GPU, with Google Drive used for persistent dataset and checkpoint storage between sessions. Install dependencies with:

```bash
pip install -r requirements.txt
```

## Repository structure

```
AML_V2.ipynb           Main research notebook, run top to bottom, outputs saved
AML_ratio_project/      Checkpoints, per-run logs, and result CSVs produced during training
aml_segmentation/       Pseudo-label masks and supporting AML target-domain data assets
requirements.txt        Python dependencies
README.md               This file
```

## How to run

The notebook is designed to run in Google Colab with a mounted Google Drive.

1. Open `AML_V2.ipynb` in Colab.
2. Run the **Setup** section, this mounts Google Drive and installs dependencies.
3. Run **Phase I** top to bottom. This downloads WBCAtt+, defines the model, loss, and training loop, and runs the full 18-run ablation grid (2 backbones, ratio-consistency loss on/off, RFG module on/off, 3 seeds each). This is the most compute-intensive step.
4. Run **Section 6**, this builds the ablation results table, the paired significance tests, and the RFG correlation diagnostics.
5. Run **Phase II**, this generates AML pseudo-labels, fine-tunes the best Phase I checkpoint on the AML target domain, and compares it against training from scratch across five independent data splits, with paired significance testing.
6. Run **Section 10**, a controlled follow-up that repeats fine-tuning at a learning rate matched to the scratch regime, to check whether the Phase II result depends on that setting.

Every cell was executed in this order to produce the results reported in the paper. The notebook is pushed with all outputs saved, so the full workflow can be inspected without re-running anything.

## Where each paper result comes from

| Paper item | Notebook section | What it reports |
|---|---|---|
| Section III (Methodology), model and loss definitions | Section 2 (Model), Section 3 (Loss) | `SegModel`, `RatioFeedbackGate`, `DiceLoss`, `FocalLoss`, `RatioConsistencyLoss`, `TotalLoss` |
| Table III, Phase I ablation results | Section 6, ablation results table cell | Mean and standard deviation of Dice and N:C ratio error across 3 seeds, 6 configurations |
| Table IV, Phase I paired significance tests | Section 6b, paired significance test cell | Paired t-test of each ablated configuration against baseline |
| Table V, RFG correlation diagnostic | RFG mechanism check cells | Correlation between the RFG probe's internal ratio estimate and ground truth, all 9 RFG-enabled runs |
| Fig. 3, Phase I bar chart | Ablation bar chart cell | `fig_ablation_bars.png` |
| Section III-D, AML pseudo-label generation | Phase 02, pseudo-label generation and QC cells | `segment_cell_color`, cell-fraction and nucleus-fragmentation QC, final 352-image subset |
| Table VI and VII, Phase II target-domain results | Section 09 (multi-seed loop and significance test), Section 10 (matched learning rate follow-up) | Scratch vs. fine-tuned comparison across 5 splits, before and after controlling the learning-rate confound |
| Fig. 5, Phase II qualitative comparison | Phase II visualization cell | `fig_phase2_comparison.png` |

## Notes on reproducibility

- Checkpoint selection during training uses the lowest validation N:C ratio error as the primary criterion, with validation Dice used only to break an exact tie, consistently across every training function in this repository.
- Phase I is repeated across 3 random seeds per configuration; Phase II is repeated across 5 independent random splits of the target-domain subset, both to support the paired statistical tests reported in the paper rather than relying on single-run point estimates.
- The AML-Cytomorphology LMU reference masks used in Phase II are automatically generated pseudo-labels, not expert-verified ground truth, as described in the Methodology and Limitations sections of the paper.
