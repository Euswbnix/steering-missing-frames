# Preregistration

## 1. Research question and hypotheses

**Question**: in steering-angle prediction from driving video, do LTC and CfC give a better trade-off between accuracy and compute cost than an LSTM that receives the same timing information, as observations become sparse?

The two hypotheses are kept separate and **cannot stand in for each other**:

| | Hypothesis | Notes |
|---|---|---|
| **Primary** | Under the pre-specified sparse condition, CfC (and LTC) have a lower macro MAE than the LSTM (given the frame interval Δt as an input) | The main comparison; tests "more accurate when sparse" |
| **Secondary (error-increase comparison)** | From full history to the sparse condition, the relative error increase of CfC / LTC is smaller than that of the LSTM | A robustness supplement. **A model that predicts poorly to begin with can also show a small increase**, so the secondary hypothesis holding on its own does not count as support for the main question |

Condition for the main comparison: **75% of the history observations dropped at random** (the current frame is always kept). 50% dropped and full history are reported as secondary conditions.

## 2. Models and parameter matching

All models share the same frozen DINOv2 ViT-B/14 features (3 s of history, 10 Hz base rate, 30 steps) and a projection layer and readout head of the same structure; only the temporal block differs.

| Block | Trainable parameters (measured, input 64, hidden 64) |
|---|---|
| Shared projection Linear(1536 → 64) | 98,368 |
| LSTM: LSTM(65 → 64) | 33,536 |
| CfC (hand-written in this repository) | 33,024 |
| LTC (ncps 1.0.1, fully connected wiring) | 33,216 (plus 8,192 frozen mask entries, listed separately and not counted) |
| Transformer (minimal experiment, added 2026-09-25, see section 9) | 33,472; one layer, 64-dim, 4 heads, feed-forward width 128 |
| Shared readout head Linear(64 → 1) | 65 |

- The three temporal blocks are within 1.5% of each other. This is the natural match at hidden size 64 and is not adjusted further.
- **Parameter counts are reported per block** (projection / temporal / readout), not only as a total, and matching is not claimed just because "all hidden sizes are 64".
- The CfC that ships with ncps has 49,536 parameters in the same configuration and is not used; the hand-written one in the repository is.

## 3. Training and evaluation

- **Training**: frames are dropped at random during training. Each batch draws one drop rate (0 / 50% / 75%) and uses only that rate (LTC produces NaN at Δt = 0). All models use the same data and the same augmentation protocol and are trained independently.
- **Evaluation**: the same checkpoint is evaluated under all three conditions. Only history observations are dropped, the current frame is kept, and Δt is recomputed from the frames that remain.
- **Tighter window rule**: the largest step inside a window is ≤ 0.2 s (the history experiment capped the total span instead, which let gaps into 1.54% of the training windows).
- **Target frames**: the same as in the single-frame and history experiments (speed > 3 m/s, label not interpolated across a CAN gap).
- **Supplementary condition**: samples with gaps in the original timestamps (frames dropped at recording) are evaluated separately, reported only, and not part of the main test.
- **Tuning budget**: per model 4 learning-rate runs ({1e-3, 3e-4, 1e-4, 3e-5}, 1 seed each) plus 8 confirmation seeds, at most 60 epochs, early-stopping patience 8. Learning-rate selection and early stopping both use the mean macro MAE over the three conditions.

## 4. Metrics

- **Primary metric**: macro MAE (the mean over three bins of |steering angle|: < 5°, 5–15°, ≥ 15°; `metrics.macro_mae`).
- **Secondary metrics**: CCC, per-bin MAE, and measured compute cost (same device, same precision, same batch, timed after warm-up with synchronisation; single-window latency and batch throughput reported separately, with the implementation stated).
- Primary statistic: the difference in macro MAE between two models on **the same target frames** (frame-weighted), decomposed by route group.

## 5. Statistical method and power (a pilot run must come before finalisation)

**The variance from the history experiment cannot be used for the power estimate**: the history experiment has only overall scores for the MLP and the LSTM, no paired differences, does not separate the three sources of variation (seed, route group, drop mask), and did not train with random frame dropping.

**Pilot run done **: LSTM and CfC, 3 seeds each, trained with random frame dropping. Per-frame predictions were saved for 190,000 target frames, with 5 extra random masks per drop condition.

Variance decomposition of the paired difference (CfC − LSTM, macro MAE in degrees):

| | 75% dropped (main condition) | Full history |
|---|---|---|
| Mean difference | +0.210 | +2.034 |
| SD across seeds | 0.178 | 0.175 |
| SD across masks within a seed | 0.035 | — |
| SD of the route-group contribution | 0.069 | 0.219 |
| Seed × route-group bootstrap 95% interval | [−0.181, +0.620], half-width 0.401 | [+1.078, +3.107], half-width 1.015 |

**Fixed on this basis (not to be changed):**

- **Equivalence margin δ = 0.5°** (about 3% of the 16° error under the main condition). Reason: with random frame dropping in training, dropping 75% raises the LSTM's error by only 0.38–0.65°. The margin must not be larger than the effect under study, so 1.0° is too wide and 0.5° is used.
- **Final number of seeds n = 8** (per model). The extrapolated half-width is 0.245°: if the true difference is near +0.21, the upper end of the interval is about 0.45°, still inside ±0.5. The half-width of 0.401 at n = 3 is not enough. The test split has 16 route groups (validation only 7), so the route-group part of the variation will be smaller still and this estimate is conservative.
- **Masks are not the bottleneck**: the SD across masks within a seed is only 0.035°, so averaging 5 masks per condition in the final evaluation is enough.
- **Training budget widened (symmetrically for all models)**: learning-rate grid {1e-3, 3e-4, 1e-4, 3e-5}, 1 seed each; at most 60 epochs, early-stopping patience 8. Reason: in the pilot CfC picked 1e-4, the smallest value of the grid, and one seed ran the full 30 epochs before stopping, so the original budget was holding it back. Learning-rate selection and early stopping both use the mean macro MAE over the three conditions.
- **Decision rule**: the primary statistic is the paired macro MAE difference between CfC and LSTM with 75% dropped, decomposed by route group, with a 95% interval from a two-level seed × route-group bootstrap.
  - interval entirely within ±δ → report "practically equivalent";
  - interval excludes 0 → report "different", with direction and size;
  - otherwise → report "inconclusive", not "equivalent".

**Statistical form (proposed)**:
- Paired differences are first averaged over seeds and masks, then aggregated by route group; the primary test uses two-level seed × route-group resampling.
- Sensitivity analyses: clustering by session and by date. **If the conclusion changes with the clustering, the sensitivity is reported as it is, and the more significant version is not picked.**
- The equivalence margin δ is written down before any test result is seen and may not be widened because of the pilot results. If power is insufficient, report "inconclusive", not "practically equivalent".
- Multiple comparisons: there is a single primary test (the primary hypothesis, CfC vs LSTM with 75% dropped). LTC vs LSTM, LTC vs CfC, the error-increase comparisons and the per-condition curves form the secondary family, with Holm correction.

## 6. Extra requirements for LTC

Content: Δt mismatch when the batch size equals the number of hidden units, NaN at Δt = 0, a fixed seed for the `erev` initialisation, whether Δt actually has an effect, andapply_weight_constraints()` must be called after every optimiser step, otherwise parameters turn negative and the ODE diverges. In addition:

- **Sensitivity to the number of integration steps**: with the same weights, evaluate under the largest-gap condition with 6, 12 and 24 integration steps; if the results change noticeably, retrain with finer integration. This is a numerical check on fixed weights, not a comparison of three independent models.
  - **Criterion: use the trained seed-0 weights, 75% dropped, mean over 5 masks each. If the validation macro MAE at 6 and at 24 integration steps differs by more than **0.1°** (one fifth of the 0.5° equivalence margin), integration is judged not converged and all LTC models are retrained with 24 integration steps; otherwise 6 steps stay. Only this one round of checking is done, with no choosing among several step settings.
- If no stable result is obtained within the budget, LTC is marked "no valid comparison completed"; a low score from coarse integration may not be written up as LTC lacking capability.
- Differences between LTC and CfC cannot be attributed only to "numerical integration vs closed-form approximation"; the two are not strictly equivalent implementations of the same network.

### Additional rules during the runs

- If the training loss or the validation score becomes non-finite, training stops at once and is recorded as "diverged"; the best state before divergence is kept, and the divergence is not counted as "no improvement" while training continues.
- Learning-rate grid points that diverge **do not take part** in the selection, however good their scores before divergence; if all four diverge, the model is marked "no valid comparison completed" under the rule above.
- The first round of LTC results (4 grid points, 3 seeds of final training) was trained without the parameter constraints and is entirely void. The files are kept on the server in `~/workspace/hdd/results/invalid_ltc_no_weight_constraints/` and enter no analysis. LSTM and CfC are not affected.

### Result of the integration-step check: failed, all LTC models retrained with 24 integration steps

Evaluated on the validation split with the trained seed-0 weights, under the rule written above (75% dropped, mean over 5 masks):

| Integration steps | 75% dropped | Full history |
|---|---|---|
| 6 (original setting) | 17.563 | 17.141 |
| 12 | 17.725 | 17.133 |
| 24 | 17.961 | 17.147 |

- With 75% dropped, 6 and 24 steps differ by **0.398°**, above the 0.1° threshold, so integration is judged not converged. Under full history the three are nearly identical: the problem is integration that is too coarse when the gaps between observations are large, which is exactly what this check is meant to catch.
- Per the rule, **all LTC models are retrained with 24 integration steps**, and the learning-rate grid is rerun as well (a rate chosen at 6 integration steps need not carry over). All 6-step results (4 grid points, seed 0 complete, seeds 1–7 each up to epoch 8) are archived on the server in `~/workspace/hdd/results/ltc_unfolds6_superseded/` and are **not part of the main comparison**.
- At 24 integration steps a single LTC process needs about 11.4 GB of GPU memory (backpropagation stores the intermediate results of every integration step), so at most 2 run at once. The batch size stays at 256 and is not changed for memory reasons, so the training protocol stays the same across models.
- Run time will be clearly longer. Since there is a checkpoint every epoch, runs can continue over several nights.

### LTC switched to a compiled integration cell

- At 24 integration steps the native implementation is too slow (about 18 minutes per epoch) and uses too much memory (about 11 GB per process), so the ncps LTC cell (input mapping, 24 integration steps, output mapping) is compiled with `torch.compile`. **The maths is unchanged**: it is still ncps's own code, and differences come only from the order of floating-point operations.
- Verification (on real windows, 6 different batch sizes, alternating training and evaluation): maximum relative error 1.1×10⁻⁶ for training outputs, 1.3×10⁻⁵ for the gradients of all parameters and 1.1×10⁻⁶ for evaluation outputs. Only 3 graphs were compiled and none were added after timing, so it did not silently fall back to native mode.
- Effect: a training step went from 114.9 ms to 18.9 ms (6.1× faster) and peak GPU memory from 9.37 GiB to 0.53 GiB, so the 4 grid points can run at once.
- The first epoch at learning rates 1e-3 and 3e-4 was trained with the native implementation, and the first 4 epochs at 3e-5 during an attempt at static-shape compilation; after that all runs resumed from checkpoints with the dynamic-shape compiled version. The numbers agree, so the results are unaffected; this is recorded here as it happened.
- Compilation only affects training speed. The compute-cost comparison of section 4 is measured separately, with all three models timed in the same implementation.

### Training logged to SwanLab

- From this day all training scripts (`hdd/sparse_run.py`, and in the earlier repository `single_frame_train.py`, `history_train.py`, `sparse_pilot.py` and the root `train.py`) send each epoch's numbers to SwanLab through `tracking.py`. The SwanLab project of the sparse-observation experiment is named `CSC413-D4`. Each checkpoint (model, learning rate, seed, integration steps) is one SwanLab run; the run ID is stored in the checkpoint, so a run resumed after a shutdown continues the same curve.
- What is logged: training loss (standardised mean squared error, averaged per epoch); the mean macro MAE over the three validation conditions (the selection criterion); macro MAE and CCC for each condition; the best score so far, the patience used, the time per epoch and the peak GPU memory. In the final training stage, the final evaluation over 5 sampled masks is added.
- **Logging only, never part of any decision**: learning-rate selection, early stopping and the analysis of section 5 still read only the JSON and prediction files under `results/sparse/`, as before.
- Logging does not affect training results: swanlab never touches the numpy or torch random number generators. It uses Python's built-in `random` to pick a run colour and ID, and `tracking.py` saves and restores that state around every call (`tests/test_tracking.py` checks that all three random states are bit-identical before and after a full run). The training loss is accumulated on the GPU and synchronised once per epoch. Logs are sent only after the checkpoint has been written to disk.
- Measured (2026-09-25, LSTM, learning rate 3e-4, seed 0, real data, in a separate test project): one epoch with logging on and one with logging off gave bit-identical model weights, the same validation score of 20.291029517827507 and the same three random states. Stopping with logging on and resuming from the checkpoint to epoch 2 reattached to the same SwanLab run, and the epoch-2 validation score of 22.112 matched the earlier resume check without logging.
- Only summary numbers, the run configuration, console output and run environment information (host name, package list, GPU usage and the other items swanlab collects by default) are uploaded. The console output contains only summary numbers: no session IDs, frames, features or trajectories. The project is private by default. The local copy is written to `~/workspace/hdd/swanlog/` on the server.
- For runs that had started before logging was added (LTC seeds 1 and 6), the SwanLab curves begin at the epoch where they resumed; earlier epochs are only in the log files. Runs that had finished before logging was added are not on SwanLab.
- Also, in the final training stage a seed whose result JSON already exists is skipped. The result JSON is written only after training ends, and a finished checkpoint is not trained further, so re-evaluating it would only give exactly the same numbers and open an extra SwanLab run.

## 7. Test-set discipline

- Before finalisation no test-set features are extracted and no test-set evaluation is done.
- The final evaluation is done once: each model, with its selected configuration and the final number of seeds, is run once on the test split, and the plan is not changed afterwards.
- Once the test split has been used for any choice, it is no longer confirmatory evidence.

## 8. Status of the originally open items (updated 2026-09-25)

1. The minimal Transformer experiment is specified in section 9, with its parameters reported the same way.
2. LTC was switched to 24 integration steps under section 6; the added 24/48-step diagnostic follows section 9.
3. This file's commit hash is written into the README.

All elements of the primary test (primary hypothesis: CfC vs LSTM with 75% dropped) are fixed, and the runs can start.

## 9. Additional decisions before the test (2026-09-25)

**When this was recorded**: the 8 final seeds of LSTM, CfC and LTC were finished and the team had seen the validation comparison of the three; according to the team, the test split had not been evaluated. This section fills in the Transformer configuration and the cost-measurement details that had been left open, and adds one LTC numerical diagnostic. It is not a rule made before the validation results were available. The original main comparison, the equivalence margin, the learning-rate grid and the results of completed runs stay as they are.

### 9.1 Minimal Transformer experiment

- **Role**: a supplementary course experiment, reported separately as a descriptive result and not added to the primary test or the secondary family of section 5. Whether the course accepts it as the Advanced Concept depends on the requirements the professor confirms.
- **Fixed structure**: the same LayerNorm without affine parameters, Linear(1536 → 64) and Linear(64 → 1) as the other models; in between one `TransformerEncoderLayer(d_model=64, nhead=4, dim_feedforward=128, dropout=0, activation="gelu", batch_first=True, norm_first=True)`, no final LayerNorm, no architecture search.
- **Timing information**: an encoding of real time relative to the current target frame. Let `p_i = (t_i - t_target) / 0.1 s`; dimensions `2j` / `2j+1` are `sin(p_i / 10000^(2j/64))` / `cos(p_i / 10000^(2j/64))` for `j=0..31`, added to the projected features. After frame dropping the original timestamps are still used, not compacted indices. The time encoding adds no trainable parameters.
- **Readout and masks**: frames sorted by time, with a causal attention mask and a padding mask; only the last valid token, the one for the current frame, is read; automatic nested-tensor conversion is disabled. Synthetic inputs are used to check that changing the padding values does not change the prediction and that the current frame is never dropped.
- **Budget and data**: the random frame dropping in training, target frames, window filter, optimiser and selection criterion of section 3; 4 learning rates at seed 0 plus seeds 0–7 at the selected rate, at most 12 training runs, 60 epochs, patience 8. The final evaluation uses the same 5 masks as the other models. No architecture search or time-encoding ablation is added.
- **Parameters**: temporal block 33,472, projection 98,368, readout 65, total trainable 131,905; counted locally on instances of the PyTorch modules above, to be checked again once the server implementation is done. Interface and mask semantics follow the [PyTorch TransformerEncoderLayer documentation](https://docs.pytorch.org/docs/2.14/generated/torch.nn.TransformerEncoderLayer.html).

### 9.2 Compute cost of the three recurrent models

- **The main table uses one implementation for all**: LSTMCell, the repository's CfC cell and the ncps LTCCell all run step by step in eager mode with an outer Python time loop, without `torch.compile`; LTC uses 24 integration steps. The LSTMCell copies the existing LSTM's weights and is not retrained.
- **Verify before timing**: with the existing seed-0 checkpoints and the same validation windows, check that the converted outputs match the original inference path (standardised outputs, `rtol=1e-4, atol=1e-5`). If they do not, fix the conversion first; the timing of a wrong implementation is not compared.
- **Scope of the measurement**: the main table covers only the temporal block, with 64-dim inputs and Δt already on the GPU. It excludes the visual encoder, projection, readout, disk reads, mask generation and CPU/GPU transfers. If the time from projection to readout is measured as well, its scope is listed separately.
- **Fixed conditions**: same GPU, float32, same TF32 setting, `eval()` and `inference_mode()`; all three conditions (full, keep 50%, keep 25%) are measured. Inputs and masks are fixed and shared, real length differences are kept, and the executed length after padding is stated. The model state is reinitialised for every window.
- **Reporting**: batch=1 for single-window latency, batch=256 for throughput and peak allocated GPU memory. Each shape is warmed up 50 times, then measured 200 times, in 3 rounds. The wall-clock time of one complete call is taken after synchronisation; the median, p95 and windows per second are reported. Nothing else trains on the GPU during the measurement.
- The engineering speed of cuDNN LSTM and of compiled CfC/LTC can be reported in a separate table, labelled with each optimisation path. It is not ranked together with the main table, and the eager main table is not interpreted as hardware-independent architecture speed. Warm-up, synchronisation and repeated measurement follow the [PyTorch benchmark documentation](https://docs.pytorch.org/docs/2.14/benchmark_utils.html).

### 9.3 LTC 24 vs 48 integration-step diagnostic: done once, evaluation only

- **Purpose and limits**: the single round of 6/12/24 checks set out in section 6 is complete. This is a numerical diagnostic added after seeing the validation results, to check the sensitivity of the final weights trained with 24 steps. It is not to be written up retroactively as the original preregistered check, and the setting is not chosen by whether 48 steps is more accurate.
- **Fixed inputs**: the best seed-0 checkpoint of the final 24-step training, weights unchanged; inference with 24 and with 48 steps on the same validation target frames. Both full history and 75% dropped are checked; the drop condition uses the original 5 shared masks (seeds 7, 1007, 2007, 3007, 4007). The test split is not read.
- **Criterion**: the absolute difference in mean macro MAE for each condition. Only if both are ≤ 0.1° is it recorded as "stable in this check"; this does not prove numerical convergence for all weights or all time gaps. The differences and the per-mask results are reported in full, not just pass or fail.
- **If it passes**: the registered performance and cost comparisons continue with the trained 24-step model, without switching to 48 steps.
- **If it fails**: the current 24-step validation results are kept and labelled "integration sensitivity unresolved". Under the limits of section 6, LTC's disadvantage is then not interpreted as an architecture conclusion from a completed valid comparison, and no confirmatory test evaluation of LTC is started. There is no automatic extension to 96 steps and no retraining of the 8 seeds. If retraining is decided later, a plan amendment and its budget must be recorded first; the original primary test of CfC vs LSTM is not changed by it.

### 9.4 Order of execution and analysis checks

First the fixed-weight 24/48 check and the Transformer implementation check, then the Transformer runs within the budget above; cost timing is scheduled when the GPU is idle. The final test is run only after the corresponding configurations and analysis scripts are locked, and is not started at the time of this supplementary record.

Before the test, the Holm correction of the secondary family already promised in section 5 must be completed: the current `hdd/sparse_analyze.py` gives a nominal 95% interval for each pair separately and does not yet implement the joint correction of the whole secondary family. The existing validation table cannot serve as a Holm-corrected confirmatory conclusion. The primary test remains CfC vs LSTM (75% dropped) only; the comparison family does not change because of LTC's validation results.

### 9.5 Execution record (2026-09-25)

**9.1 Transformer implementation check** (`TransformerBlock` in `hdd/sparse_run.py`, `tests/test_sparse_transformer.py`)

- The structure is implemented exactly as in 9.1. `TransformerEncoderLayer` is used directly, not wrapped in `TransformerEncoder`, so there is no nested-tensor conversion; in addition `torch.backends.mha.set_fastpath_enabled(False)` turns off the inference fast path, so training and evaluation use the same computation (a test checks that the two modes give the same output).
- Relative time is computed by `forward_masked` from the original timestamps of the frames kept after dropping (`kept_t - t_target`) and passed to the model. LSTM, CfC and LTC do not read this input (a test checks that their outputs do not depend on whether or what is passed), and it does not change how many random numbers are consumed, so completed checkpoints and predictions are unaffected.
- Measured on the server (torch 2.14): temporal block 33,472, projection 98,368, readout 65, total 131,905, matching 9.1.
- All tests on synthetic inputs pass: features and times at padding positions can take any value without changing the prediction; under the causal mask, changing later frames does not affect the outputs at earlier positions; after dropping, the times are the original timestamps rather than compacted indices; the target frame is always kept, is the last valid position and has relative time 0.

**9.3 LTC 24 vs 48 integration-step diagnostic** (`hdd/ltc_unfolds48_check.py`; result file on the server at `~/workspace/hdd/results/analysis/ltc_unfolds_24v48_seed0.json`)

| Condition | 24 steps | 48 steps | 48 − 24 |
|---|---|---|---|
| Full history | 17.7225 | 17.7005 | −0.0220 |
| 75% dropped (mean of 5 masks) | 18.5476 | 18.6066 | +0.0590 |

- Per-mask differences (seeds 7, 1007, 2007, 3007, 4007): +0.0577, +0.0622, +0.0555, +0.0637, +0.0560.
- Both absolute differences are ≤ 0.1°, so under 9.3 the result is recorded as **"stable in this check"** and the trained 24-step model continues to be used. This does not prove numerical convergence for all weights and all time gaps; with 75% dropped, 48 steps is systematically higher by about 0.06°, always in the same direction, and this is recorded as it is.
- Cross-check: the eager 24-step results match the archived final evaluation of the compiled version mask by mask (full history 17.7225, every mask identical to 4 decimal places).

**9.4 Implementation of the Holm correction for the secondary family** (`hdd/sparse_analyze.py --family`, `tests/test_sparse_analyze_family.py`)

- Primary test: CfC − LSTM, 75% dropped, still decided by the 95%-interval rule of section 5, uncorrected.
- Secondary family (m = 11 when all three registered models have predictions): CfC − LSTM under full history and keep 50%; LTC − LSTM and LTC − CfC under the three conditions; the error-increase comparisons of the three pairs. The Transformer is in no family. A pair that has not been evaluated is listed as "not evaluated" and the family shrinks accordingly; nothing is dropped silently.
- Each secondary comparison gets two p-values from the same set of bootstrap samples: a two-sided p for a difference of 0 (`p_diff`), and a TOST p for the difference lying outside ±δ (`p_equiv`). At α = 0.05 they reproduce "the 95% interval excludes 0" and "the 95% interval lies within ±δ" respectively. p-values are computed as (count + 1)/(B + 1), so they are never 0.
- Holm is applied over the whole secondary family, once for "different" and once for "practically equivalent" conclusions; the two null hypotheses of the same comparison (the difference is 0, the difference is outside ±δ) cannot both be true. The error-increase comparisons have no registered equivalence margin and get only a difference test. After correction: `p_equiv` ≤ 0.05 → practically equivalent; otherwise `p_diff` ≤ 0.05 → different; otherwise → inconclusive. **The team confirmed this reading on 2026-09-25, before the test evaluation started.**
- If there are too few bootstrap samples to support the correction (the smallest possible p-value times m exceeds α/5), the script refuses to run; the registered 10,000 is enough for m = 11.

**9.2 Compute cost** (`hdd/sparse_cost.py`; result file on the server at `~/workspace/hdd/results/analysis/cost_seed0.json`)

- Conditions: RTX 5090, torch 2.14.0+cu130, cuDNN 9.24, float32, TF32 off for both matmul and cuDNN, `eval()` + `inference_mode()`; no other process was on the GPU at the start or the end of timing. 256 validation windows were drawn from all sessions with a fixed seed, with mask seed 7, shared by the three models; batch=1 uses the first of these windows. Each shape was warmed up 50 times and measured 200 times × 3 rounds; the medians of the three rounds are within 1% of each other.
- Verification: for the seed-0 checkpoints of all three models, the step-by-step implementation and the original inference path (cuDNN `nn.LSTM` for the LSTM) pass `rtol=1e-4, atol=1e-5` on all 6 shapes; the maximum absolute error is 1.2×10⁻⁵ for the LSTMCell and 0 for CfC and LTC.
- Main table (eager, step by step, temporal block only; executed length = number of steps after padding, with the mean number of kept steps at batch 256 in parentheses):

| Condition | Executed length batch=1 / batch=256 | Latency batch=1: LSTMCell / CfC / LTC (median, ms) | Throughput batch=256: LSTMCell / CfC / LTC (windows/s) |
|---|---|---|---|
| Full history | 30 / 30 | 0.608 / 1.682 / 39.37 | 417,340 / 134,403 / 6,353 |
| Keep 50% | 14 / 23 (15.5) | 0.292 / 0.796 / 18.26 | 540,286 / 172,507 / 8,307 |
| Keep 25% | 6 / 15 (8.3) | 0.133 / 0.353 / 7.90 | 810,083 / 265,065 / 12,634 |

- p95 is within 3% of the median. Peak allocated GPU memory at batch 256 is 104–157 MiB, of which at most 31 MiB (LTC) goes beyond the inputs and weights.
- The time at batch=1 and at batch=256 is almost the same, so this table mainly reflects the number of kernels launched per step (LTC runs 24 integration steps per step), not arithmetic; as agreed in 9.2, it is not interpreted as hardware-independent architecture speed.
- Separate table (the path each model actually used in training, not ranked with the main table): cuDNN `nn.LSTM` latency 0.055 / 0.047 / 0.043 ms (full / 50% / 25%), throughput 3.31 M / 3.45 M / 3.98 M windows/s; compiled LTC cell 3.08 / 1.46 / 0.65 ms, 53.5 k / 69.6 k / 106 k windows/s; compiled CfC cell 1.78 / 0.85 / 0.37 ms, slightly slower than eager (the kernels saved by compiling each step do not make up for the per-call checking overhead). All three paths first passed the same output verification.

**Trial run on the validation split** (2026-09-25, `sparse_analyze.py --family --split val`, B = 10,000; the validation split was also used to select the learning rates, so this is a preview only, not a confirmatory conclusion): primary test CfC − LSTM (75% dropped) +0.360 [−0.070, +0.822], inconclusive. Secondary family m = 11, all "different" after Holm correction: CfC − LSTM full history +1.98 (Holm p 0.002), keep 50% +0.87 (0.004), difference in error increase −1.62 (0.002); LTC − LSTM under the three conditions +3.11 / +3.26 / +3.36 (all 0.002), difference in error increase +0.25 (0.011); LTC − CfC under the three conditions +1.13 / +2.39 / +3.00 (0.003 / 0.002 / 0.002), difference in error increase +1.87 (0.002). 0.0002 is the smallest p-value 10,000 samples can give; times 11 it is 0.0022.

**Preparation for the test evaluation (2026-09-25, not yet run)**

- `hdd/run_sparse_test.sh test` is the complete procedure of the single test evaluation: extract features for the 21 test sessions with the same settings as for training and validation (DINOv2 ViT-B/14, stride 1, 392×224); append only the test sessions to the cache with `make_cache.py --add`, without changing any file of the existing sessions; score the 32 completed checkpoints of the four models at the frozen learning rates; then run the registered analysis (`--family`) and the descriptive Transformer comparison. It makes no choices.
- The first run writes `results/TEST_EVALUATION_STARTED` with the start time and the SHA-256 of every script and learning-rate selection file; any later run is refused, and only `--resume` can continue that one run if it is interrupted.
- When `sparse_run.py` is pointed at the test split, it exits at once if any checkpoint is unfinished, so it can never train or early-stop on the test split as if it were validation.
- A full rehearsal on the validation split (`run_sparse_test.sh rehearse`, on copies of the checkpoints, not logged to SwanLab) reproduced the existing results bit for bit: the per-frame predictions and per-mask scores of all 32 models differ by 0, all estimates, intervals, Holm p-values and decisions of the registered analysis are the same, and so is the descriptive Transformer comparison.

**Test evaluation run** (2026-09-25 23:02–23:58, locked version: commit [`dc4a36c`](https://github.com/Euswbnix/CSC413_Project/commit/dc4a36c) of the earlier repository): the primary test is "practically equivalent"; results in [`results_2026-09-25_sparse_test.md`](results_2026-09-25_sparse_test.md). The plan is not changed after this.

**Implementation issue found after the test (2026-09-26)**: Δt was passed to every model in seconds, which does not match the CfC implementation's convention of "units of the training-set median step" and may put CfC and LTC at a disadvantage. It is written into the test results as a limitation. The comparison with corrected units is a post hoc exploratory experiment with its own plan ([`explore_2026-09-26_dt_units.md`](explore_2026-09-26_dt_units.md)); it does not modify this plan and does not use the test split again. Exploratory result (2026-09-27): with corrected units neither CfC nor LTC improved, so by that plan's pre-written rule the units are not a material confounder.
