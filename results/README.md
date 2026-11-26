# Result files

This folder holds the aggregate result files behind the README, [docs/results.md](../docs/results.md) and the
figures made by [notebooks/figures.ipynb](../notebooks/figures.ipynb). They are JSON files written by the
evaluation scripts, kept at the path the code writes them to (`results/main_v04/<name>`; for the second
backbone, `results/main_v04_bb2/<name>`). They contain only aggregate numbers: no audio, no transcripts and
no per-item outputs of the test or replication split.

Splits: `dev` is the development set (all tuning), `test` the held-out test set (used once, after every
setting was fixed), `replication` a second held-out split used once for the masked-sound checks.

## Files

| File | Split | What it holds | Written by | Used for |
|---|---|---|---|---|
| `decision_test_final.json` | test | Main comparison. `systems`: five-part mean (`primary`) and the five part accuracies (`parts`) of every system. `comparisons`: ours minus each system with 95% intervals. `per_rival`: the comparisons fixed in advance against the five rivals, with the decision rule's flags. `decision_rule` and `readings`: the margins and how the rule was read. `answer_loss_subtracted`: the development-set answer loss of the Gemini systems. `plan_as_frozen`: the same table with the original read-out as "ours". `by_noise`: ours minus each rival per background-noise recording. `manifest`: thresholds and the relative paths of the per-item score files (not published). | `src/p1v4_test_report.py` | Figure (a), README tables |
| `grounding_v06_test.json` | test | Matched grounding test and natural-data analyses. `m1[system][part]`: grounded group accuracy (`decision.right`) with its interval (`grounded_ci`), plus results per injected label (`by_condition`). `criteria`: the criteria fixed in advance (G1a and G1a-S give ours' grounded accuracy with its interval). `reported`: further analyses (graded grounding, subsets, seeds). `natural`: natural-data candidates, reliance tables and `errors_by_front_end_part` (error rates on "respond, hearing" split by transcript, speaker tag and detector state). | `src/p1v4_grounding_v06.py` | Figures (b) and (f) |
| `grounding_v06_dev.json` | dev | The same report on the development set. `natural.candidates` holds the natural five-part mean and the grounded accuracy of every read-out version. | `src/p1v4_grounding_v06.py` | Figure (e) |
| `noise_test.json` | test | The two sound-only parts re-rendered with 5 and 10 dB more background noise. `parts[level][part][system]`: accuracy `clean` and `degraded`. `detector`: recall and false-alarm rate per event class and noise level. `gap_change`: ours minus each system in the change. `G5`: whether ours loses less than the four label-reading systems. | `src/p1v4_noise_final.py` | Figure (c) |
| `offsets_2d_test.json` | test | "Stop, movement" with the decision point 0.5 to 2 s after the door onset: group accuracy, hits and false alarms per read-out version (`by_offset`) and the criterion on shifted offsets. | `src/p1v4_readout_2d.py` | docs |
| `reliability_confirm_replication.json` | replication | Masked-sound reliability checks: background noise raised by 10 to 40 dB. `cells[part][condition][system]`: accuracy, share of confident answers and confident-error rate per noise level. `claims`: every claim written down before the run, with its bar, interval and verdict. `summary`: verdicts of the 19 key claims and counts per claim family. | `src/p1v4_reliability_confirm.py` | docs/results.md, section 6 |
| `mirror_vs_ears_replication.json` | replication | One cell per part, background-noise recording and noise level: ours and the original read-out with no labels (`D_off_none`) and with true labels (`T_off_truth`); `ear_damage` (ours' accuracy lost to the noise with no labels), `mirror` (ours minus the original read-out, both with true labels) and `label_help`. | `src/p1v4_mirror_vs_ears.py` | Figure (d) |
| `mirror_vs_ears_newnoise.json` | dev + replication | Least-squares summaries of `mirror` against `ear_damage`: `summary_demand` pools the development and replication noise types, `summary_new_types` covers four appliance noise types from a second noise corpus. | `src/p1v4_mirror_vs_ears.py` | Figure (d) notes, docs/iterations.md |
| `newnoise_replication.json` | replication | The reliability checks repeated with the four appliance noise types (not confirmatory): `cells`, `claims`, `by_type`, `mirror_cells`, `summary`. | `src/p1v4_newnoise.py` | docs/iterations.md |
| `latency_dev.json` | dev | Median and 90th-percentile time per component and the median time per decision of each system (`totals_median_s`, seconds) on an RTX 5060 Laptop GPU. | `src/p1v4_latency.py` | README speed table |

## Files of the second backbone (`main_v04_bb2/`)

The same recipe on Voxtral Mini 3B ([docs/evaluation_protocol.md](../docs/evaluation_protocol.md), section
17). Backbone 1 is VoxTurn, the model of every file above.

| File | Split | What it holds | Written by | Used for |
|---|---|---|---|---|
| `decision_test_final.json` | test | The main comparison, with the same keys as `main_v04/decision_test_final.json` plus `notes`. `ours` is backbone 2's step 2d read-out, `L1` is backbone 2's original read-out (a new answer head trained with its LoRA adapter). Every other entry of `systems` comes from backbone 1's test-day outputs, unchanged: the script stops unless each one equals the backbone 1 file. `notes`: what `ours` and `L1` mean here, and that row 5's practical advantage reads backbone 1's laptop timings (`main_v04/latency_dev.json`), because backbone 2's speed was not measured. | `src/bb2_test.py report` (through `src/p1v4_test_report.py`) | README and docs/results.md, section 8 |
| `grounding_v06_test.json` | test | The matched grounding test and natural-data analyses for backbone 2, with the same keys as `main_v04/grounding_v06_test.json`. `2d` and `old head` are backbone 2's read-outs; every rival entry repeats backbone 1's test-day outputs. Criteria that were not run for backbone 2 (the noise criterion) say so. | `src/bb2_test.py report` (through `src/p1v4_grounding_v06.py`) | README and docs/results.md, section 8 |
| `offsets_2d_test.json` | test | "Stop, movement" with the decision point moved to 0.5 to 2 s after the door onset, for backbone 2's read-outs (`by_offset`) and the criterion on shifted offsets. | `src/bb2_test.py report` (through `src/bb2_report.py`) | docs/results.md, section 8 |
| `diag_offset_decoding_dev.json` | dev | Model-internal check, for each backbone (`backbones.bb1`, `backbones.bb2`): can the time from the door onset to the decision point (0.5, 1.0, 1.5 or 2.0 s; `offsets_after_door_onset_s`) be decoded from the audio summary of the window? Features: `pooled ears` (mean and max over the whole window) or `per-second ears`, at full width or reduced to 64 principal components (`pca64`). `version 1` uses the leaving version of each "stop, movement" group, `versions 1 and 3` adds the version without the knock. `accuracy` (4 classes, chance 0.25) and `mean_abs_error_s` (seconds), each as mean, min and max over 5 seeds of a 5-fold cross-validation grouped by unit. | `src/bb2_diag.py offsets` | docs/results.md, section 8 |

## Keys used in the files

| Key | Name in the README and figures |
|---|---|
| `ours`; `2d` in the grounding, noise and offsets files | Ours (step 2d read-out) |
| `L1`; `old head` in the grounding and offsets files | Same model, original read-out |
| `L3` | Fine-tuned Whisper encoder + features |
| `L2` | Late fusion of frozen audio and text features |
| `T` | Label reader trained on our data (no audio) |
| `T0`, `T0-single` | Text only; text only, current line only |
| `B1-strong` | Tuned rule system |
| `B2-local` | Local LLM reader (Qwen3.5-4B, 4-bit) |
| `B2-fast`, `B2-strong` | Gemini reader, fast model / strong model |
| `B3` (`B3 other setting`: its second prompt setting) | Gemini listening to the audio |
| `B2-ideal` | Reference: Gemini reader given perfect labels |
| `VoxTurn alone`, `No environment`, `Copy previous` | Off-the-shelf VoxTurn; VoxTurn without the situation text; copying the previous decision |
| `step 2 ears`, `2b ears`, `2c ears`, `2d` | Read-out versions step 2, step 2b, step 2c, step 2d (final) |
| `... states` | The same read-out version on last-layer states only (without the audio encoder) |
| `... s1` | The same read-out trained with a second seed |
| `control start N`, `control (8 starts)` | Control read-outs trained on natural labels, 8 random starts |
| `nat ears sN`, `nat sec8 sN` and their 8-seed means | Read-outs with the audio encoder (summary or per second) trained on natural labels |
| `shuffled` | The final read-out with the audio-encoder seconds shuffled |
| `ears probe` | A linear probe on the audio-encoder outputs alone |
| `combination`, `conditional`, `2b conditional` | Other read-out variants tried on the development set |
| `rel v4`, `rel v5`, `notification model v4/v5` | Versions 4 and 5 of the reliability fix (docs/iterations.md) |

Test parts: `r_hear` "respond, hearing", `r_react` "respond, reaction", `s_react` "stop, reaction", `s_move`
"stop, movement", `episode` "episodes". Injected labels: `none`, `c0.3` to `c0.99` (an event label of that
confidence), `mv0.5` to `mv0.99` (a movement label of that confidence). Noise: `dclean`, and `dnoise5` to
`dnoise40` for 5 to 40 dB more background noise. Reliability conditions: `D_off_none` no label,
`T_off_truth` true labels, `C_off_fake` / `C05_off_fake` a false label of confidence 0.99 / 0.5, and `A_on_`,
`B_on_`, `B05_on_` the same with device notifications switched on. Background-noise recordings: `DKITCHEN`
kitchen, `NFIELD` field, `NRIVER` river, `PRESTO` restaurant (DEMAND); `AirConditioner`, `VacuumCleaner`,
`WasherDryer`, `CopyMachine` (the second noise corpus).

`grounding_v06_dev.json` contains `NaN` for values that cannot be computed on the development set. Python's
`json` module reads it; strict JSON parsers do not.

## Changes made before publishing

The numbers are unchanged. Every file was re-written with LF line endings and the same layout (indent 1).

| File | Change |
|---|---|
| `decision_test_final.json` | In `readings`, section numbers of the internal analysis-plan document were removed: "(amendment 2.6)" deleted in R8; "amendment 13.1" / "amendment 13.2" shortened to "amendment" in R10 to R12. |
| `grounding_v06_test.json` | In `reported.graded["main grounded, stop, by door recording group"]`, the 61 keys were source IDs of the test-set door recordings; they were renamed `door recording 01` to `door recording 61` in the original order (values unchanged). |
| `grounding_v06_dev.json` | The same renaming for the 5 development-set door recordings (`door recording 1` to `door recording 5`). |
| `offsets_2d_test.json` | Removed `per_unit_right`: right or wrong for each of the 199 test units at each of 4 offsets for 4 read-outs (3,184 entries keyed by test unit IDs). The aggregates in `by_offset` and the criterion are kept. |
| `reliability_confirm_replication.json` | `note` named the path of an internal registration document; replaced by "evidence-reliability confirmation; the claims and their bars were written down before the run". |
| `mirror_vs_ears_replication.json` | `design` named an internal log file and time; replaced by "design written down before computing; descriptive, not one of the registered checks". |
| `newnoise_replication.json` | `note` named the path of an internal plan document; that reference was replaced by "design written down before the run" (the rest of the note is kept). |
| `noise_test.json`, `mirror_vs_ears_newnoise.json`, `latency_dev.json` | No change. |
| `main_v04_bb2/decision_test_final.json` | The same edit of `readings` as for backbone 1's file (its readings were identical before the edit). |
| `main_v04_bb2/grounding_v06_test.json` | The same renaming of the 61 test door recordings as for backbone 1 (`door recording 01` to `door recording 61`; the recordings are in the same order in both files, so equal names mean the same recording). Two notes named the path of an internal plan document; replaced by "docs/evaluation_protocol.md, section 17". |
| `main_v04_bb2/offsets_2d_test.json` | Removed `per_unit_right` (per-unit test results keyed by test unit IDs), as for backbone 1. |
| `main_v04_bb2/diag_offset_decoding_dev.json` | No change. |
