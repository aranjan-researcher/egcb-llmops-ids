# Entropy-Guided Circuit Breakers for LLM-Augmented Intrusion Detection

Replication package for the paper *"Entropy-Guided Circuit Breakers for LLM-Augmented Intrusion Detection: An Empirical Evaluation of Uncertainty-Routed Cascades in LLMOps"*.

The study asks two questions about putting a Large Language Model (LLM) inside an intrusion detection pipeline:

1. **Is it worth calling an LLM at all?** A Random Forest classifies each network flow and sends only low-confidence flows to the LLM.
2. **Can token entropy tell us when the LLM's reasoning is going wrong?** An entropy-guided circuit breaker (EGCB) watches the model's token entropy during generation and falls back to the Random Forest when entropy is high.

The answers, in short: routing works and larger LLMs help on the flows the Random Forest finds hard, while token entropy is too weak a signal to control generation. Agreement across sampled answers (self-consistency) fails in the same way.

Everything needed to reproduce these results is in this repository.

---

## What is here

```
notebooks/EGCB_experiments.ipynb   One notebook that produces every table and figure
results/tables/                    All result tables (CSV)
results/figures/                   All figures (PNG and PDF)
results/data/eval_set.csv          The exact flows evaluated, with their prompts
results/llm_runs/                  Raw per-flow generation logs (JSONL)
results/sc_runs/                   Raw self-consistency samples (JSONL)
results/config.json                Every parameter of the run
results/environment.json           Hardware and library versions used
results/tau_star.json              The calibrated entropy thresholds
results/experimental_setup.txt     The setup paragraph of the paper, generated from the run
requirements.txt                   Python dependencies
```

The raw per-flow logs in `results/llm_runs/` and `results/sc_runs/` hold the token entropies, per-token timestamps and generated text for every generation. See "Raw logs" below.

## Main results

| Finding | Evidence |
|---|---|
| The Random Forest routes only 0.28–9.48% of flows to the LLM | `T2_reflex_kernel.csv` |
| LLM accuracy on routed flows rises with size: 52.3% → 71.3% (Random Forest 56.5%) | `T3_main_results_naive_vs_egcb.csv` |
| Token entropy predicts wrong answers with AUROC 0.505–0.564 | `T6_entropy_predicts_failures_auroc.csv` |
| The calibrated breaker trips on 100% of flows, saving 88–95% of tokens but costing accuracy | `T3`, `T4_tau_sweep_calibration.csv` |
| No entropy threshold beats a random breaker by more than 2.7 points | `T9_oracle_tau_vs_random.csv` |
| Self-consistency (5 samples) does not beat greedy decoding | `T14`, `T15`, `T16` |
| A Neural Network reflex is faster but routes less usefully | `T12_reflex_rf_vs_mlp.csv` |
| The live breaker matches the offline replay on all 4,800 generations | `T8_live_vs_replay_agreement.csv` |

## Datasets

The three benchmarks are public but are **not redistributed here**. Download them from their original sources and keep their licences:

- **CIC-IDS2017** — https://www.unb.ca/cic/datasets/ids-2017.html (the eight `*_ISCX.csv` files)
- **CICIoT2023** — https://www.unb.ca/cic/datasets/iotdataset-2023.html (the official `train.csv` and `test.csv`)
- **NSL-KDD** — https://www.unb.ca/cic/datasets/nsl.html (`KDDTrain+.txt` and `KDDTest+.txt`)

## How to reproduce

**In Google Colab (what we used)**

1. Open `notebooks/EGCB_experiments.ipynb` in Colab and choose a GPU runtime. We used one NVIDIA A100 80 GB; a 24 GB GPU is enough if the 14B model is loaded in 4-bit.
2. Put the three datasets (zipped or extracted) in Google Drive, and set the paths in cell 0.
3. Run the cells in order. Results are written to Drive after every flow, so an interrupted session can simply be resumed by running again.

**Locally**

```bash
pip install -r requirements.txt
jupyter lab notebooks/EGCB_experiments.ipynb
```
Set `MOUNT_DRIVE = False` in cell 0 and point `ZIP_PATHS` at your local copies of the data.

**Runtime.** Preprocessing and the Random Forest take a few minutes on CPU. The main LLM runs take about 6–9 hours on an A100 for four models; the self-consistency baseline adds roughly 3.5 hours. Set `QUICK_TEST = True` first for an end-to-end check that finishes in minutes.

**What each cell does**

| Cell | Purpose |
|---|---|
| 0–2 | Configuration, installation, output folders |
| 3–4 | Load the datasets, clean them and build the splits (Table 1) |
| 5 | Train the Random Forest reflex, measure it, apply the routing threshold (Table 2) |
| 6 | Build the evaluation flows and their prompts, including the retrieved reference flows |
| 7–9 | Load each LLM and run it without a breaker, logging entropy and timing for every token |
| 10 | Calibrate the entropy threshold on training flows only |
| 11 | Run the live circuit breaker on the test flows and check it against the offline replay |
| 12–15 | Build the result tables and figures, and print the setup paragraph of the paper |
| 16 | Oracle threshold analysis and the answer-level verifier (CPU only) |
| 17 | Neural Network reflex, compared with the Random Forest (CPU only) |
| 18 | Self-consistency baseline (GPU) |
| 19 | Analysis of the self-consistency samples (CPU only) |

Cells 16, 17 and 19 read the saved logs, so they can be re-run without a GPU.

**Reproducing the threshold studies without a GPU.** Because decoding is greedy, a run with the breaker is exactly a prefix of the run without it. Cell 16 replays the logged generations to compute what any threshold would have done, so the calibration and oracle analyses can be repeated on a laptop once the logs are downloaded.

## Raw logs

`results/llm_runs/` holds one JSONL file per model and run mode, and `results/sc_runs/` one per model for the self-consistency baseline. Every record carries the per-token entropy trace, the cumulative per-token timestamps, the prompt and generated token counts, the stop reason and the generated text.

| File | Records | Contents |
|---|---|---|
| `llm_runs/{model}__naive.jsonl` | 1,650 each | 1,200 test flows (600 routed, 600 stress) plus the 450 calibration flows, generated without a breaker |
| `llm_runs/{model}__nsacb.jsonl` | 1,200 each | The same test flows under the live circuit breaker |
| `sc_runs/{model}__selfconsistency.jsonl` | 1,050 each | Five sampled answers per flow, for the 600 routed test flows and the 450 calibration flows (7B and 14B only) |

All 11,400 records carry the same `run_version` fingerprint, `c67f7c79bc`, which the notebook computes from the prompt template and the run settings; records from any other configuration are ignored on load.

The breaker logs keep the file name `__nsacb.jsonl` from before the system was renamed to EGCB. Nothing reads them by name — cells 16, 17 and 19 read the `__naive` and `__selfconsistency` files — so they are released under the names the run produced.

## Citation

If you use this code, please cite the paper (see `CITATION.cff`).

## Licence

Code: MIT (see `LICENSE`). The datasets keep the licences of their original providers.

## Note on the published record

This repository accompanies a revised version of the manuscript. During the revision we found errors in our earlier evaluation scripts, rebuilt the pipeline from scratch and regenerated every result. The notebook here is that rebuilt pipeline, and it is the only source of the numbers in the paper.
