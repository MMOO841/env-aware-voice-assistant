# Credits for the sample scenes

Every sound in `data/samples/` is listed here: the recorded sounds, the TV speech, the background noise,
the room impulse responses and the text-to-speech voices. **The audio files in `data/samples/` are
shared under CC BY-SA 3.0** (https://creativecommons.org/licenses/by-sa/3.0/), because every scene
contains DEMAND background noise, which is CC BY-SA 3.0. All the other sources are CC0 or Apache-2.0.
No recording under a non-commercial licence is used.

All scenes are synthetic: the sources below were processed and mixed by `src/p1v4_units.py` and
`src/p1_render.py`. Times and levels per scene are in each `scene.json`.

## Recorded sounds (FSD50K, from Freesound)

All twelve recordings come from FSD50K (Eduardo Fonseca, Xavier Favory, Jordi Pons, Frederic Font and
Xavier Serra, "FSD50K: An Open Dataset of Human-Labeled Sound Events", IEEE/ACM Transactions on Audio,
Speech, and Language Processing 30, 2022; https://zenodo.org/records/4060432). FSD50K as a whole is
licensed CC BY 4.0; each clip keeps the licence its author chose on Freesound. All twelve clips used here
are **CC0 1.0** (public domain dedication, https://creativecommons.org/publicdomain/zero/1.0/): no
attribution is required, and they are credited anyway. The title and author are the clip's Freesound
title and uploader as recorded in the FSD50K metadata.

**Changes made to every recording:** decoded, mixed down to mono and resampled to 16 kHz; trimmed to its
active part (the span of 20 ms frames within 35 dB of the loudest frame); cut, or looped, to the length
given below, with 20 ms fades at both ends (footsteps: 10 ms fades on each step); convolved with a
simulated room impulse response (see "Room impulse responses"); scaled to the level given below,
relative to the speech reference level; mixed with the speech and the background noise; and cut into the
8 s windows. The recordings themselves are not distributed here.

### 79440: "telephone ring.wav"

- Author: xyzr_kx (Freesound)
- Source: https://freesound.org/people/xyzr_kx/sounds/79440/ (FSD50K development set)
- Licence: CC0 1.0
- Used in: `respond_hearing`, version 1 (phone ring). Changes: active part 16.4 s, cut to 2.5 s; level +2.0 dB.

### 181261: "Small Metal Pots Rattling.wav"

- Author: CapsLok (Freesound)
- Source: https://freesound.org/people/CapsLok/sounds/181261/ (FSD50K evaluation set)
- Licence: CC0 1.0
- Used in: `respond_hearing`, version 3 (unrelated kitchen sound). Changes: active part 11.5 s, cut to 2.5 s; level +2.0 dB.

### 117565: "20-RA2.mp3"

- Author: mojomills (Freesound)
- Source: https://freesound.org/people/mojomills/sounds/117565/ (FSD50K development set)
- Licence: CC0 1.0
- Used in: `respond_reaction`, versions 1 and 2 (phone ring). Changes: active part 8.2 s, cut to 2.9 s; level -6.3 dB.

### 117566: "21-RA2.mp3"

- Author: mojomills (Freesound)
- Source: https://freesound.org/people/mojomills/sounds/117566/ (FSD50K development set)
- Licence: CC0 1.0
- Used in: `stop_reaction`, versions 1, 2, 3 and 4 (phone ring). Changes: active part 8.2 s, cut to 3.3 s in version 1; looped to 19.2 s in versions 2, 3 and 4; level -3.0 dB.

### 415164: "doorbell 80's-1.wav"

- Author: zooanimaluprising (Freesound)
- Source: https://freesound.org/people/zooanimaluprising/sounds/415164/ (FSD50K development set)
- Licence: CC0 1.0
- Used in: `stop_movement`, versions 1 and 2 (doorbell). Changes: active part 3.4 s, cut to 1.2 s; level -3.7 dB.

### 378398: "Walk.wav"

- Author: 13FPanska_Stranska_Michaela (Freesound)
- Source: https://freesound.org/people/13FPanska_Stranska_Michaela/sounds/378398/ (FSD50K development set)
- Licence: CC0 1.0
- Used in: `stop_movement`, versions 1 and 3 (footsteps). Changes: cut into 3 steps of 0.8 to 1.0 s (consecutive pieces of the first 2.5 s of its active part), each reverberated with a different impulse response along a path away from the device; level -6.0 dB.

### 119077: "Door opening; unlocking_1-2.aif"

- Author: domrodrig (Freesound)
- Source: https://freesound.org/people/domrodrig/sounds/119077/ (FSD50K evaluation set)
- Licence: CC0 1.0
- Used in: `stop_movement`, version 1 (door). Changes: active part 2.8 s, cut to 1.0 s; level -5.2 dB.

### 427301: "DRAWER CLOSE 2.wav"

- Author: tmrobin1 (Freesound)
- Source: https://freesound.org/people/tmrobin1/sounds/427301/ (FSD50K development set)
- Licence: CC0 1.0
- Used in: `stop_movement`, version 3 (drawer). Changes: active part 2.8 s, cut to 1.0 s; level -5.2 dB.

### 117570: "25-RA2.mp3"

- Author: mojomills (Freesound)
- Source: https://freesound.org/people/mojomills/sounds/117570/ (FSD50K development set)
- Licence: CC0 1.0
- Used in: `episode_phone_call`, version 1 (phone ring). Changes: active part 6.8 s, cut to 2.3 s; level +3.9 dB.

### 336391: "door knocking pounding.wav"

- Author: ghaelen (Freesound)
- Source: https://freesound.org/people/ghaelen/sounds/336391/ (FSD50K development set)
- Licence: CC0 1.0
- Used in: `episode_visitor`, version 1 (knock). Changes: active part 2.3 s, cut to 1.0 s; level -6.7 dB.

### 388504: "Footsteps.wav"

- Author: singingmonk02 (Freesound)
- Source: https://freesound.org/people/singingmonk02/sounds/388504/ (FSD50K development set)
- Licence: CC0 1.0
- Used in: `episode_visitor`, version 1 (footsteps). Changes: cut into 3 steps of 0.8 to 0.9 s (consecutive pieces of the first 2.7 s of its active part), each reverberated with a different impulse response along a path away from the device; and 4 steps of 0.8 to 0.9 s (consecutive pieces of the first 3.3 s of its active part), each reverberated with a different impulse response along a path back towards the device; level -8.8 dB.

### 71212: "18 Close Door.wav"

- Author: adegenerate (Freesound)
- Source: https://freesound.org/people/adegenerate/sounds/71212/ (FSD50K evaluation set)
- Licence: CC0 1.0
- Used in: `episode_visitor`, version 1 (the door opening). Changes: active part 2.4 s, cut to 0.9 s; level -0.4 dB.
- Used in: `episode_visitor`, version 1 (the door closing). Changes: active part 2.4 s, cut to 0.6 s; level -0.4 dB.

## TV speech (VoxPopuli)

- Recording: VoxPopuli English test set, audio id `20131022-0900-PLENARY-5-en_20131022-10:34:20_2` (speaker 96959), a European
  Parliament plenary speech. Transcript: "the harsh reality of course is that we are living from day to day in budgetary terms. following the commitments agreed in the mff back in the boom times of two thousand and seven it is now proving very difficult for member states laden with debt to honour the bills as they come in today."
- Source: VoxPopuli (Changhan Wang et al., "VoxPopuli: A Large-Scale Multilingual Speech Corpus for
  Representation Learning, Semi-Supervised Learning and Interpretation", ACL 2021),
  https://huggingface.co/datasets/facebook/voxpopuli (revision 42f0187)
- Licence: CC0 1.0
- Used in: `stop_movement`, version 4 (a TV playing in the room). Changes: 16 kHz mono; the first 8.5 s; band-pass filtered to 150 Hz to 7 kHz (a loudspeaker-like sound); reverberated (across the room); level -11.2 dB; mixed.

## Background noise (DEMAND): CC BY-SA 3.0

DEMAND (Joachim Thiemann, Nobutaka Ito and Emmanuel Vincent, "The Diverse Environments Multi-channel
Acoustic Noise Database: A database of multichannel environmental noise recordings", Proceedings of
Meetings on Acoustics 19, 2013), https://zenodo.org/records/1227121. Licence: CC BY-SA 3.0
(https://creativecommons.org/licenses/by-sa/3.0/), as stated in the dataset's documentation (the Zenodo
record lists CC BY 4.0); the stricter terms are applied here. Because of this noise, every sample audio
file is shared under CC BY-SA 3.0.

**Changes made:** channel 1 of the 16 kHz recording of the environment; a segment as long as the scene,
starting at the offset below; scaled so that the speech reference level is the given SNR above the
noise; mixed under all speech and sounds; cut into the 8 s windows.

| Environment (DEMAND) | Scene | Offset in the recording | SNR |
|---|---|---|---|
| OHALLWAY | `respond_hearing` | 208.4 s | 26.2 dB |
| DWASHING | `respond_reaction` | 81.8 s | 25.6 dB |
| TBUS | `stop_reaction` | 132.3 s | 15.2 dB |
| DWASHING | `stop_movement` | 137.4 s | 21.4 dB |
| OMEETING | `episode_phone_call` | 158.8 s | 18.8 dB |
| OMEETING | `episode_visitor` | 90.9 s | 12.2 dB |

## Room impulse responses (OpenSLR 28): Apache-2.0

Simulated room impulse responses from the RIRS_NOISES set (Tom Ko, Vijayaditya Peddinti, Daniel Povey,
Michael L. Seltzer and Sanjeev Khudanpur, "A Study on Data Augmentation of Reverberant Speech for Robust
Speech Recognition", ICASSP 2017), https://www.openslr.org/28/, licence Apache-2.0
(https://www.apache.org/licenses/LICENSE-2.0). **Changes made:** each response was normalised to a peak
of 1 and convolved with the speech and sounds; the responses themselves are not distributed here.

| Scene | Simulated room | Impulse responses (paths inside RIRS_NOISES) |
|---|---|---|
| `respond_hearing` | medium-Room003 | `simulated_rirs/mediumroom/Room003/Room003-00078.wav`, `simulated_rirs/mediumroom/Room003/Room003-00081.wav` |
| `respond_reaction` | small-Room041 | `simulated_rirs/smallroom/Room041/Room041-00007.wav`, `simulated_rirs/smallroom/Room041/Room041-00062.wav` |
| `stop_reaction` | medium-Room145 | `simulated_rirs/mediumroom/Room145/Room145-00036.wav`, `simulated_rirs/mediumroom/Room145/Room145-00070.wav` |
| `stop_movement` | small-Room141 | `simulated_rirs/smallroom/Room141/Room141-00007.wav`, `simulated_rirs/smallroom/Room141/Room141-00042.wav`, `simulated_rirs/smallroom/Room141/Room141-00055.wav`, `simulated_rirs/smallroom/Room141/Room141-00056.wav`, `simulated_rirs/smallroom/Room141/Room141-00066.wav`, `simulated_rirs/smallroom/Room141/Room141-00073.wav`, `simulated_rirs/smallroom/Room141/Room141-00090.wav` |
| `episode_phone_call` | small-Room120 | `simulated_rirs/smallroom/Room120/Room120-00005.wav`, `simulated_rirs/smallroom/Room120/Room120-00073.wav` |
| `episode_visitor` | small-Room098 | `simulated_rirs/smallroom/Room098/Room098-00009.wav`, `simulated_rirs/smallroom/Room098/Room098-00034.wav`, `simulated_rirs/smallroom/Room098/Room098-00043.wav`, `simulated_rirs/smallroom/Room098/Room098-00056.wav`, `simulated_rirs/smallroom/Room098/Room098-00060.wav`, `simulated_rirs/smallroom/Room098/Room098-00077.wav`, `simulated_rirs/smallroom/Room098/Room098-00090.wav`, `simulated_rirs/smallroom/Room098/Room098-00093.wav` |

## Speech (Kokoro text-to-speech): Apache-2.0

Every voice in the samples except the TV recording (the users and the visitors) is synthetic, made
with Kokoro-82M v1.0 (hexgrad), https://huggingface.co/hexgrad/Kokoro-82M, licence Apache-2.0.
**Changes made:** synthesised
at 24 kHz and resampled to 16 kHz; leading and trailing silence trimmed; reverberated; scaled to the
speech reference level (0 dB near the device, about -4 dB across the room, about -8 dB at the door);
mixed. The words are dialogue lines written for this project by a language model from fixed prompts and
checked by a second model (see `ATTRIBUTION.md`).

| Voice | Role | Scenes |
|---|---|---|
| af_alloy | user | `stop_reaction` |
| af_bella | visitor | `stop_movement` |
| af_river | user | `episode_phone_call` |
| af_sarah | user | `stop_movement`, `episode_visitor` |
| am_liam | user | `respond_reaction` |
| am_onyx | user | `respond_hearing` |
| am_santa | visitor | `episode_visitor` |

## Not in the audio

The assistant's own voice is not in any sample. Its speaking times were measured with the Piper
text-to-speech voice en_US-libritts-high (speaker 0; piper-tts, GPL-3.0-or-later; voice model CC BY 4.0),
but that audio is not mixed into any of these samples.
