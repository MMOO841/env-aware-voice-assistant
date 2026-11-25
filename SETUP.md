# Setup

How to install the project, run the trained system on the sample scenes (Quick Start), and repeat the
full pipeline. Commands are for a bash shell run from the repository root (on Windows, Git Bash);
differences in PowerShell are noted where they matter. Everything was tested on Windows 11 with Python
3.12.10 and an NVIDIA GeForce RTX 5060 Laptop GPU (8 GB).

| | Quick Start (sections 1 to 4) | Full pipeline (sections 1, 2 and 6) |
|---|---|---|
| What runs | The trained system on the six sample scenes in `data/samples/` | Data generation, training, read-outs, evaluation |
| What it needs | The GPU environment, about 3 GB of model downloads, an NVIDIA GPU with about 4 GB free | Both environments, the outside datasets ([data/README.md](data/README.md)), paid API keys, many GPU hours |

## 1. Python and the environments

Use Python 3.12.10. There is one environment per requirements file:

| Folder | Requirements | Used for |
|---|---|---|
| `.venv-gpu` | `requirements.txt` | Everything with a model: the demo, the front end, training, read-outs, evaluation, and the Gemini rivals (plain HTTPS calls) |
| `.venv` | `requirements-data.txt` | Text-to-speech (Kokoro, Piper) and data generation, on the CPU |
| `.venv-nb` | `notebooks/requirements.txt` | The figures notebook only (section 7) |

**GPU environment** (needed for everything):

```bash
py -3.12 -m venv .venv-gpu              # Linux: python3.12 -m venv .venv-gpu
source .venv-gpu/Scripts/activate       # PowerShell: .venv-gpu\Scripts\Activate.ps1   Linux: source .venv-gpu/bin/activate
python -m pip install -r requirements.txt
python -c "import torch; print(torch.__version__, torch.cuda.is_available())"    # 2.13.0+cu130 True
```

- `requirements.txt` installs PyTorch 2.13.0 built for CUDA 13.0 from the PyTorch package index (the
  file's first line). The NVIDIA driver must therefore support CUDA 13.0.
- The environment takes about 4 GB.

**Data environment** (only for generating data):

```bash
py -3.12 -m venv .venv
source .venv/Scripts/activate
python -m pip install -r requirements-data.txt --extra-index-url https://download.pytorch.org/whl/cu126
```

- The extra index is needed because the file pins `torch==2.13.0+cu126`, which exists only on the
  PyTorch index.
- The file also pins the spaCy English model and the espeak-ng loader that Kokoro uses, so no system
  package had to be installed on the test machine.

**Conventions for every command below:**

- Run from the repository root. The scripts read and write relative paths (`data/`, `results/`,
  `models/`).
- Use `python -X utf8 src/<script>.py`, which gives UTF-8 file and console input and output on Windows.
- The usage lines inside the scripts say `scripts/`. The files are the same, in `src/`.
- GPU scripts cap PyTorch at 85% of the card (`gpu_guard` in `src/p1v4_data.py`). On Windows, CUDA
  otherwise spills silently into system memory and runs about 20 times slower. With the cap, a job that
  does not fit stops with an out-of-memory error.

## 2. Base models

None of these are stored in the repository: `data/models_dl/` is git-ignored, and the Hugging Face
cache is `~/.cache/huggingface/hub` (moved by `HF_HOME`). With the GPU environment active, from the
repository root (about 2.7 GB in total):

```bash
hf download AustinXiao/voxturn --revision fe3d4a319c4e74279544188bd0e8292325852dbc --local-dir data/models_dl/voxturn_fe3d4a3
hf download Qwen/Qwen3-ASR-0.6B-hf --revision 7f1569a48a89f3e3f4dc3a5c9d28bddd903bc76c
hf download openai/whisper-small --revision 973afd24965f72e36ca33b3055d56a652f456b4d added_tokens.json config.json generation_config.json merges.txt model.safetensors normalizer.json preprocessor_config.json special_tokens_map.json tokenizer.json tokenizer_config.json vocab.json
hf download speechbrain/spkrec-ecapa-voxceleb --revision 0f99f2d0ebe89ac095bcc5903c4dd8f72b367286 --local-dir data/models_dl/spkrec-ecapa-voxceleb_0f99f2d
python -c "import os, urllib.request as u; d = 'data/models_dl/panns_d2f4b8c/'; os.makedirs(d, exist_ok=True); b = 'https://raw.githubusercontent.com/qiuqiangkong/audioset_tagging_cnn/d2f4b8c18eab44737fcc0de1248ae21eb43f6aa4/'; [u.urlretrieve(b + s, d + n) for s, n in (('pytorch/models.py', 'models.py'), ('pytorch/pytorch_utils.py', 'pytorch_utils.py'), ('LICENSE.MIT', 'LICENSE.MIT'))]"
```

| Model | Where the code finds it | Size | Licence |
|---|---|---|---|
| VoxTurn v1 (`AustinXiao/voxturn`): inference code and weights | `data/models_dl/voxturn_fe3d4a3/` (`src/l1_voxturn.py`) | 48 MB | CC BY-NC 4.0 |
| Qwen3-ASR-0.6B (`Qwen/Qwen3-ASR-0.6B-hf`), VoxTurn's backbone | Hugging Face cache; VoxTurn's `voxturn.py` loads it at this revision | 1.5 GB | Apache-2.0 |
| Whisper small (`openai/whisper-small`) | Hugging Face cache (`WHISPER` in `src/p1v4_feats.py`) | 0.97 GB | MIT |
| ECAPA-TDNN (`speechbrain/spkrec-ecapa-voxceleb`) | `data/models_dl/spkrec-ecapa-voxceleb_0f99f2d/`; on first use SpeechBrain copies it to `data/models_dl/spkrec-ecapa-voxceleb/` | 89 MB | Apache-2.0 |
| PANNs CNN14 code (`qiuqiangkong/audioset_tagging_cnn`, commit d2f4b8c) | `data/models_dl/panns_d2f4b8c/` (`PANNS_DIR` in `src/p1v4_sed.py`) | 0.1 MB | MIT |

After these downloads the demo needs no network; setting `HF_HUB_OFFLINE=1` makes sure of it. The base
models that only the full pipeline uses are listed in section 6.

## 3. Trained weights

Our own trained files are in `models/`, all under 100 MB ([models/README.md](models/README.md)):

- `l1_new.pt`: the LoRA adapter, with the original read-out;
- `readouts_2d.npz`: ours;
- `settings.json`: the frozen thresholds;
- `speaker_enrolment.json`: the enrolment of the sample users' voices.

The fine-tuned sound-event detector, `sed.pt` (323 MB), is too large for the repository.
1. Download it from this repository's GitHub Release `v1.0`:
   <https://github.com/MMOO841/env-aware-voice-assistant/releases/download/v1.0/sed.pt>
2. Save it as `models/release/sed.pt`. See [models/release/README.md](models/release/README.md); its
   sha256 is `58be2ddacc795e5801bd252ee7c46ba87733c89a8709d759701a5a01ec49d2d3`.

The demo checks the sha256 of all three weight files before it loads them.

## 4. Quick Start: the demo

```bash
source .venv-gpu/Scripts/activate
python -X utf8 src/demo.py                                      # all six sample scenes
python -X utf8 src/demo.py --scene stop_movement --show-text    # one scene, with the situation text the model reads
python -X utf8 src/demo.py --front-end stored                   # stored front-end outputs instead of recomputing them
```

For every decision point of every sample scene, the demo:
- runs the front end: the sound-event detector, Whisper-small transcripts, and speaker tags (user / not
  the user) from ECAPA-TDNN embeddings compared with the user's enrolment;
- builds the situation text and runs the fine-tuned VoxTurn on the 8 s window and the text;
- prints what was perceived: the line just said, its speaker tag, and the detected sounds;
- for each question asked at that point, prints the right answer and then, for ours and for the same
  model's original read-out, the probability and the decision at the frozen threshold.

A line of output looks like this:

```
     27.6 s  just said (user): "Hello." | sounds reaching into the window: a household sound 23.7-26.3 s (1.00)
      respond  right: respond       | ours 0.008 don't respond WRONG | original 1.000 respond       ok
```

**Options:**
- `--scene ID ...`: run only these scenes.
- `--show-text`: print the situation text of every point.
- `--out file.json`: save every probability, decision and input text.
- `--batch N`: windows per forward pass (default 8).
- Paths, when the defaults do not apply: `--samples`, `--weights`, `--detector`, `--models-dl`.

**Front end:**
- `live` (default) recomputes the front end from the audio.
  - The two episodes ship their whole audio, so everything is recomputed as when the training set was
    processed. The detector then runs on the audio from the scene start to each decision point.
  - The four single-line scenes ship only their 8 s windows. Speech inside a window is transcribed and
    tagged, and the detector runs on the window. The user's opening request, which ends before the
    window, keeps its stored transcript and tag.
- `stored` uses the front-end outputs saved with each scene, so every input is exactly the training
  input. The rebuilt situation text is checked against the stored one.

**Expected results on the RTX 5060 Laptop GPU:**
- The models load in 6 to 25 s; the first run, which reads them from disk, is the slowest.
- Each scene then takes 0.3 to 1.3 s for front end and decision model together. The first scene of a
  run takes up to about 1.5 s longer, because the GPU warms up.
- Peak GPU memory is 3.5 GB allocated (4.1 GB reserved). With `--front-end stored` it is 2.7 GB and
  about 0.4 s per scene.
- Ours answers 28 of the 29 scored questions right; the original read-out answers all 29. Ours does not
  respond to "Hello?" after a kitchen sound in `respond_hearing`, version 3 (a training scene).
- In `live` mode the two episodes get exactly the stored situation texts. In the single-line scenes only
  the detected sound times move, by 0.1 to 0.3 s.
- With `--front-end stored`, the probabilities match the training-time forward passes on the same
  windows within 0.008 (ours) and 0.0001 (original read-out). No decision differs.

The demo needs an NVIDIA GPU; the code uses CUDA throughout and has no CPU path.

## 5. API keys and the Gemini rivals

No key is needed for the Quick Start, and no key is stored in this repository. Every script that calls
a paid API takes the path of a JSON key file on the command line and reads one field from it. No
environment variable is read, and no key is printed.

| Service | Used by | Fields read |
|---|---|---|
| Gemini API | `src/p1v4_api.py` (`run`, `probe`, `drift-check`, `final`, `rechoose`, `answer-drift`), `src/p1v4_grounding.py api`, `src/p1v4_noise_final.py api`, `src/p1v4_best_on_test.py ask` | `GEMINI_API_KEY` (or `GOOGLE_API_KEY`) |
| OpenAI API | `src/step0_generate_lines.py`, `src/step0v4_generate.py` (dialogue lines and episode scripts) | `OPENAI_API_KEY` (or `api_key`) |
| Freesound API | `src/freesound_client.py` and the two Freesound download scripts | `~/freesound_env.json` (`FREESOUND_CLIENT_ID`, `FREESOUND_API_KEY`) and `~/freesound_oauth.json` (OAuth tokens; the script that writes this file is not included) |

Name a key file `.gemini_env.json` (or `.openai_env.json`) in the repository root: `.gitignore`
excludes `*.env.json` and `.*_env.json`. For example:

```bash
echo '{"GEMINI_API_KEY": "<your key>"}' > .gemini_env.json
python -X utf8 src/p1v4_api.py run b2fast K48_think .gemini_env.json
```

In Windows PowerShell 5.1, create the file in a text editor instead: there, `echo ... >` writes UTF-16,
and the scripts read the key file as UTF-8.

**Checking the system without a key:**

- The demo (section 4) needs no key.
- The rivals' results are in the aggregate result files in `results/main_v04/` (described in
  [results/README.md](results/README.md) and summarised in [docs/results.md](docs/results.md)):
  - `decision_test_final.json`: the main comparison;
  - `grounding_v06_test.json`: the grounding test;
  - `noise_test.json`: the noise analysis.
  `notebooks/figures.ipynb` redraws the figures from these files (section 7).
- The rivals' per-question answers are not shipped (they are per-item outputs on the test split), so the
  rival rows cannot be recomputed from this repository.
- **Re-running the rivals needs your own Gemini API key and costs real money.** At November 2026 prices:
  - the development-set runs of the four Gemini systems, over all candidate settings, cost about $580;
  - all API work on the development set cost more than $900;
  - the test-set runs cost several times the development-set runs.
- The models used (`gemini-4-hydrogen`, `gemini-4-carbon`) can change or be withdrawn, so new answers can
  differ. On 2026-11-19 the API stopped returning token probabilities. The code then scores the answers
  themselves (`src/p1v4_answer_only_dev.py`; `src/p1v4_api.py probe` tests which mode is available).

## 6. The full pipeline

Read this first:

- **What each script does:** [docs/code_map.md](docs/code_map.md) (section 2) describes every script,
  and its section 3 explains the order below step by step. The evaluation rules are in
  [docs/evaluation_protocol.md](docs/evaluation_protocol.md).
- **Data:** the data are not shipped. The scripts rebuild the dataset (about 36 GB of renders) from the
  outside sources in [data/README.md](data/README.md). The development, test and replication renders,
  and all per-item outputs, are not released.
- **Steps from outside this repository.** A few inputs come from outside, and the scripts that read them
  stop without them:
  - The answers of the line and episode checks came from a second model family (prompts in `configs/`).
    No script here calls it.
  - Some small helper scripts are not included. They wrote:
    - `results/step0_v04/excluded_units.json`, which every data loader reads;
    - `data/p1v4/runs/readout/scores_dev_readout.json` and `scores_dev_combination.json`, read by
      `p1v4_conditional.py eval` and `p1v4_final.py compare`;
    - `results/main_v04/noise_pilot_dev.json` and `amendment_numbers*_dev.json`, read by
      `p1v4_freeze.py`;
    - the Freesound OAuth token file.
- **The freeze and the test run cannot be repeated from this repository alone.** The final test run
  (6.8 and 6.9) reads the freeze record `results/main_v04/freeze.json`, which is not shipped.
  `src/p1v4_freeze.py` expects the original folder layout (code under `scripts/`, plus design documents
  that are not included), and here it stops with "missing files". Those sections document the order of
  the single final run.
- **Time on the laptop GPU:**
  - fine-tuning VoxTurn takes about 9 hours (2 epochs);
  - the detector takes about 40 minutes;
  - the front end and the read-out forward passes over the 86,000 training windows take hours each.
- **`--help`:** most scripts print their usage when given an unknown command, but some start working
  instead. Read the docstring at the top of these files rather than calling them with `--help`:
  - `p1v4_shortcut_check.py` overwrites its result file;
  - `p1v4_latency.py` runs the timing and writes a file named `--help`;
  - `p1v4_dev_report.py`, `p1v4_inject_pilot.py`, `p1v4_trust_curves.py`, `p1v4_combo_pilot.py`,
    `p1v4_detector_precision.py`, `p1v4_answer_only_dev.py`, `p1v4_step0_confidence.py`,
    `p1v4_step8_check.py`, `step0v4_episodes.py`, `step0_split_aux.py` and `step0v4_rir_positions.py`
    start working.

**Base models used only by the full pipeline** (in addition to section 2):

```bash
hf download openai/whisper-base --revision e37978b90ca9030d5170a5c07aadb050351a65bb config.json generation_config.json model.safetensors preprocessor_config.json README.md --local-dir data/models_dl/whisper-base_e37978b
hf download rhasspy/piper-voices --revision 6b9cfaacb4fb2e02a9621227e3c1a48bec9710c3 en/en_US/libritts/high/en_US-libritts-high.onnx en/en_US/libritts/high/en_US-libritts-high.onnx.json en/en_US/libritts/high/MODEL_CARD --local-dir data/models_dl/piper_libritts_high_6b9cfaa
hf download hexgrad/Kokoro-82M --revision f3ff3571791e39611d31c381e3a41a3af07b4987
python -c "import urllib.request as u; u.urlretrieve('https://zenodo.org/records/3987831/files/Cnn14_DecisionLevelAtt_mAP%3D0.425.pth?download=1', 'data/models_dl/panns_d2f4b8c/Cnn14_DecisionLevelAtt_mAP=0.425.pth')"
```

- Whisper base is used by the fine-tuned Whisper rival. The Piper voice is the second text-to-speech
  engine. Kokoro is the main text-to-speech engine: `src/p1_render.py` reads its snapshot from the
  default cache folder `~/.cache/huggingface/hub`, so do not move the cache when you render. The PANNs
  AudioSet checkpoint (331 MB, md5 `c8281ca2b9967244b91d557aa941e8ca`) is needed to train the detector.
- Three more models download into the Hugging Face cache on first use, at pinned revisions:
  - `openai/whisper-large-v3-turbo` (41f01f3), the transcripts for the Gemini readers;
  - `sentence-transformers/all-MiniLM-L6-v2` (1110a24), text embeddings;
  - `Qwen/Qwen3.5-4B` (851bf6e), the local LLM rival.

### 6.1 Data generation (data environment, CPU)

Step 0 selects the recordings, voices, rooms and noise, and fixes the splits (outputs in
`results/step0/` and `results/step0_v04/`; not shipped):

```bash
python -X utf8 src/step0_fsd50k_counts.py <FSD50K extract dir> <output json>
python -X utf8 src/step0_curate_pools.py <selection json> <output json>
python -X utf8 src/fsd50k_extract.py <dir with the archive parts | remote> <selection json> <output dir>
python -X utf8 src/step0_fingerprint.py <clip dir> <selection json> <extra clip dir | -> <output json>
python -X utf8 src/step0_musan_match.py <clip dir> <fp selection json> <MUSAN noise dir> <fingerprint json> <output json>
python -X utf8 src/step0_allocate.py <pools json> <fingerprint json> <output json> [--min name=clips,uploaders ...]
python -X utf8 src/freesound_client.py check
python -X utf8 src/step0_freesound_subset.py search        # then: download, check
python -X utf8 src/step0_split_aux.py results/step0/aux_splits.json
python -X utf8 src/step0v4_new_classes.py select            # then: check, allocate
python -X utf8 src/step0v4_freesound_doors.py search        # then: download, check, assign
python -X utf8 src/step0v4_tv_speech.py <VoxPopuli en_test parquet>
python -X utf8 src/step0v4_speaker_overlap.py <VoxCeleb1 meta csv> <VoxCeleb2 names> <VGGFace2 train labels .npy> <Parliament data dir>
python -X utf8 src/step0v4_rir_positions.py
```

Dialogue lines and episode scripts (OpenAI key; a new run gives different lines). The checker answers,
written from the prompts in `configs/`, go between `prepare` and `collect`:

```bash
python -X utf8 src/step0_generate_lines.py .openai_env.json data/lines_gen all
python -X utf8 src/step0_check_batches.py prepare           # checker answers (configs/checker_prompts_v1.md), then: stage2, collect
python -X utf8 src/step0_assign_lines.py
python -X utf8 src/step0v4_generate.py .openai_env.json pools      # and: convlines
python -X utf8 src/step0v4_lines.py prepare                 # checker answers (configs/checker_prompts_v2.md), then: collect, assign
python -X utf8 src/step0v4_generate.py .openai_env.json episodes   # and: episodes2
python -X utf8 src/step0v4_episodes.py validate             # then: prepare (one round per decision point), checker answers, collect
```

Rendering (`data/p1v4/render/`; episodes only in training draws 0 to 3) and the data checks:

```bash
for D in 0 1 2 3; do python -X utf8 src/p1v4_render_split.py train --draw $D; done
for D in 4 5 6 7 8 9 10 11; do python -X utf8 src/p1v4_render_split.py train --draw $D --parts r_hear,r_react,s_react,s_move,floor; done
python -X utf8 src/p1v4_render_split.py dev
python -X utf8 src/p1v4_shortcut_check.py                   # generator settings alone must not predict the answers
python -X utf8 src/p1v4_render_split.py test
python -X utf8 src/p1v4_render_split.py replication
python -X utf8 src/p1v4_auto_check.py data/p1v4/render/test        # and replication: counts, hashes, levels, seal
python -X utf8 src/p1v4_leak_check.py data/p1v4/render/*/          # no source crosses splits
python -X utf8 src/p1v4_data.py verify-seal dev
```

### 6.2 Front end (GPU environment, except where marked)

```bash
python -X utf8 src/p1v4_sed.py train data/p1v4/runs/sed            # the detector (or copy models/release/sed.pt there)
python -X utf8 src/p1v4_sed.py windows data/p1v4/runs/sed/sed.pt train data/p1v4/feats/sed_win_train
python -X utf8 src/p1v4_sed.py windows data/p1v4/runs/sed/sed.pt dev data/p1v4/feats/sed_win_dev
python -X utf8 src/p1v4_feats.py enrol-tts train                   # data environment: enrolment passages (Kokoro)
python -X utf8 src/p1v4_feats.py enrol-tts dev                     # data environment
for F in dev train_d00 train_d01 train_d02 train_d03 train_d04 train_d05 train_d06 train_d07 train_d08 train_d09 train_d10 train_d11; do
  python -X utf8 src/p1v4_feats.py run $F data/p1v4/runs/sed/sed.pt   # transcripts, speaker embeddings, detected sounds
done
python -X utf8 src/p1v4_context.py threshold                       # speaker threshold, chosen on dev
python -X utf8 src/p1v4_voxturn.py train data/p1v4/feats/voxturn_train.json
python -X utf8 src/p1v4_voxturn.py dev data/p1v4/feats/voxturn_dev.json
python -X utf8 src/p1v4_l2.py features train
python -X utf8 src/p1v4_l2.py features dev
python -X utf8 src/p1v4_asr_turbo.py dev                           # transcripts for the Gemini readers
```

### 6.3 Training and the local rivals (GPU environment)

Each script keeps the epoch or setting chosen on the development set.

```bash
python -X utf8 src/p1v4_l1.py train data/p1v4/runs/l1              # fine-tuned VoxTurn and its original read-out
python -X utf8 src/p1v4_l2.py train l2 data/p1v4/runs/l2           # late fusion
python -X utf8 src/p1v4_l2.py train t data/p1v4/runs/t             # label reader
python -X utf8 src/p1v4_l2.py train t0 data/p1v4/runs/t0           # text only
python -X utf8 src/p1v4_l2.py train t0single data/p1v4/runs/t0single
python -X utf8 src/p1v4_l3.py train data/p1v4/runs/l3              # fine-tuned Whisper encoder
python -X utf8 src/p1v4_b1.py dev data/p1v4/runs/b1                # tuned rule system
python -X utf8 src/p1v4_b2.py local data/p1v4/runs/b2local         # local LLM reader; then: rechoose data/p1v4/runs/b2local
python -X utf8 src/p1v4_asr_turbo.py examples                      # transcripts of the prompt examples
python -X utf8 src/p1v4_refs.py dev data/p1v4/runs/refs            # references
```

### 6.4 Gemini rivals on the development set (paid)

```bash
python -X utf8 src/p1v4_detector_precision.py                      # the detector's measured reliability, for the "_prec" prompts
python -X utf8 src/p1v4_api.py run <system> <setting> .gemini_env.json   # every candidate setting of b2fast, b2strong and b3
python -X utf8 src/p1v4_api.py choose <system>
python -X utf8 src/p1v4_api.py run b2ideal <b2strong's chosen setting> .gemini_env.json
python -X utf8 src/p1v4_answer_only_dev.py                         # scores from the answers (no token probabilities)
python -X utf8 src/p1v4_api.py answer-drift <system> .gemini_env.json
```

- Settings of the readers: K6, K24 or K48 examples, `_think` or `_nothink`, optionally ending in
  `_note` or `_prec`, for example `K48_think_prec`.
- Settings of the listening system: `A0_` or `A6_`, then `think` or `nothink`.

### 6.5 Development reports

```bash
python -X utf8 src/p1v4_calibrate.py                               # checks the interval method on the dev structure
python -X utf8 src/p1v4_dev_report.py                              # every system at its dev operating point
python -X utf8 src/p1v4_latency.py results/main_v04/latency_dev.json
```

### 6.6 Read-out iterations and the grounding test (development set)

```bash
python -X utf8 src/p1v4_inject_pilot.py                            # false labels at every no-event point
python -X utf8 src/p1v4_trust_curves.py
python -X utf8 src/p1v4_combo_pilot.py
python -X utf8 src/p1v4_probe_pilot.py extract train               # then: extract dev, fit
python -X utf8 src/p1v4_readout.py extract train                   # step 2; then: extract dev, fit, eval
python -X utf8 src/p1v4_conditional.py probe                       # then: eval
python -X utf8 src/p1v4_grounding.py groups dev                    # then: l1 dev, voxturn dev, local dev, control
python -X utf8 src/p1v4_grounding_report.py dev --calibrate        # systems without runs yet (the Gemini ones) are skipped
python -X utf8 src/p1v4_readout_allclass.py extract                # step 2b; then: fit, eval
python -X utf8 src/p1v4_control_starts.py fit                      # then: dev
python -X utf8 src/p1v4_readout_2c.py edits                        # step 2c; then: extract, fit, eval
python -X utf8 src/p1v4_ears_time_diag.py extract                  # then: probe
python -X utf8 src/p1v4_readout_2d.py ears                         # step 2d = ours; then: fit, eval
python -X utf8 src/p1v4_readout_extra.py fit
```

### 6.7 Rehearsal on the development set, and the freeze

The rehearsal record, `results/main_v04/freeze_rehearsal.json`, holds the frozen settings. Every
command in this block after the first `make` reads it on the development set.

```bash
python -X utf8 src/p1v4_feats.py enrol-tts-voices test             # data environment; and replication
python -X utf8 src/p1v4_freeze.py make                             # rehearsal record
python -X utf8 src/p1v4_readout_2d.py offsets                      # shifted decision points; then: report
python -X utf8 src/p1v4_grounding.py api b3 dev .gemini_env.json   # and b2fast, b2strong (paid)
python -X utf8 src/p1v4_noise_final.py render dev dnoise10         # then: front dev dnoise10, local dev dnoise10
python -X utf8 src/p1v4_noise_final.py api b2fast .gemini_env.json dev dnoise10   # and b2strong, b3 (paid)
python -X utf8 src/p1v4_noise_final.py report dev                  # after the same steps for the variant dclean
python -X utf8 src/p1v4_final.py local dev                         # every local system from its frozen files
python -X utf8 src/p1v4_readout_extra.py dev
python -X utf8 src/p1v4_final.py compare dev                       # must print REHEARSAL PASS
python -X utf8 src/p1v4_test_report.py dev --final
python -X utf8 src/p1v4_grounding_v06.py dev                       # and: selftest
python -X utf8 src/p1v4_freeze.py make                             # the rehearsal record again, now with every input
python -X utf8 src/p1v4_freeze.py make results/main_v04/freeze.json   # the final record (clean git tree; never overwritten)
python -X utf8 src/p1v4_freeze.py check
```

On a run from scratch these steps form a loop:
- `make` stops while any file on its list is missing.
- Two of those files (`offsets_2d_dev.json` and `readout_extra_dev.json`) are written by steps above that
  read the rehearsal record.
- The original run could proceed because a rehearsal record already existed when these entries were
  added to the list.

### 6.8 Final test run (once)

The test and replication splits open only in this run. The data loaders refuse them unless the freeze
record exists and each command is prefixed with `P1V4_FINAL_RUN=1`. The probe and drift checks of the
Gemini systems come first:

```bash
python -X utf8 src/p1v4_api.py probe .gemini_env.json --split test
python -X utf8 src/p1v4_api.py drift-check b2fast .gemini_env.json --split test     # and b2strong, b3, b2ideal
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_data.py verify-seal test --files
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_test_front.py test
for S in l1 l3 readouts t l2 t0 t0single b1 refs b2local; do P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_final.py local test $S; done
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_api.py final b2fast .gemini_env.json --split test   # and b2strong, b3, b2ideal
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_best_on_test.py ask test .gemini_env.json
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_grounding.py groups test   # then: l1 test, voxturn test, local test
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_grounding.py api b3 test .gemini_env.json --repeat 34   # and b2fast, b2strong
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_noise_final.py render test dnoise10   # front, local, api as on dev; for dclean, dnoise5, dnoise10
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_step8_check.py test       # completeness check; reads no score
```

### 6.9 Reports (only after 6.8 is complete)

```bash
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_readout_2d.py offsets test
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_ears_time_diag.py probe test l1_none
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_noise_final.py report test
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_test_report.py test       # main comparison: results/main_v04/decision_test_final.json
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_grounding_v06.py test     # grounding test: results/main_v04/grounding_v06_test.json
P1V4_FINAL_RUN=1 python -X utf8 src/p1v4_best_on_test.py report test
```

The analyses after the test (the reliability checks on the replication split and the new noise types:
`p1v4_step0_confidence.py`, `p1v4_reliability_*.py`, `p1v4_mirror_vs_ears.py`, `p1v4_newnoise.py`) are
listed in order in [docs/code_map.md](docs/code_map.md), section 3, steps 43 to 48.

## 7. Figures

`notebooks/figures.ipynb` redraws the figures in `docs/figures/`. It reads only the aggregate result
files in `results/main_v04/`, so it needs no GPU, no weights and no data. It has its own small
environment:

```bash
py -3.12 -m venv .venv-nb
source .venv-nb/Scripts/activate
python -m pip install -r notebooks/requirements.txt
python -c "import nbformat; from nbclient import NotebookClient; nb = nbformat.read('notebooks/figures.ipynb', as_version=4); NotebookClient(nb, kernel_name='python3').execute(); nbformat.write(nb, 'notebooks/figures.ipynb')"
```

The last line runs every cell and saves the outputs into the notebook. You can also open the notebook
in Jupyter or VS Code with this environment as the kernel and run all cells.
