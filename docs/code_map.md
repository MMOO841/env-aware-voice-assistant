# Code map

This file connects the code to the rest of the documentation. It has four parts: the names used in the
code and in result files, the role of every file in `src/` and `configs/`, the order of a full run, and
notes on reading the comments in the code. The evaluation rules themselves are in
[evaluation_protocol.md](evaluation_protocol.md).

## 1. Names in the code and in the README

### 1.1 Systems

| Code name (code and result files) | Name in the README and docs | Main files in `src/` that build, train or run it |
|---|---|---|
| `ours`, `2d`, "main system" (`settings.main` in the freeze record) | Ours: fine-tuned speech LM + listening read-out (read-out step 2d, final) | `p1v4_l1.py` (fine-tuned backbone), `p1v4_readout_2c.py` (training labels and forward pass), `p1v4_readout_2d.py` (read-out fit and dev evaluation), `p1v4_final.py local <split> readouts` (final run) |
| `L1`, `old head`, "L1 (the old head)" | Same model, original read-out | `l1_voxturn.py`, `p1v4_l1.py`, `p1v4_final.py local <split> l1` |
| `L3` | Fine-tuned Whisper encoder + features | `p1v4_l3.py` |
| `L2` | Late fusion of frozen audio and text features | `p1v4_l2.py` (`features`, `train l2`) |
| `T` | Label reader trained on our data (text + detector labels, no audio) | `p1v4_l2.py` (`train t`) |
| `T0` | Text only | `p1v4_l2.py` (`train t0`); situation-text variant `no_sounds` in `p1v4_situation.py` |
| `T0-single`, `t0single` | Text only, current line only | `p1v4_l2.py` (`train t0single`); variant `line_only` |
| `B1-strong`, `b1` | Tuned rule system | `p1v4_b1.py` |
| `B2-local`, `b2local` | Local LLM reader (Qwen3.5-4B, 4-bit) | `p1v4_b2.py` |
| `B2-fast`, `b2fast` | Gemini reader, fast model | `p1v4_api.py` (prompt from `p1v4_b2.py`, transcripts from `p1v4_asr_turbo.py`) |
| `B2-strong`, `b2strong` | Gemini reader, strong model | `p1v4_api.py` |
| `B3`, `b3`; `B3 other setting` | Gemini listening to the audio; its second prompt setting in the grounding test | `p1v4_api.py`, `p1v4_grounding.py api b3` |
| `B2-ideal`, `b2ideal` | Reference: Gemini reader given perfect labels | `p1v4_api.py`, `p1v4_situation.py` (`ideal_inputs`) |
| `VoxTurn alone`, `voxturn` | Off-the-shelf VoxTurn | `p1v4_voxturn.py` (its answers), `p1v4_refs.py` (decision rule) |
| `No environment`, `noenv` | Environment-blind gatekeeper (reference) | `p1v4_refs.py` |
| `Copy previous`, `copy` | Copy the previous decision (reference) | `p1v4_refs.py` |
| `SED`, `sed` | Sound-event detector (fine-tuned PANNs CNN14) | `p1v4_sed.py` |

The two references named in plain words (`p1v4_refs.py`; fixed rules, no settings, reported only):

- **Environment-blind gatekeeper** (`No environment`): the usual turn-taking gatekeeper that ignores all
  sounds and the history. It stops the assistant whenever a line tagged as the user's has just ended,
  unless off-the-shelf VoxTurn calls the line a backchannel (probability at least 0.5), and continues when
  nobody spoke. It answers every line tagged as the user's when VoxTurn's end-of-turn probability is at
  least 0.8, and never answers a line tagged as someone else's.
- **Copy the previous decision** (`Copy previous`): in an episode it repeats the correct answer of the
  previous decision point (at the first point: respond and continue); in single-line groups it always
  responds and continues. It shows how much a model could score by carrying the state forward alone.

### 1.2 Read-out versions and other read-outs

All read-outs below sit on the same fine-tuned model (frozen VoxTurn plus our LoRA adapter). They are
logistic regressions, one for "stop" and one for "respond". In the code, "states" or "decision vector"
means the language model's last-layer states at the two answer-option positions (2,048 numbers); "ears"
means the audio encoder's output averaged and max-pooled over the 8 s window (2,048 numbers); "per-second
ears" or "sec8" means the same mean and max for each of the 8 one-second chunks (16,384 numbers).

| Code name | Name in the docs | File |
|---|---|---|
| `old head` | Original read-out (trained with the adapter on natural labels) | `p1v4_l1.py` |
| `step 2 states`, `step 2 ears` | Step 2, on the states only / with the ears | `p1v4_readout.py` |
| `2b states`, `2b ears` | Step 2b | `p1v4_readout_allclass.py` |
| `2c states`, `2c ears` | Step 2c | `p1v4_readout_2c.py` |
| `2d` | Step 2d (final) = ours | `p1v4_readout_2d.py` |
| `conditional`, `2b conditional` | Conditional read-out: the original read-out where the label and an audio probe agree, the step 2 (or 2b) read-out where they disagree | `p1v4_conditional.py` |
| `combination` | Average of the original read-out's and the fine-tuned Whisper model's scores | `p1v4_combo_pilot.py` |
| `control start 0` ... `control start 7`, `control (8 starts)`, `stored control` | Natural-label control: the states-only read-out trained on natural labels, averaged over 8 random starts | `p1v4_grounding.py control`, `p1v4_control_starts.py` |
| `shuffled` | Step 2d design with the 8 seconds in random order (criterion G4c) | `p1v4_readout_extra.py` |
| `nat ears s0` ... `s7`, `nat sec8 s0` ... `s7` | Step 2c-ears and step 2d designs trained on natural labels, 8 seeds (criterion G6a) | `p1v4_readout_extra.py` |
| `2c states s1`, `2d s1` | The same read-outs refitted from a second random start (reported only) | `p1v4_readout_extra.py` |
| ears probe | Linear probe on the ears: is a real ring, doorbell or knock in the window? | `p1v4_conditional.py probe` |

### 1.3 Data, parts and labels

| Code name | Meaning |
|---|---|
| `r_hear`, `r_react`, `s_react`, `s_move`, `episode` | Test parts: respond, hearing; respond, reaction; stop, reaction; stop, movement; episodes |
| `floor` | Floor checks. Kinds `D` (phone rings during a question, user ignores it), `Dk` (same with a knock), `HE` / `H-E` ("hold on, someone's at the door" while the assistant is silent), `HS` / `H-S` (the same while it talks), `barge` (ordinary interruption), `quiet` (nobody speaks) |
| `report` (`F3`, `B2`), `echo` | Report-only cells: a family member calls the user from another room (`F3`); the user turns to talk with someone in the room (`B2`, not the Gemini reader); faint echo of the assistant (`echo`) |
| `test_only_phrase`, `piper`, `vibration`, `new_rec`, `prompt2` | Unseen subsets (test and replication only): test-only phrasings, the Piper voices, phone vibration, recordings uploaded after June 2023, episodes from a second script prompt |
| `audio`, "audio-decisive part", "must-listen part" | Sound-only part: all "respond, hearing" groups plus the "stop, movement" groups whose visitor does not speak |
| `S`, `R`; `stop`/`continue`, `respond`/`not` | The two decisions and their answers |
| unit, version, `k` | A group of versions (or one episode), one version of it (1 to 4), the k-th decision point in it |
| `full`, `no_sounds`, `line_only` | Variants of the situation text (all text readers; text only; text only, current line only) |
| `train_d00` ... `train_d11` | The 12 renders ("draws") of the training split; episodes are in draws 0 to 3 |
| `<split>_dclean`, `<split>_dnoise5`, `<split>_dnoise10` | Folders of the noise analysis (clean re-run, +5 dB and +10 dB background noise) |
| `<split>_dnoise20` ... `<split>_dnoise40`; `replication_dms0` ... `replication_dms40` | Folders of the reliability checks: the original background noise 20 to 40 dB louder; appliance noise from MS-SNSD at each unit's own level (`dms0`) and 10 to 40 dB louder |
| `rel v4`, `rel v5`, `rel v5 label-free only` | Reliability versions 4 and 5 and version 5's label-free control (evaluation_protocol.md, section 13.2) |
| `A_on_none`, `B_on_fake`, `B05_on_fake`, `C_off_fake`, `C05_off_fake`, `D_off_none`, `T_off_truth` | Input conditions of the reliability checks: device notifications on or unavailable, combined with labels removed, false labels (confidence 0.99, or 0.5 in `B05` and `C05`) or true labels (section 13.3) |
| M1, M2, V4a, V4b, V5s, TD1, TD2, V5n, V5o, V5k | Groups of the registered reliability claims (evaluation_protocol.md, section 13.4) |
| `natural`, `none`, `c0.3` ... `c0.99`, `mv0.5`, `mv0.9`, `mv0.99` | Conditions of the matched grounding test (evaluation_protocol.md, section 10.2) |
| G1a ... G7b | Grounding criteria (evaluation_protocol.md, section 10.7); G5 is the noise criterion (section 11) |
| `1 M-listen`, `2 M-data`, `3 not M`, `4 replication`, `5 I`, `6 S` | Rows of the decision table (evaluation_protocol.md, section 8.3): clear win from listening; clear win from the data; win that fails a side condition; one check on the replication split; tie with a practical advantage; otherwise |
| `plan_as_frozen` | The same decision table with the original read-out as "ours" (reported beside the main table) |
| `probabilities`, `answers` | Scoring mode of a Gemini system: token probabilities as scores, or its one-word answers |
| `K6`, `K24`, `K48`; `think`, `nothink`; `_note`, `_prec`; `A0`, `A6` | Gemini settings: number of prompt examples; default thinking or none; prompt with the front-end note / with the note and the detector reliability table; Gemini listening with 0 or 6 audio examples |
| `P1V4_FINAL_RUN=1` | Environment variable that, together with the freeze record, opens the sealed splits |

### 1.4 Second backbone

The recipe repeated on Voxtral Mini 3B ([evaluation_protocol.md](evaluation_protocol.md), section 17). The
code names of sections 1.1 to 1.3 keep their meaning on backbone 2.

| Code name | Meaning |
|---|---|
| `bb1`, `bb2`; "first backbone", "second backbone" | Backbone 1 (fine-tuned VoxTurn) and backbone 2 (fine-tuned Voxtral). `bb1` and `bb2` are also command arguments (`bb2_report.py settings bb1`, `bb2_diag.py d1 bb2`), keys in `results/main_v04_bb2/diag_offset_decoding_dev.json` and parts of the other diagnostic file names |
| `BB1`, `BB2`; `RES1`, `RES2` | Run folders `data/p1v4/runs` and `data/p1v4/runs_bb2`; results folders `results/main_v04` and `results/main_v04_bb2`. Backbone 2's files keep backbone 1's names and layout |
| `old head`, `L1` (backbone 2) | Backbone 2's original read-out: a new answer head trained with its LoRA adapter, from zero, since Voxtral has no option head of its own |
| `ours`, `2d`, "main system" (backbone 2) | Backbone 2's step 2d read-out |
| "L1 family" | The original read-out and every read-out on the same fine-tuned model, that is, everything that depends on the backbone (`L1_FAMILY_SETTINGS` in `bb2_test.py`) |
| `VoxtralVT`, `L1vx` | Voxtral behind the interface of VoxTurn's wrapper; the model class of `p1v4_l1.py` built on it |
| states, ears, per-second ears (backbone 2) | The same summaries as in section 1.2 at Voxtral's width: states 2 × 3,072 = 6,144 numbers; ears = mean and max of the window's 100 audio tokens (projector output), 6,144; per-second ears 8 × 2 × 3,072 = 49,152, with 12 or 13 tokens per second |
| `train2c`, `devnat`, `devgrd_<condition>`, `devnoise_<level>_<condition>`, `devoff_d<offset>`, `trainnat`, `trainstep2`, `train2b` | Input sets of backbone 2's forward passes: training split with step 2c's labels; natural dev; dev grounding conditions; dev noise grid; dev moved decision points; training split with natural labels, step 2's labels, step 2b's labels |
| `testnat`, `testgrd_<condition>`, `testoff_d<offset>`, `examples_test` | Test-run inputs: natural test, the ten grounding conditions, the four moved decision points; the original read-out's scoring pass |
| smoke run, `smoke_dev`, `smoke-check` | `bb2_test.py forward dev 64`: the GPU path on the first 64 items of every dev set, compared with the stored dev outputs |
| `Audit`, `deny=True` | Lists every file a report reads from backbone 1's folders; with `deny=True` (test) it stops if one of them depends on the backbone |
| `freeze_bb2.json`, `NOT_RUN` | Backbone 2's freeze record; the "not run" note written where a report would show the noise analysis |
| D1, D2 | The two dev diagnostics of `bb2_diag.py`: D1, does the decision time survive pooling of the ears; D2, audio information in the answer-option states without labels |

## 2. Files

Stages: data generation, data checks, front end, training, read-outs, rivals, scoring, final test runner,
analyses, analyses after the test, demo. "Library" means the file has no command of its own and is imported
by others.

### 2.1 `src/`

| File | Stage | What it does |
|---|---|---|
| `bb2_diag.py` | Analyses (second backbone, dev only) | Model-internal diagnostics of the two differences found between the backbones, with the same code on both: does the decision time survive pooling of the ears (the time-resolved probe and its shuffled-seconds control re-run per backbone; decoding the moved decision point from pooled and per-second ears), and how much audio information do the answer-option states hold without labels. Descriptive; changes nothing frozen. |
| `bb2_pipeline.py` | Read-outs (second backbone) | Builds every input set with backbone 1's code and checks it against backbone 1's stored inputs (`items`); runs backbone 2's forward passes and saves states, ears and per-second ears in backbone 1's file layout (`forward`); fits step 2d and every other read-out with backbone 1's fit code (`fit2d`, `fitall`). |
| `bb2_report.py` | Analyses (second backbone, dev) | Points backbone 1's report code at backbone 2's run folder (`bind`): dev thresholds by the operating-point rule (`settings`; `settings bb1` checks the code on backbone 1's files), the dev grounding report, the moved-decision-point check (`offsets`; `offsets bb1` is its code check), the step 2d dev report and reliability step 1 on dev (`mirror`). Lists every file it reads from backbone 1's folders. |
| `bb2_test.py` | Final test runner (second backbone) | Dev rehearsal of the test-day path, the GPU smoke run and its check, backbone 2's freeze record (`freeze make`, `check`, `amend`), the test inputs (checked against backbone 1's), the test forward passes and the reports with backbone 1's test-day code; stops unless every rival number equals backbone 1's test reports. |
| `bb2_voxtral.py` | Training (second backbone) | Voxtral Mini 3B behind VoxTurn's interface (log-mel, audio tokens, prompt, situation text, isolating mask, answer head); `prep` writes the training and dev examples, `check` compares with Voxtral's own forward pass on the CPU, `train` and `score` run the recipe of `p1v4_l1.py` unchanged. |
| `demo.py` | Demo (README Quick Start) | Runs the trained system end to end on the sample scenes in `data/samples/`: front end (recomputed or stored), situation text, the fine-tuned model with its original read-out and with ours, decisions at the frozen thresholds in `models/settings.json`, printed next to the right answers. |
| `freesound_client.py` | Data generation (library) | Minimal Freesound API client (search, OAuth download of original files, token refresh) used by the two Freesound scripts; never prints a key or token; `check` tests the connection. |
| `fsd50k_extract.py` | Data generation | Extracts the selected clips from FSD50K's multi-part zip archives, from local parts or from Zenodo by HTTP range requests, and checks the CRC-32 of every file. |
| `l1_voxturn.py` | Training (library) | Loads the official VoxTurn inference code from the model download and adds what training needs: a forward pass with gradients, a second LoRA adapter and a second option head; with the new adapter off it reproduces VoxTurn's own outputs. Kept from task version 0.3. |
| `p1_render.py` | Data generation (library) | Audio building blocks from task version 0.3: resampling, active level, cropping, room reverberation, cached text-to-speech (Kokoro, Piper), background-noise segments; used by `p1v4_units.py` and `p1v4_feats.py`. |
| `p1v4_amendment_numbers_dev.py` | Analyses (dev) | Development-set numbers behind the evaluation-rule amendments written before the freeze (selection, answer-mode compensation, ladder criteria). |
| `p1v4_amendment_numbers_2d_dev.py` | Analyses (dev) | The same numbers after step 2d became the main system. |
| `p1v4_answer_only_dev.py` | Rivals | When the Gemini API stopped returning token probabilities: re-chooses each Gemini system's setting on its stored development answers and measures the answer-only loss that is subtracted on the test set. |
| `p1v4_api.py` | Rivals; final test runner | The Gemini systems: prompts, calls (temperature 0), scores from token probabilities or answers, setting choice on dev; the answers-only drift rate before the freeze (`answer-drift`); test-day commands `probe`, `drift-check`, `rechoose`, `final`, `switch`. |
| `p1v4_asr_turbo.py` | Front end | Whisper large-v3-turbo transcripts of the same speech cuts, for the Gemini systems' situation text and their prompt examples. |
| `p1v4_auto_check.py` | Data checks | Automatic checks of a rendered split (file hashes, lengths, peak and loudness levels, timings); prints no content. |
| `p1v4_b1.py` | Rivals | Tuned rule system: a two-state rule over the situation text's inputs; evaluates all 2,488,320 settings on dev exactly and keeps the best. |
| `p1v4_b2.py` | Rivals | Prompt building shared by all LLM readers (behaviour rules + K training examples + the case) and the local LLM reader (Qwen3.5-4B, 4-bit) with a cached prompt prefix. |
| `p1v4_best_on_test.py` | Final reports | The non-confirmatory row with each rival at its best setting on the test set; its rule was fixed before any test number existed. |
| `p1v4_calibrate.py` | Scoring | Calibrates the interval method by simulation on the development-set structure, before the development checkpoint. |
| `p1v4_combo_pilot.py` | Read-outs (exploratory) | Dev pilot of averaging the original read-out's and the Whisper model's scores; its `combine` function is reused by the final runner. |
| `p1v4_conditional.py` | Read-outs (exploratory) | Fits the ears probe and evaluates the conditional read-out on dev. |
| `p1v4_context.py` | Front end | Per decision point: transcripts, speaker scores and tags, detected sounds and the situation text; `threshold` chooses the user / not-the-user cut on dev. |
| `p1v4_control_starts.py` | Read-outs (grounding control) | Refits the natural-label control from 8 fixed random starts and evaluates its criteria on dev. |
| `p1v4_data.py` | Shared data access; test lock | One row per decision window with gold labels; loads train and dev; opens test and replication only with the freeze record and `P1V4_FINAL_RUN=1`; checks the split seals (`verify-seal`). |
| `p1v4_detector_precision.py` | Rivals | How reliable the detector's labels are on the development set, written into `configs/b2_detector_reliability_v1.txt` for the Gemini readers' fairness prompts. |
| `p1v4_dev_report.py` | Scoring (dev checkpoint) | Every system at its dev operating point on dev: five-part mean, parts, floors, sound-only part, intervals against the rivals, and the basic checks. |
| `p1v4_ears_time_control.py` | Analyses (dev) | Control for the time-resolved audio-encoder diagnostic, added after its results: the same probe with the 8 seconds shuffled per window (all of them, or all but the last); `bb2_diag.py d1` re-runs it on both backbones. |
| `p1v4_ears_time_diag.py` | Analyses | Time-resolved audio-encoder diagnostic (probes with whole-window, three-segment, last-second and per-second pooling) that led to step 2d; re-run on test as a described item. |
| `p1v4_feats.py` | Front end | Speech cuts at the scripted times, Whisper-small transcripts, ECAPA speaker embeddings (with enrolment passages), the detector's sound list; checks that the stored windows equal the re-mixed audio. |
| `p1v4_final.py` | Final test runner | Scores every local system on one split from its frozen files, without retraining; `readouts` runs the batch-of-8 forward pass and scores every read-out candidate; `compare dev` is the rehearsal check. |
| `p1v4_freeze.py` | Final test runner | Writes the freeze record (sha256 of every file the final run reads, plus the frozen settings), checks it (`check`) and records bug-fix re-hashes (`amend`). |
| `p1v4_grounding.py` | Analyses (grounding test runs) | Builds the matched groups and conditions and runs every system on them (local models, VoxTurn answers, Gemini calls); `control` trains the single natural-label control. |
| `p1v4_grounding_report.py` | Analyses (grounding report, first version, dev only) | Measurements 1 to 4 and the criteria on dev; its bootstrap, t-interval and concordance functions are used by the later reports. |
| `p1v4_grounding_v06.py` | Final reports | The fixed (version 0.6) grounding report on dev, test or replication: criteria G1a to G7b, reliance tables, natural comparisons and extra rows; `selftest` checks the subset and invalid-group rules on a changed dev copy. |
| `p1v4_inject_pilot.py` | Analyses (exploratory) | Adds one false "phone" label to every dev point without a real event and measures each local system's reaction. |
| `p1v4_l1.py` | Training | Fine-tunes VoxTurn with a new LoRA adapter and option head for the stop and respond questions; keeps the epoch chosen on dev; scores dev. |
| `p1v4_l2.py` | Training | Builds the feature blocks (audio, detector, text, line) and trains the late-fusion model, the label reader and the two text-only controls; configurations compared on dev. |
| `p1v4_l3.py` | Training | Fine-tunes the Whisper-base encoder with a two-layer head over the same non-audio features as the label reader. |
| `p1v4_latency.py` | Analyses | Compute time per decision (median, p90) of every local system on the laptop GPU, front end included. |
| `p1v4_leak_check.py` | Data checks | Asserts that recordings, voices, rooms, noise, lines and TV speakers stay in their split, that recording-group caps hold, and that the situation text does not depend on gold labels. |
| `p1v4_mirror_vs_ears.py` | Analyses after the test | Descriptive relation, per noise type and level, between the damage to ours' hearing (accuracy lost without labels) and ours' deficit against the original read-out with true labels; also draws the figures. |
| `p1v4_newnoise.py` | Analyses after the test | The check with new noise types: fetches and screens appliance noise from MS-SNSD, re-renders the replication units with it at five levels, and runs the confirmation's forward pass and statistics on them. |
| `p1v4_noise_pilot.py` | Analyses (dev) | Development-set pilot of the noise analysis: re-renders with louder background noise and measures how much each system loses. |
| `p1v4_noise_final.py` | Analyses; final reports | Noise analysis: re-renders the sound-only parts with louder background noise, re-runs the front end and every system at frozen settings, reports criterion G5. |
| `p1v4_probe_pilot.py` | Read-outs (exploratory) | Linear probes on the audio encoder and on several language-model layers: does the fine-tuned model hear the real event even where its decision follows the label? |
| `p1v4_readout.py` | Read-outs | Step 2: retrains only the read-out, on training data whose event labels are made independent of the truth; shared read-out code (forward pass, fitting, scoring). |
| `p1v4_readout_2c.py` | Read-outs | Step 2c: step 2b with false labels copied from the timing of real labels. |
| `p1v4_readout_2d.py` | Read-outs | Step 2d (final): states plus per-second ears; dev evaluation, the shifted-decision-point check (`offsets`) and the decision-table rehearsal (`report`). |
| `p1v4_readout_decision_dev.py` | Read-outs (dev) | Decision-table dry run of the read-out candidates on the development set. |
| `p1v4_readout_allclass.py` | Read-outs | Step 2b: every detector class made independent of the truth. |
| `p1v4_readout_extra.py` | Read-outs (grounding criteria) | The extra read-outs for criteria G4c and G6 and the second random starts. |
| `p1v4_refs.py` | Rivals (references) | Off-the-shelf VoxTurn, the environment-blind gatekeeper and copy the previous decision. |
| `p1v4_reliability_cluster.py` | Analyses after the test | Report-only interval of the 19 key reliability claims with five cluster factors (t interval), computed after the main result. |
| `p1v4_reliability_confirm.py` | Analyses after the test | The registered confirmation on the replication split: front end, matched groups, noisy renders, forward passes in seven input conditions, the claim register and its hash file, and the evaluation with the fixed reading rule. |
| `p1v4_reliability_dev.py` | Analyses after the test (dev only) | Reliability step 1: the matched dev groups with no, false or true labels at five noise levels, run through the frozen systems and the audio probe. |
| `p1v4_reliability_fix.py` | Analyses after the test (dev only) | Reliability fix, version 1: the step 2d read-out retrained on mixed label regimes and noisy training audio. |
| `p1v4_reliability_fix2.py` | Analyses after the test (dev only) | Version 2: adds a clarity signal (a probe from the audio encoder to the signal-to-noise ratio) and clarity-dependent read-out weights. |
| `p1v4_reliability_fix3.py` | Analyses after the test (dev only) | Version 3: decoupled training labels plus a device-notification line in the situation text (notifications on or unavailable). |
| `p1v4_reliability_fix3_diag.py` | Analyses after the test (dev only) | Read-only diagnostic of version 3: how far a notification line moves the decision, and the base rates. |
| `p1v4_reliability_fix4.py` | Analyses after the test (dev only) | Version 4: keeps ours and adds a confidence model (margin, chosen side, clarity) and a notification model with structured notification inputs. |
| `p1v4_reliability_fix5.py` | Analyses after the test (dev only) | Version 5: version 4 plus label dependence (score with labels minus score without), a words-only respond score, a piecewise clarity basis and longer fold training; also its label-free control. |
| `p1v4_reliability_step4_facts.py` | Analyses after the test (dev only) | Read-only facts checked before the version 4 plan (notification timing and answer base rates; no model output). |
| `p1v4_reliability_step5_facts.py` | Analyses after the test (dev only) | Read-only facts checked before the version 5 plan (training texts that change when labels are removed; who says the respond lines). |
| `p1v4_render_split.py` | Data generation | Plans and renders one split (units, versions, 8 s windows), enforces the recording-group caps, prints the seal of a sealed split. |
| `p1v4_score.py` | Scoring (library) | Group scoring, five-part mean, floors, sound-only and unseen parts, the dev operating-point rule, cluster-bootstrap intervals and their calibration. |
| `p1v4_sed.py` | Front end | Fine-tunes the PANNs CNN14 detector on seven classes; writes frame outputs and embeddings of every window. |
| `p1v4_shortcut_check.py` | Data checks | A classifier on generator metadata only, per contrast; every gating AUC must be at most 0.55 before training. |
| `p1v4_situation.py` | Front end (library) | Builds the situation text in its three variants, and the perfect-perception inputs of the reference. |
| `p1v4_step0_confidence.py` | Analyses after the test | Descriptive, from the stored noise-analysis outputs: whether each local system's confidence drops as the sound gets worse (confident share, confident errors, AUROC). |
| `p1v4_step8_check.py` | Final test runner | Test-day completeness check (step 8, written before any metric): every API and local output present, no missing answer, modes consistent, every answer asked after a passed drift check of the same day; reads no score. |
| `p1v4_test_front.py` | Final test runner | Runs the whole front end on a sealed split in the test-day order. |
| `p1v4_test_report.py` | Final reports | Decision report: every system at its frozen setting, differences with intervals, the answer-loss adjustment, the decision table, the table with the original read-out as ours, per-noise-recording rows. |
| `p1v4_trust_curves.py` | Analyses | How often each system acts on a ring or knock label, by the label's confidence, for real and false labels (dev). |
| `p1v4_units.py` | Data generation (library) | Assets per split, seeded plans for every part, timelines from real TTS durations, mixing and the sample-identity check. |
| `p1v4_voxturn.py` | Front end | Off-the-shelf VoxTurn's five answers for every window (late-fusion features; inputs of two references). |
| `step0_allocate.py` | Data generation (version 0.3) | Assigns whole recording groups to test, replication, dev and train with a seeded rule: minimum clip and uploader counts for test and replication, or fixed proportions (for phone and doorbell groups). |
| `step0_assign_lines.py` | Data generation (version 0.3) | After the two-sided check: splits the conversations (300 / 100 / 600 / 600) and the version 0.3 line pools (each unique line in one split), keeps test-only flip phrasings apart, and checks that each main cell has at least 5 passing lines. |
| `step0_check_batches.py` | Data checks (version 0.3) | Builds and collects the two-sided check of the flip lines and "hold on" lines; its text normalisation is reused by later scripts. |
| `step0_fetch_assets.sh` | Data generation | Downloads the pinned Kokoro voices, the Piper voice and whisper-base, plus DEMAND (md5-checked) and RIRS_NOISES. |
| `step0_curate_pools.py` | Data generation (version 0.3) | Builds the phone, doorbell, knock and decoy pools (and two unseen event types) from FSD50K titles, tags and descriptions by fixed rules, before any audio is used. |
| `step0_fingerprint.py` | Data checks (version 0.3) | Audio fingerprints with a calibrated match threshold; "same sound" groups (joined with "same uploader") for the event and decoy recordings. |
| `step0_freesound_subset.py` | Data generation (version 0.3) | Recordings uploaded to Freesound after June 2023 (doorbell, knock, phone ring, vibration, intercom) for the unseen subset: search, download, fingerprint check against all older sources, uploader groups to test or replication 2:1. |
| `step0_fsd50k_counts.py` | Data checks (version 0.3) | Counts FSD50K clips per event and decoy class (licence, single source, rating, uploaders) from the metadata only. |
| `step0_generate_lines.py` | Data generation (version 0.3) | Generates the conversations and the version 0.3 line pools (flip lines, "hold on" lines, family-call and side-talk lines, return lines) with fixed prompts (gpt-6.1-sol). |
| `step0_musan_match.py` | Data checks (version 0.3) | Drops pool clips whose fingerprint matches a MUSAN noise file (part of VoxTurn's training data). |
| `step0_split_aux.py` | Data generation (version 0.3) | Seeded splits of the Kokoro voices, the Piper speakers, the simulated rooms (by room) and the DEMAND noise environments. |
| `step0v4_episodes.py` | Data checks | Episode structure rules, decision points from the script, time-ordered check items and their collection (unclear episodes are dropped). |
| `step0v4_freesound_doors.py` | Data generation | Door recordings uploaded to Freesound after June 2023, for test and replication only: search, download, fingerprint check, split by uploader. |
| `step0v4_generate.py` | Data generation | Generates line pools, per-conversation lines and episode scripts with fixed prompts (gpt-6.1-sol), including the second prompt for unseen episodes. |
| `step0v4_lines.py` | Data checks | Normalises and checks the new lines, then splits them by phrasing (train 50%, dev 15%, test 20%, replication 15%). |
| `step0v4_new_classes.py` | Data generation | Selects the footstep, door and cupboard or drawer recordings from FSD50K by fixed text rules, checks them by fingerprint (MUSAN matches dropped) and splits them by uploader and sound group. |
| `step0v4_rir_positions.py` | Data generation | Estimates each simulated room response's source distance and labels the near, across-the-room and door positions used by the renderer (footsteps moving away use increasing distances). |
| `step0v4_speaker_overlap.py` | Data checks | Compares the TV speakers' names with the identities in the speaker-ID model's training data (VoxCeleb; VGGFace2 as a broader reference) and writes the list of excluded TV speakers. |
| `step0v4_tv_speech.py` | Data generation | The real half of the TV or radio voices from VoxPopuli (utterances of 6 to 20 s, at most 8 per speaker; speakers split train 50%, dev 15%, test 20%, replication 15%). |

### 2.2 `configs/`

| File | Stage | What it does |
|---|---|---|
| `policy_v2_prompt.txt` | Rivals | The behaviour rules every LLM reader gets in its prompt (the same rules define the gold answers). |
| `b2_note_v1.txt` | Rivals | Note for the Gemini readers that the situation text comes from automatic tools that can be wrong (settings ending in `_note` or `_prec`). |
| `b2_detector_reliability_v1.txt` | Rivals | The detector's measured reliability on dev by class and confidence band, added in the `_prec` settings of the Gemini readers and in one Gemini listening setting. |
| `checker_prompts_v1.md` | Data checks (version 0.3) | Prompts of the two-sided line check (flip lines, "hold on" lines, decoy sound and line pairs). |
| `checker_prompts_v2.md` | Data checks | Prompts of the new-line check and the time-ordered episode check. |

## 3. Order of a full run

Commands are run from the repository root as `python -X utf8 src/<script>.py ...` (the usage lines in the
docstrings say `scripts/`; the files are the same). Model steps use the GPU environment
(`requirements.txt`); text-to-speech steps use the data environment (`requirements-data.txt`). The list
groups the scripts by stage; [SETUP.md](../SETUP.md), section 6, gives the exact command order in which each
step's inputs already exist.

**A. Data preparation**

1. Downloads: `bash src/step0_fetch_assets.sh` (the pinned Kokoro and Piper voices, whisper-base, DEMAND and
   RIRS_NOISES); the FSD50K metadata and audio come from its Zenodo record.
2. Event and decoy recordings: `step0_fsd50k_counts.py <FSD50K dir> <out>`, `step0_curate_pools.py <selection>
   <out>`, `fsd50k_extract.py <parts dir | remote> <selection> <out dir>`, `step0_fingerprint.py <clip dir>
   <selection> <extra clip dir | -> <out>`, `step0_musan_match.py <clip dir> <selection> <MUSAN noise dir>
   <fingerprints> <out>`, then `step0_allocate.py <pools> <fingerprints> <out>
   [--proportional=doorbell,phone]`.
3. Recordings uploaded after June 2023: `freesound_client.py check`, then `step0_freesound_subset.py search`,
   `download`, `check`.
4. `step0_split_aux.py <out>` (voices, rooms, noise environments).
5. New recording classes and TV speech: `step0v4_new_classes.py select`, `check`, `allocate`;
   `step0v4_freesound_doors.py search`, `download`, `check`, `assign`; `step0v4_tv_speech.py <VoxPopuli
   parquet>`; `step0v4_speaker_overlap.py <VoxCeleb1 meta> <VoxCeleb2 names> <VGGFace2 labels> <Parliament
   data dir>`; `step0v4_rir_positions.py`.
6. Version 0.3 conversations and lines: `step0_generate_lines.py <key> <out dir> all`, the two-sided check
   with `step0_check_batches.py prepare`, `stage2`, `collect` (`configs/checker_prompts_v1.md`), then
   `step0_assign_lines.py`.
7. New lines: `step0v4_generate.py <key> pools` and `convlines`, then `step0v4_lines.py prepare`, the
   checker answers (`configs/checker_prompts_v2.md`), `step0v4_lines.py collect` and `assign`.
8. Episode scripts: `step0v4_generate.py <key> episodes` and `episodes2`, then `step0v4_episodes.py validate`,
   `prepare` (one round per decision-point index), the checker answers, `collect`; dropped episodes are
   regenerated with a new seed.
9. Render: `p1v4_render_split.py train --draw D` for D = 0 to 11, and `p1v4_render_split.py dev`.
10. `p1v4_shortcut_check.py` (train against dev; every gating AUC at most 0.55).
11. `p1v4_render_split.py test` and `replication`: automatic checks only (`p1v4_auto_check.py`), seal
    printed.
12. `p1v4_leak_check.py <render dirs>` on all splits.

**B. Front end (train and dev)**

13. `p1v4_sed.py train <out>`, then `p1v4_sed.py windows <checkpoint> train|dev <prefix>`.
14. `p1v4_feats.py enrol-tts train|dev` (data environment), then `p1v4_feats.py run <folder> <detector.pt>`
    for `dev` and `train_d00` ... `train_d11`.
15. `p1v4_context.py threshold` (speaker-tag threshold on dev).
16. `p1v4_voxturn.py train|dev <out.json>`; `p1v4_asr_turbo.py dev` and `examples`.
17. `p1v4_l2.py features train` and `features dev`.

**C. Training and local rivals**

18. `p1v4_l1.py train <out>` and `p1v4_l1.py score <out> dev`.
19. `p1v4_l2.py train l2|t|t0|t0single <out>`; `p1v4_l3.py train <out>`.
20. `p1v4_b1.py dev <out>`; `p1v4_b2.py local <out>` (then `rechoose`); `p1v4_refs.py dev <out>`.
21. `p1v4_calibrate.py` (interval calibration), `p1v4_dev_report.py` (development checkpoint) and
    `p1v4_latency.py <out.json>`.

**D. Gemini systems on dev**

22. Detector reliability table `configs/b2_detector_reliability_v1.txt` (`p1v4_detector_precision.py`).
23. `p1v4_api.py run <system> <setting> <key>` for every candidate setting, then `p1v4_api.py choose <system>`;
    re-choice on the stored answers for answers-only scoring (`p1v4_answer_only_dev.py`); `p1v4_api.py
    answer-drift <system> <key>`. Then `p1v4_dev_report.py` again.

**E. Read-out iterations and the grounding test on dev**

24. Pilots: `p1v4_inject_pilot.py`, `p1v4_trust_curves.py`, `p1v4_combo_pilot.py`,
    `p1v4_probe_pilot.py extract train|dev` and `fit`.
25. Step 2: `p1v4_readout.py extract train`, `extract dev`, `fit`, `eval`; conditional read-out:
    `p1v4_conditional.py probe`, `eval`.
26. Grounding runs on dev: `p1v4_grounding.py groups dev`, `l1 dev`, `voxturn dev`, `local dev`, `control`,
    `api <system> dev <key>`; first report `p1v4_grounding_report.py dev --calibrate`.
27. Step 2b: `p1v4_readout_allclass.py extract`, `fit`, `eval`; `p1v4_control_starts.py fit`, `dev`.
28. Step 2c: `p1v4_readout_2c.py edits`, `extract`, `fit`, `eval`; diagnostic `p1v4_ears_time_diag.py
    extract`, `probe`.
29. Step 2d: `p1v4_readout_2d.py ears`, `fit`, `eval`, `offsets`, `report`; `p1v4_readout_extra.py fit`, `dev`.
30. Noise analysis rehearsal: `p1v4_noise_final.py render|front|local dev <variant>`, `api <system> <key> dev
    <variant>`, `report dev`.
31. `p1v4_grounding_v06.py dev` and `p1v4_grounding_v06.py selftest`.

**F. Freeze**

32. `p1v4_feats.py enrol-tts-voices test` and `replication` (data environment; enrolment audio only).
33. Rehearsal: `p1v4_final.py local dev`, `p1v4_final.py compare dev`, `p1v4_test_report.py dev --final`,
    `p1v4_freeze.py make` (rehearsal record).
34. `p1v4_freeze.py make results/main_v04/freeze.json` (clean working tree; never overwritten).

**G. Test day** (from step 36 on, every command is prefixed with `P1V4_FINAL_RUN=1`)

35. `p1v4_freeze.py check`; `p1v4_api.py probe <key> --split test`; `p1v4_api.py drift-check <system> <key>
    --split test` for each Gemini system (`rechoose` only after a failed check).
36. `p1v4_data.py verify-seal test --files`; `p1v4_test_front.py test`.
37. `p1v4_final.py local test` with, in this order, `l1`, `l3`, `readouts`, `t`, `l2`, `t0`, `t0single`, `b1`,
    `refs`, `b2local`.
38. `p1v4_api.py final b2fast|b2strong|b3|b2ideal <key> --split test`; the API rivals' best-on-test row
    (`p1v4_best_on_test.py`).
39. `p1v4_grounding.py groups test`, `l1 test`, `voxturn test`, `local test`,
    `api b3|b2fast|b2strong test <key> --repeat 34`, and `api b3 test <key> --setting <other> --answers 1`.
40. `p1v4_noise_final.py render|front|local test <variant>` and `api <system> <key> test <variant>` for
    `dclean`, `dnoise5` and `dnoise10`.
41. Completeness check: `p1v4_step8_check.py test` (reads no score).
42. Metrics, in this order: `p1v4_readout_2d.py offsets test`, `p1v4_ears_time_diag.py probe test l1_none`,
    `p1v4_noise_final.py report test`, `p1v4_test_report.py test`, `p1v4_grounding_v06.py test`; then the
    report of the API rivals' best-on-test row (`p1v4_best_on_test.py`).

**H. After the test** (evaluation_protocol.md, sections 13 and 14)

43. Descriptive: `p1v4_step0_confidence.py test` (with `P1V4_FINAL_RUN=1`).
44. Reliability exploration on dev: `p1v4_reliability_dev.py render dnoise20 dnoise30 dnoise40`, `forward`,
    `report`; version 1 `p1v4_reliability_fix.py assign`, `render`, `extract`, `fit`, `eval`; version 2
    `p1v4_reliability_fix2.py probe`, `fit`, `eval`; version 3 `p1v4_reliability_fix3.py assign`, `smoke`,
    `train`, `dev`, `fit`, `eval`, then `p1v4_reliability_fix3_diag.py`; version 4
    `p1v4_reliability_step4_facts.py`, then `p1v4_reliability_fix4.py smoke`, `crossfit`, `fit`, `eval`;
    version 5 `p1v4_reliability_step5_facts.py`, then `p1v4_reliability_fix5.py smoke`, `forward`, `crossfit`,
    `fit`, `eval`.
45. Registration: `p1v4_reliability_confirm.py eval dev` (must reproduce the stored dev results),
    `check-texts`, `rehearse`, `register`; the new use of the replication split recorded with
    `p1v4_freeze.py amend`; `p1v4_reliability_confirm.py freeze`; `p1v4_freeze.py check`.
46. Confirmation (each command with `P1V4_FINAL_RUN=1`): `p1v4_reliability_confirm.py front replication`,
    `groups replication`, `render replication dnoise10 dnoise20 dnoise30 dnoise40`, `forward replication`,
    `eval replication`.
47. Report-only rows: `p1v4_reliability_cluster.py replication`; `p1v4_mirror_vs_ears.py compute dev` and
    `compute replication`, then `figure`.
48. New noise types: `p1v4_newnoise.py fetch`, `screen`, `render`, `forward`, `eval`; then
    `p1v4_mirror_vs_ears.py figure-newnoise`.

**I. Demo** (README Quick Start): `demo.py [--scene ID ...] [--front-end live|stored]` on the sample scenes,
with the released weights (see SETUP.md).

**J. Second backbone** (optional; evaluation_protocol.md, section 17; exact commands in SETUP.md, section 8).
Needs the steps above on train and dev, since every input is rebuilt by backbone 1's code and checked
against its stored files.

49. `bb2_voxtral.py prep` and `check`; `bb2_pipeline.py items <set>` for every input set.
50. On one 48 GB GPU: `bb2_voxtral.py train <out>`; `bb2_pipeline.py forward` for `devnat`, `devgrd`,
    `devnoise`, `devoff` and `train2c`; `fit2d`; `fitall`; `forward` for `trainstep2`, `trainnat` and
    `train2b`; `fitall` again.
51. Dev evaluation: `bb2_report.py settings bb1` (code check), `settings`, `copy`, `offsets bb1` (code check),
    `offsets`, `grounding`, `readout2d`, `mirror`.
52. Dev diagnostics: `bb2_diag.py prep`, `d1 bb1`, `d1 bb2`, `offsets`, `d2`.
53. Freeze and test run: `bb2_test.py rehearse`, `forward dev 64` and `smoke-check`, `freeze make`; then
    `items`, `forward test` (GPU), `copy`, `readouts`, `report`, every one except `forward test` with
    `P1V4_FINAL_RUN=1`.

## 4. Reading the comments

- **Versions of the task.** "P1 v0.4" in a docstring is version 0.4 of the task design (two decisions:
  stop and respond); this is the version evaluated here. "v0.3" is the earlier version (respond decision
  only). Files marked "version 0.3" above come from it: `p1_render.py`, `l1_voxturn.py` and
  `step0_check_batches.py` are still imported, and the `step0_*` data scripts built the recording pools,
  voices, rooms, noise, conversations and lines that version 0.4 reuses. "Step 0" is the data-building and
  checking stage before any training; "step 0 item N" is item N of its checklist, which differs between
  the two versions (evaluation_protocol.md, sections 2.5 and 15.1).
- **Rule documents.** Comments cite three rule documents by section number: "plan X.Y", "amendment X.Y"
  (also "freeze amendment third draft X.Y" and "amendment 13.2 supplement") and "package X.Y" (also
  "package v0.6 15.x", "v0.6 15.x", "measurement 1" to "measurement 4"). All three are summarised in
  [evaluation_protocol.md](evaluation_protocol.md), which ends with a table from each cited section to the
  heading that explains it.
- **Reviews.** "review M5", "review C-minor 9", "post-fix review M6", "review item 3" and similar are
  findings of the independent AI reviews of the plans and the code (see [ATTRIBUTION.md](../ATTRIBUTION.md)).
- **Reliability scripts.** In `p1v4_reliability_dev.py`, `p1v4_reliability_fix*.py` and the two `*_facts.py`
  scripts, "plan section 5", "plan section 8", "plan 1.2", "the plan's four criteria" and "criteria 1-6"
  refer to the plans of those dev steps, summarised in [iterations.md](iterations.md). "Registration N" in
  `p1v4_reliability_confirm.py` and `p1v4_reliability_cluster.py` refers to the registered reliability check
  (evaluation_protocol.md, section 13); "runbook step N" to the test-day order (section 12.6).
- **Dated notes.** "docs/iterations.md, <date>" points to the dated note on that iteration. "Fixed in the
  log before running" means the design, decision rule and expected result were written down in the dated
  work log before the run (the log itself is not part of this repository).
- **Readings R1 to R12** in `p1v4_test_report.py` are the code's explicit readings of the decision table
  (evaluation_protocol.md, section 8.4).
- **Second-backbone scripts.** In `bb2_*.py`, "docs/evaluation_protocol.md, section 17" points to the summary
  of the second-backbone plan. "Laptop" and "GPU machine" (or "the GPU job") are the two computers used: data
  preparation, reports and diagnostics ran on the laptop; training and the forward passes ran on one 48 GB
  GPU elsewhere. The job scripts for that machine (`bb2_pack.py` and the two `.sbatch` files listed in
  `bb2_test.py`) are not included; the code still names the machine in a few constants, in two hash-file
  names and in the `freeze` sub-command that writes the GPU job's hash list.
- **Scripts named in comments but not in `src/`.** `p1_asr.py`, `p1_latency.py` and `p1_plan_items.py`
  (version 0.3 transcripts, timing and item specs, replaced by `p1v4_feats.py`, `p1v4_latency.py` and
  `p1v4_units.py`); `p1v4_grounding_v05.py` (replaced by `p1v4_grounding_v06.py`); `freesound_authorize.py`
  (one-time Freesound login that stores the access token outside the repository);
  `p1v4_answer_only_update.py`, `p1v4_allclass_dev_report.py`, `p1v4_control_seeds_dev.py`,
  `p1v4_smove_diag_dev.py`, `p1v4_b2_memcheck.py`, `p1v4_b2_cache_diag.py` (one-off dev diagnostics).
  `voxturn.py` is VoxTurn's own inference code and `models.py` the PANNs model code, both loaded from their
  downloads.
