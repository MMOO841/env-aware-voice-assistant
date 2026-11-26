# Attribution

## Roles and AI use

**Roles.** The author defined the task and its scope, made the key design choices listed below, approved
every experiment before it was run, and checked the results. Claude Code (Anthropic) implemented these
decisions: it wrote the code, ran the experiments, proposed options for the author to choose from, and
drafted the documentation and an outline of the video narration, which the author revised.

**Key decisions by the author.** Code comments record them with dates ("approved by the author",
"chosen by the author").

- Chose the problem: voice assistants that ignore household sounds such as phone rings, doorbells and
  knocks.
- Framed the task as two decisions that depend on the assistant's state: whether to stop while it is
  talking, and whether to respond when the user speaks. The first version only decided whether to
  respond after a sound.
- Pointed out that real users often reject a call and keep talking to the assistant, and that a talking
  assistant should not stop at every sound. This led to the test parts and their minimal-pair versions.
- Chose to measure front-end errors by injecting false detector labels into fixed test scenes, rather
  than by redesigning the test set.
- Insisted on a model that really hears the environment, instead of one that leans on the pretrained
  model and the detector's text. This led to the probe, the matched grounding test and the read-out
  iterations (steps 2 to 2d).
- Asked why the stop decision was less grounded than the respond decision, chose to fix it rather than
  only report it (step 2c), and chose the per-second read-out (step 2d) among the options.
- Listened to the audio of one noise recording on which the detector raised many false phone labels,
  confirmed that it contains faint singing and no ring, and so kept the gold labels.
- Approved extra fairness measures for the Gemini rivals (a note that detector labels can be wrong,
  plus the detector's measured precision), although they made the rivals stronger.
- After the masking-noise result, chose to try an explicit doubt signal read from the model's own audio
  encoder.
- Decided to repeat the whole recipe on a second backbone from another model family (Voxtral Mini 3B) to
  test whether the method depends on VoxTurn.

**What was generated.** Claude Code (model Claude Opus 5.5), working in the author's local repository
and terminal, wrote all of the code in this repository. Every file in `src/` carries a file-level note
saying so. Claude also drafted the documentation (README.md, SETUP.md, this file and `docs/`) and an
outline of the video narration and slides; the author edited them and recorded the videos.

**Independent AI reviews.** Before the test set was opened, the plan, the code and the analysis were
reviewed by fresh Claude subagents (Claude Fable 5.2 or Claude Opus 5.5) that had only the documents and
the code, not Claude's own conclusions. Their findings are cited in code comments as "review M5", "review
C-minor 9" and similar. Each finding was checked against the code and data before it was fixed.

**What had to be debugged or reworked.** AI-generated code was often wrong in ways that only showed up
in checks or reviews. Examples:

- *Confidence intervals.* The first cluster bootstrap multiplied one random weight per conversation and
  one per recording group. This inflates the variance (a simulation gave 2.8 times the true standard
  error). It was replaced by a combination of one-way cluster bootstraps (1.1 times), calibrated by
  simulation before use.
- *Label leak.* The respond question was first asked only when the gold stop answer was "stop", so the
  question itself revealed the answer. Both questions are now asked at every decision point.
- *Test-set lock.* One loader could open the sealed test split through a flag meant for the final run.
  The lock was fixed and a lock check added before the test.
- *Wrong model input.* The off-the-shelf VoxTurn reference first received the whole situation text
  instead of the line transcript (its documented input), which made two different references give
  identical decisions at all 3,692 points. Caught by a sanity check and fixed.
- *GPU memory on Windows.* Fine-tuning at micro-batch 4 needed 16 GB on the 8 GB card. Windows silently
  moved the excess into system memory (26 s per step), and one stalled check cost 6.5 hours. The fix was
  micro-batch 1 with 8 accumulation steps and a hard cap on GPU memory.
- *A false out-of-memory error.* The local LLM rival built a 1.6 GB attention table when encoding its
  48-example prompt in one pass. The prompt is now encoded in pieces, and every run compares the first
  scores with an uncached pass.
- *Scene generation.* Per-recording caps were trimmed after scenes were built, and the trim did not
  converge on one split. The trim was rewritten to use the same measure as the check.
- *API change.* Gemini stopped returning token probabilities in the middle of the project. A fallback
  that uses the answers was written down before it was used, with a correction for what the rivals lose.
- *Random starts.* A control model's result depended on its random initialisation. It is now reported
  as the mean over 8 fixed starts.
- *Wrong results folder.* The first development report for the second backbone contained two rows that
  were in fact backbone 1's, because one report function did not follow the redirected results folder.
  Caught by comparing with backbone 1's report; fixed and re-run before the freeze.

## Models

| Model | Used for | Source and licence |
|---|---|---|
| VoxTurn (Yang Xiao, 2026) | Backbone of our decision model; fine-tuned with LoRA | huggingface.co/AustinXiao/voxturn, revision fe3d4a3; weights and inference code CC BY-NC 4.0. Our adapter and read-outs are built on it and are shared under the same non-commercial terms. |
| Qwen3-ASR-0.6B | VoxTurn's own backbone | Qwen, Apache-2.0 |
| Voxtral Mini 3B (Mistral AI; Liu et al., 2025) | Second backbone: the same recipe (LoRA, answer head, read-out) repeated to test whether the method depends on VoxTurn | huggingface.co/mistralai/Voxtral-Mini-3B-2507, revision 3060fe3; Apache-2.0. Its fine-tuned weights are not released in this repository. |
| PANNs CNN14, decision-level attention (Kong et al., 2020) | Sound-event detector, fully fine-tuned on our classes; the fine-tuned checkpoint `sed.pt` (GitHub Release asset) is derived from the AudioSet checkpoint | github.com/qiuqiangkong/audioset_tagging_cnn (MIT), AudioSet checkpoint on Zenodo (CC BY 4.0) |
| Whisper small, base and large-v3-turbo (OpenAI) | Transcripts (small for local systems, large-v3-turbo for the Gemini readers); the base encoder is fine-tuned in one rival | MIT |
| ECAPA-TDNN (speechbrain/spkrec-ecapa-voxceleb, revision 0f99f2d) | Speaker tags (user / not the user) | Apache-2.0 |
| all-MiniLM-L6-v2 (sentence-transformers) | Text embeddings in the late-fusion and label-reader models | Apache-2.0 |
| Qwen3.5-4B (revision 851bf6e), 4-bit NF4 | Local LLM rival | Apache-2.0 |
| Gemini (gemini-4-hydrogen, gemini-4-carbon) | Rival systems via the Gemini API (text readers, audio listener, perfect-label reference) | Google Gemini API terms |
| Kokoro-82M (hexgrad, v1.0) | Text-to-speech for users, visitors and synthetic TV speech | Apache-2.0 |
| Piper, voice en_US-libritts-high | Second text-to-speech engine: the assistant's voice (speaker 0) and user voices reserved for the held-out splits | piper-tts GPL-3.0-or-later; voice model CC BY 4.0 |
| GPT-6.1 (gpt-6.1-sol, OpenAI API) | Generated the dialogue lines and multi-turn scripts from fixed prompts | OpenAI API terms |
| Claude Opus 5.5 and Claude Sonnet 5.5 | Checked every generated line and scene against the behaviour rules (prompts in `configs/checker_prompts_v1.md`, `configs/checker_prompts_v2.md`) | Anthropic |

## Datasets and sound sources

| Source | Used for | Licence |
|---|---|---|
| FSD50K (Fonseca et al., 2022; Zenodo record 4060432) | Event sounds (phone, doorbell, knock), unrelated household sounds, footsteps, door and cabinet sounds | Only single-source clips under CC0 or CC BY 3.0 were used (per-clip licences from the FSD50K metadata); the dataset is released under CC BY 4.0 |
| Freesound (freesound.org) | Extra door sounds, and recordings uploaded after June 2023 for the test split | CC0 or CC BY only |
| Wikimedia Commons | Four recordings used only in the test split and not released: File:Telephone.ogg and File:Doorbell-classic-dingdong.ogg (public domain), File:Knock on door.wav (CC0), File:Model 500 Telephone British ring.ogg (CC BY-SA 3.0; author on the file page) | As listed |
| DEMAND (Thiemann et al., 2013) | Background noise recordings, split by environment | CC BY-SA 3.0 in the dataset text (the Zenodo record lists CC BY 4.0) |
| MS-SNSD (Reddy et al., 2019; github.com/microsoft/MS-SNSD, commit fe61c4b) | Appliance noise (air conditioner, vacuum cleaner, washer/dryer, copy machine) for a check with noise types not used in development | Freesound recordings under CC0 |
| OpenSLR 28, RIRS_NOISES (Ko et al., 2017) | Simulated room impulse responses, split by room | Apache-2.0 |
| VoxPopuli (Wang et al., 2021), English test split | TV and radio speech in the background | CC0 |
| Rainbow Passage (Fairbanks, 1960), first two sentences | Text read by each synthetic voice to enrol it for the speaker tags | Quoted text |
| MUSAN (Snyder et al., 2015) | Not used as data: our clips were compared with it, and clips that also occur in MUSAN (VoxTurn's training noise) were removed | — |

Every clip that appears in a released audio sample is credited by title, author and link in
[data/samples/CREDITS.md](data/samples/CREDITS.md). All released sample audio contains DEMAND background
noise and is therefore shared under CC BY-SA 3.0.

## Libraries

PyTorch (BSD-3-Clause) and torchaudio (BSD), Hugging Face Transformers and PEFT (Apache-2.0), Accelerate (Apache-2.0),
bitsandbytes (MIT), scikit-learn, NumPy, SciPy and pandas (BSD-3-Clause), SpeechBrain (Apache-2.0),
soundfile (BSD-3-Clause), soxr (LGPL-2.1), librosa (ISC), torchlibrosa (MIT, used by the PANNs model code),
Kokoro and misaki (Apache-2.0), espeak-ng through espeakng-loader (GPL-3.0), piper-tts
(GPL-3.0-or-later), phonemizer (GPL-3.0), spaCy (MIT) and num2words (LGPL). Exact versions are in `requirements.txt` and `requirements-data.txt`.

## Videos

The slides in both videos were built with python-pptx and PowerPoint. Their icons are from Bootstrap
Icons 1.11.3 (MIT licence, icons.getbootstrap.com). The audio clips played in the demo video are the
training-split samples in [data/samples/](data/samples/), credited in
[data/samples/CREDITS.md](data/samples/CREDITS.md).
