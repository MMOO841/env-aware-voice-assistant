# Trained models and settings

Everything needed to run the trained system (`src/demo.py`, see the Quick Start in
[SETUP.md](../SETUP.md)). The base models it builds on are downloaded separately (below).

## Files

| File | Size | What it is | Made by | sha256 |
|---|---|---|---|---|
| `l1_new.pt` | 40.5 MB | Our LoRA adapter on VoxTurn's language model plus a new option head. The head is the **original read-out** (it turns the model's last-layer states into P(stop) and P(respond)). | `src/p1v4_l1.py train` | `a8a68fcb35e64c81f61eb667d4b16c8dca18643f3a6ddc57af895e50733ed64b` |
| `readouts_2d.npz` | 0.4 MB | **Ours**: the step 2d read-out, one logistic regression per question on the same model's last-layer states and its audio encoder's per-second outputs | `src/p1v4_readout_2d.py fit` | `87ad40a33609529739fb91fbb9f3fc689840c16a0d18f4a789bf8cdf33559ebb` |
| `settings.json` | 4 KB | Frozen decision thresholds, speaker-ID threshold, front-end settings and the hashes of the weight files | the development-set choices listed below | `a6079793ac0f6d66789e20ffbf7885238f90a555b2bf487b0e566a27b56d9dff` (LF line endings) |
| `speaker_enrolment.json` | 20 KB | Speaker-ID enrolment embeddings of the five user voices in `data/samples/` | `src/p1v4_feats.py run` (enrolment step) | `39f1d8b9d621c6e80f9c2942ac3b3d75f3909aefe9169fa536cdea8187b0325a` (LF line endings) |
| `release/sed.pt` | 323 MB | The fine-tuned sound-event detector. It is **not in the repository**: download it from the GitHub Release ([release/README.md](release/README.md)) | `src/p1v4_sed.py train` | `58be2ddacc795e5801bd252ee7c46ba87733c89a8709d759701a5a01ec49d2d3` |

`src/demo.py` checks the three weight files against these hashes before it loads them.

### `l1_new.pt`: adapter and original read-out

- A PyTorch `state_dict` of 394 float32 tensors (10.1 M parameters). 392 of them are the LoRA matrices
  of the adapter named `p1v4`: rank 16, alpha 32, no dropout, on every linear layer of the 28
  language-model layers. The other two, `new_head.weight` and `new_head.bias`, form a 1,024 to 1
  linear layer, started from VoxTurn's own option head.
- VoxTurn's own weights stay frozen and are not in this file. VoxTurn's LoRA stays active under ours.
- Input: the last 8 s of audio, plus the situation text (its most recent 1,024 tokens) built by the
  front end.
- Training (`python -X utf8 src/p1v4_l1.py train data/p1v4/runs/l1`):
  - 2 epochs over the training split;
  - micro-batch 1 with 8 accumulation steps;
  - AdamW at learning rate 1e-4 with no weight decay, 300 warm-up steps, then linear decay;
  - gradient clipping at 1.0 and bf16 autocast;
  - one of four phrasings of each question at random per sample.
- The epoch kept is the second, chosen by the development-set operating point.
- Thresholds of the original read-out: P(stop) >= 0.286, P(respond) >= 0.049.

### `readouts_2d.npz`: ours

- For each question, `S` (stop) and `R` (respond), the file holds five arrays, keyed `ears8|<S or
  R>|<name>`:
  - `mu`, `sd`: the training mean and spread used to standardise the 18,432 features;
  - `w`, `b`: the logistic-regression weights and bias;
  - `wd`: the weight decay chosen on a 20% holdout of training scenes (0.01 for stop, 0.001 for
    respond).
- The 18,432 features come from the fine-tuned VoxTurn above:
  - its last-layer states at the question's two answer options (2,048);
  - the mean and the max of its audio encoder's 13 tokens in each of the 8 seconds of the window
    (16,384).
- Training data: the training split after one forward pass of the fine-tuned model in which the
  detector's labels were made independent of the sound. For every detector class, a random half of
  the decision points either lost the labels of a sound that was really there, or gained a false label
  copied, timing included, from a real one. The confidences of the kept labels were redrawn at random.
  (`src/p1v4_readout_2c.py edits` and `extract`, then `src/p1v4_readout_2d.py ears` and `fit`.)
- Thresholds: P(stop) >= 0.139, P(respond) >= 0.248, chosen on the development set by
  `src/p1v4_score.py operating_point` (`src/p1v4_readout_2d.py eval`).

### `settings.json` and `speaker_enrolment.json`

- **Decision thresholds** of both read-outs, frozen before the test set was opened.
- **Speaker ID**:
  - Each speech segment's ECAPA-TDNN embedding is compared with the user's enrolment embedding.
    Cosine >= 0.319 means "user", otherwise "not the user".
  - The threshold maximises balanced accuracy over the 4,235 development-set speech segments
    (`src/p1v4_context.py threshold`), where balanced accuracy is 0.937.
  - Enrolment: the user's voice reads the first two sentences of the Rainbow Passage once (clean
    text-to-speech, 8 to 14 s). The embedding of that recording is the reference.
  - `speaker_enrolment.json` holds these embeddings for the sample voices, as computed when the
    training set was processed. Running ECAPA again on the same enrolment recordings reproduces them
    exactly.
- **Transcripts**: Whisper small in float16, greedy decoding in English, at most 160 new tokens.
  - Every speech segment that started before the decision point is cut at its scripted times.
  - A finished segment runs to 0.2 s after its end, at most to the decision point. A segment still
    going is cut at the decision point.
  - Each cut is padded to 30 s.
- **Detector**: run on the audio from the scene start to the decision point (16 kHz upsampled to
  32 kHz, 100 frames per second).
  - Per class, runs of frames with probability >= 0.2 are kept; runs closer than 0.3 s are merged.
  - Each run is reported as (class, start, end, peak probability).

## Base models (downloaded, not in this repository)

The commands are in [SETUP.md](../SETUP.md). By default, `src/demo.py` looks for the downloaded folders
under `data/models_dl/` (option `--models-dl`).

| Model | Source and revision | Licence | Where the code finds it |
|---|---|---|---|
| VoxTurn v1 (Yang Xiao) | Hugging Face `AustinXiao/voxturn`, revision `fe3d4a319c4e74279544188bd0e8292325852dbc` | CC BY-NC 4.0 | `data/models_dl/voxturn_fe3d4a3/` |
| Qwen3-ASR-0.6B (VoxTurn's backbone) | Hugging Face `Qwen/Qwen3-ASR-0.6B-hf`, revision `7f1569a48a89f3e3f4dc3a5c9d28bddd903bc76c` | Apache-2.0 | Hugging Face cache (VoxTurn's code downloads it on first use) |
| Whisper small | Hugging Face `openai/whisper-small`, revision `973afd24965f72e36ca33b3055d56a652f456b4d` | MIT | Hugging Face cache (downloaded on first use) |
| ECAPA-TDNN | Hugging Face `speechbrain/spkrec-ecapa-voxceleb`, revision `0f99f2d0ebe89ac095bcc5903c4dd8f72b367286` | Apache-2.0 | `data/models_dl/spkrec-ecapa-voxceleb_0f99f2d/` (SpeechBrain copies it to `data/models_dl/spkrec-ecapa-voxceleb/` when it loads) |
| PANNs CNN14 code | GitHub `qiuqiangkong/audioset_tagging_cnn`, commit `d2f4b8c18eab44737fcc0de1248ae21eb43f6aa4`, files `pytorch/models.py` and `pytorch/pytorch_utils.py` | MIT | `data/models_dl/panns_d2f4b8c/` |
| PANNs CNN14 AudioSet checkpoint | Zenodo record 3987831, `Cnn14_DecisionLevelAtt_mAP=0.425.pth` (331,757,477 bytes, md5 `c8281ca2b9967244b91d557aa941e8ca`) | CC BY 4.0 | `data/models_dl/panns_d2f4b8c/`. Only needed to train the detector again, since `sed.pt` holds every weight. |

## Licence of the trained files

`l1_new.pt` and `readouts_2d.npz` are built on VoxTurn, so they are shared under the same
non-commercial terms (CC BY-NC 4.0). `sed.pt` is built on the PANNs AudioSet checkpoint (CC BY 4.0,
Kong et al., 2020). Details are in [ATTRIBUTION.md](../ATTRIBUTION.md).

## Using these files in the full pipeline

The scripts in `src/` read the weights from their training output folders. To skip a training step,
copy the file there:

| File | Copy to |
|---|---|
| `l1_new.pt` | `data/p1v4/runs/l1/l1_new.pt` |
| `readouts_2d.npz` | `data/p1v4/runs/grounding/allclass2d/readouts_2d.npz` |
| `release/sed.pt` | `data/p1v4/runs/sed/sed.pt` |

The pipeline itself reads the thresholds from its freeze record (`results/main_v04/freeze.json`,
written by `src/p1v4_freeze.py`). `settings.json` holds the same values.

## Second backbone (not included)

The files trained for the second backbone, Voxtral Mini 3B
([docs/evaluation_protocol.md](../docs/evaluation_protocol.md), section 17), are not in this repository.
Its freeze record lists:
- the checkpoint `data/p1v4/runs_bb2/l1/l1_new.pt` (the LoRA adapter on Voxtral's language model and the
  new answer head, 27.0 M parameters): 108 MB, more than GitHub's 100 MB limit for a single file;
- the fitted read-outs, about 20 MB in all, of which ours (step 2d, `grounding/allclass2d/readouts_2d.npz`)
  is 1.3 MB;
- Voxtral's own weights (9.4 GB), which are downloaded, not trained.

To train them again, see [SETUP.md](../SETUP.md), section 8.
