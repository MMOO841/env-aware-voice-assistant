# Sample scenes

Six scenes from the **training split**, chosen by hand as clear examples of the five test parts (they
are not a random sample). Development, test and replication audio is not released. Every file here
is small enough to listen to directly: 16 kHz mono FLAC, 5.3 MB in total.

| Scene | Part | Versions (what differs) | Decision points |
|---|---|---|---|
| `respond_hearing` | Respond, hearing | 1 telephone ring, 2 no sound, 3 pots rattling in the kitchen; then the same "Hello?" | 3 (one per version) |
| `respond_reaction` | Respond, reaction | after the same ringtone: 1 the user answers the call, 2 the user rejects it and asks the assistant | 2 |
| `stop_reaction` | Stop, reaction | the assistant talks, a phone rings: 1 the user answers, 2 "keep telling me", 3 "Uh-huh.", 4 silence | 4 |
| `stop_movement` | Stop, movement | the assistant talks, nobody speaks to it: 1 doorbell, footsteps, front door, visitor; 2 doorbell only; 3 the same footsteps, then a drawer; 4 TV speech | 4 |
| `episode_phone_call` | Episodes | one 57 s scene: a call during an answer, a question to the assistant in the middle of the call, a return | 8 |
| `episode_visitor` | Episodes | one 59 s scene: a knock during an answer, the user goes to the door without a word, talks with a visitor, returns | 6 |

Versions of one scene are minimal pairs: they share the room, the voices and the background noise,
and differ only in what the contrast is about: the sound ("respond, hearing" and "stop, movement",
where the words are identical) or the user's reaction to the same sound ("respond, reaction" and
"stop, reaction"). Each scene's `description` field says what the right decisions are and why.

## What you hear

- **`v<version>_p<k>.flac`** is the 8 s window that ends at decision point `k` of that version: exactly
  the audio the models receive at that point (the samples are identical to the rendered training
  windows; FLAC is lossless).
- **`full.flac`** (episodes only) is the whole scene from its start. Each episode window is the slice
  `[t - 8 s, t]` of it. For the single-line parts only the windows are stored.
- **The assistant's own voice is not in the audio**, as on a device that cancels its own playback. What
  the assistant says, and when, is in the JSON (`assistant`).
- The caller in a phone call is never heard (as in a real room). The visitor at the door is.

## Files

```
data/samples/
  manifest.json            index of the scenes
  README.md                this file
  CREDITS.md               every recording used, its author, link, licence and the changes made
  <scene_id>/scene.json    everything about one scene
  <scene_id>/v<version>_p<k>.flac
  <scene_id>/full.flac     episodes only
```

All paths inside `manifest.json` and `scene.json` are relative to `data/samples/`. Times are in
seconds from the start of the scene (the start of `full.flac` for episodes). A window covers
`[window_start, t]` on that clock, so a sound at scene time `x` is at `x - window_start` in the
window file. The JSON files are ASCII (non-ASCII characters are escaped).

### manifest.json

| Field | Meaning |
|---|---|
| `format_version` | 1 |
| `split` | `"train"` |
| `sample_rate`, `window_seconds` | 16000, 8.0 |
| `paths_relative_to` | `"data/samples/"` |
| `scenes[]` | `scene_id`, `part`, `part_title`, `title`, `scene_json` (path), `versions` (list of version ids), `full_audio` (paths, episodes only), `n_points` |

`part` is one of `respond_hearing`, `respond_reaction`, `stop_reaction`, `stop_movement`, `episode`.

### scene.json

| Field | Meaning |
|---|---|
| `scene_id`, `split`, `part`, `part_title`, `title`, `description` | as above; `description` explains the scene and its right answers |
| `source_unit` | id of the scene in the full training render |
| `room` | simulated room of the impulse responses (OpenSLR 28), e.g. `small-Room141` |
| `noise` | background noise: `dataset` (DEMAND), `environment`, `channel`, `offset_s` (start in the recording), `snr_db` (relative to the speech reference level) |
| `voices` | text-to-speech voices (Kokoro) of the user and, where present, the visitor |
| `assistant_voice_in_audio` | always `false` |
| `licence` | licence of the audio (see below) |
| `versions[]` | one entry per version, below |

Each **version**:

| Field | Meaning |
|---|---|
| `version`, `label` | version id (`"1"`, `"2"`, ...) and what is special about it |
| `audio` | `full.flac` path (episodes) or `null` |
| `duration_s` | length of the whole version |
| `mix_gain_db` | gain applied to the whole mix to avoid clipping (0.0 = none) |
| `speech[]` | scripted speech in the room: `id`, `who` (`user`, `visitor`, or `tv` for a synthetic TV voice; none in these samples), `text`, `start`, `end`, `position`, `level_db`, `voice` |
| `assistant[]` | the assistant's turns (not audible): `text`, `start`, `end` (when it stopped), `full_end` (when the turn would have ended), `cut_off` |
| `sounds[]` | recorded sounds: `id`, `class`, `clip_id`, `start`, `end`, `position`, `level_db`, and `role` (`door_open` / `door_close`) for doors in episodes |
| `points[]` | decision points, below |

`class` is one of `phone`, `doorbell`, `knock`, `footsteps`, `door`, `cabinet` (cupboard or drawer),
`decoy` (an unrelated household sound) or `tv_recording`. `clip_id` is the Freesound sound id of an
FSD50K clip, or `voxpopuli:<audio id>` for the TV recording; both are listed in `CREDITS.md`.
`position` is where the source is in the room: `near` (close to the device), `across` (across the
room), `door` (at the room's door), or `walk0`, `walk1`, ... (successive footsteps moving away from or
back towards the device). `level_db` is relative to the speech reference level (speech active RMS 0.05,
about -26 dBFS). The assistant's turns are as spoken when every earlier decision was right: where the
right answer is "stop", its turn ends at that decision point.

Each **decision point**:

| Field | Meaning |
|---|---|
| `id` | `"<scene_id>\|<version>\|<k>"` |
| `version`, `k` | version id and point index within the version |
| `t` | decision time (0.2 s after the line just said ends, or a probe time when nobody spoke) |
| `window`, `window_start` | path of the 8 s window and its start time (`t - 8`) |
| `assistant_speaking` | whether the assistant is talking just before `t` (the decision model uses it to choose its "assistant speaking" / "user speaking" marker) |
| `asks` | the questions the systems answer here: `"stop"` while the assistant talks, `"respond"` when a line just ended |
| `gold` | `{"stop": "stop" \| "continue" \| null, "respond": "respond" \| "not" \| null}`; `null` means not scored here |
| `line`, `line_text` | id (in `speech`) and scripted words of the line just said, or `null` when nobody spoke |
| `category` | `r_hear`, `r_react`, `s_react`, `s_move` for single-line scenes; in episodes `pickup` (first reaction to the call or visitor), `away` (a line to the caller or visitor), `mid` (a question to the assistant during the call), `return` (back to the assistant), `stop_or_not` (a probe without a line, or a backchannel or keep-going line while the assistant talks), `control` (a line in an episode without a call or visitor, or before the user turns away) |
| `front_end` | the stored outputs of the perception front end at this point (below) |
| `situation_text`, `questions` | the text summary every text-reading system receives here, and the question lines that go with it |

`respond` is asked whenever a line just ended but is scored only where it is defined: after a line
spoken while the assistant was talking, only when the right answer is "stop" (so it is `null` for
"keep telling me" and "Uh-huh.").

**Front end** (`front_end`), computed on the audio before `t` only:

| Field | Meaning |
|---|---|
| `transcripts` | Whisper-small transcript of every speech segment that started before `t`, keyed by its `id` in `speech`; a segment still going at `t` is cut there and keyed `"<id>@<k>"` |
| `speaker_score` | cosine between the segment's ECAPA-TDNN speaker embedding and the user's enrolment embedding |
| `speaker_tag` | `"user"` when the score is at least 0.319 (threshold fixed on the development set), else `"other"` |
| `detected_sounds` | runs of the sound-event detector (fine-tuned PANNs CNN14) from the scene start to `t`: `class`, `start`, `end`, `peak` probability; runs with a peak of at least 0.2, runs closer than 0.3 s merged |

These are the outputs stored when the training set was processed, errors included: for example, the
speaker model tags some short user lines ("Uh-huh.", "Okay.") as `other`, and the detector reports a
weak `cabinet` and `decoy` beside the door in `episode_visitor`. The situation text is built from them
by `src/p1v4_situation.py` (variant "full").

### Reading the samples

```python
import json
import soundfile as sf

root = "data/samples/"
manifest = json.load(open(root + "manifest.json", encoding="utf-8"))
for entry in manifest["scenes"]:
    scene = json.load(open(root + entry["scene_json"], encoding="utf-8"))
    for version in scene["versions"]:
        for p in version["points"]:
            audio, sr = sf.read(root + p["window"], dtype="float32")   # 128000 samples, sr = 16000
            print(p["id"], p["asks"], p["gold"], p["situation_text"].splitlines()[-1])
```

## Licence

Every sample contains background noise from the DEMAND database, which is shared under CC BY-SA 3.0,
so **the audio files in this folder are shared under CC BY-SA 3.0**
(https://creativecommons.org/licenses/by-sa/3.0/). All other sources in them are CC0 (every event
recording and the TV speech) or Apache-2.0 (the text-to-speech model and the room impulse responses).
Authors, links and changes are listed in [CREDITS.md](CREDITS.md).
