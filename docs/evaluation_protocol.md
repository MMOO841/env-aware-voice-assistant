# Evaluation protocol

This file explains how the systems in this repository were evaluated. The rules of sections 1 to 12 were
written down and fixed before the test split was opened (one logged bug fix followed on the test day,
section 12.7); those of sections 13 and 14 were fixed after the test, before the replication split was
opened. All choices (epochs, thresholds, prompts, rival settings, the main system) were made on the
development set ("dev"); the test split was used once; the replication split stayed sealed until a fixed
reliability check (section 13).

The rules come from three fixed documents that the code cites by section number: the evaluation plan
("plan"), the freeze amendment ("amendment"; it replaced the main system and fixed the test-day procedure
before the freeze) and the grounding measurement package ("package", final version 0.6). Where they
conflict, the amendment's section 13 and the package's section 15 take precedence, and the amendment takes
precedence over the plan. This file is a summary of all three, not a translation. Section 16 maps every
section number cited in the code to the heading below that explains it. Section 17 summarises a later plan,
written after the test and before its own single test run, that repeated the same recipe on a second
backbone.

Systems and data parts are named by what they are; their code names are listed in
[code_map.md](code_map.md). The short ids of criteria (such as G1a) and of registered claims (such as M1)
are kept in parentheses because the code and result files use them. "Points" are percentage points;
"interval" means a 95% interval (section 7). Results are in the README and [results.md](results.md).

## At a glance

- **Data:** synthetic household scenes built as minimal pairs (versions of a scene differ only in one
  sound or one reaction). Train, dev and test share no conversations, recording groups, voices, rooms or
  noise recordings; the replication split has its own conversations and recordings but reuses the test
  voices, rooms and noise. Test and replication received automatic checks only and were sealed by hash.
- **Choices on dev only:** epochs, thresholds (one operating-point rule for every system), the rivals'
  prompts, example counts and settings, and the main read-out.
- **One test run:** a freeze record hashed every file and setting the final run reads; test metrics were
  computed only after every output was complete.
- **Main score:** the mean of five part accuracies; a scene counts as right only if all its versions are
  right. Every system's operating point is chosen on dev so that each floor check (a simple control case)
  reaches 0.90 where possible.
- **Decision rule:** ours must beat every rival by at least 5 points with the 95% interval above 0, with
  conditions on the floor checks and the unseen subsets; then a "listen step" on the parts where only the
  sound decides.
  Intervals combine one-way cluster bootstraps over conversation, event recording and a second recording.
- **Grounding:** a matched test gives every version of a scene identical text and detector labels and
  injects false labels at fixed confidences; criteria fixed in advance (G1a to G7b) decide what may be said.
- **Noise:** the test's sound-only parts were re-run with louder background noise under one fixed criterion
  (G5).
- **Afterwards:** the replication split was used once for a registered reliability check (59 claims, 19
  key), and a labelled check with new noise types followed. The whole recipe was also repeated once on a
  second backbone from another model family (section 17).

## 1. Task and decision points

### 1.1 The two decisions

- **Stop:** while the assistant is talking, should it stop or continue?
- **Respond:** when the user has just finished a line, is the line for the assistant (respond) or for
  someone else (do not respond)?
- **Decision point:** 0.2 s after the end of the user's line; for points without a line, a time fixed by
  the scene (section 3). A system may use only audio and text from before the point. The audio input is the
  8 s window that ends at the point.
- **What is asked and what is scored:** where a line ends while the assistant talks, every system gives
  both answers. Respond is scored there only when the correct stop answer is "stop"; after a backchannel or
  a "keep going" line, respond has no meaning and is not scored. When the assistant is silent, only respond
  is asked; at points without a line, only stop. Which answers are scored is known only to the scorer and
  never enters any input.
- **Versions and groups:** a scene is built in 2 to 4 versions that share the event recording, the user
  voice and the room and are identical sample by sample except for the one element under test (checked to
  within 1 least significant bit). Versions are numbered 1 to 4; all versions of a scene form a group.
- **Correct history:** in multi-turn episodes, the earlier turns are given as if the assistant had acted
  correctly, so an error does not carry over to later points.

### 1.2 What every system receives

All systems get the 8 s window and the same "situation text", built by one function:
- whether the assistant is talking, what it is saying, how long it has been silent;
- every earlier speech segment (user, visitor, TV or radio, family member), cut at its scripted start and
  end (perfect segmentation, the same for every system), with its transcript, speaker tag (user / not the
  user) and times;
- the assistant's earlier turns (from the correct history);
- detected sounds: the shared detector's output before the point, every detection with peak confidence at
  least 0.2, with class, start, end and peak;
- the transcript and speaker tag of the line just said.

Transcripts come from Whisper-small for the local systems and from Whisper large-v3-turbo for the Gemini
systems (a choice in the rivals' favour). The text-only controls get reduced variants (section 5.3).

Assumptions, stated with the results: no echo of the assistant in the microphone (headphones or echo
cancellation; a faint-echo subset is reported separately); the assistant runs on a smart speaker or
computer and the phone is a separate device; in door scenes the user is the one who goes to the door;
whether the assistant resumes by itself after the user returns is not studied.

### 1.3 Correct behaviour

The gold answers follow a fixed policy; the same text is given to every LLM reader
(`configs/policy_v2_prompt.txt`).

| Situation | Assistant talking | Assistant silent, user just spoke |
|---|---|---|
| No event; the user talks to the assistant | stop (ordinary interruption) | respond |
| The user backchannels ("mm-hm", "okay") | continue | — |
| An event happens and the user ignores it (lets it ring, rejects the call, does not go to the door) | continue | later lines to the assistant: respond |
| The user answers the event ("Hello?", "Coming!") or silently goes to open the door | stop | lines to the caller or visitor: do not respond |
| The user tells the assistant about the event ("Hold on, someone's at the door.") | stop and respond (briefly) | respond (briefly) |
| The user tells the assistant to go on ("Ignore that, keep going.") | continue | — |
| The user is talking with someone else | stay stopped | do not respond |
| The user turns to the assistant in the middle of a call or visit | — | respond; back in the call: do not respond |
| The user comes back to the assistant (with or without "I'm back", with or without a goodbye) | — | respond |
| Unrelated household sound (decoy; footsteps of chores, a cupboard) | continue | respond as usual |
| Speech from a TV or radio | continue | — |

An ignored ring rings out (about 15 to 30 s); a rejected call stops at once. The assistant continues in
both cases.

## 2. Data and splits

### 2.1 Splits and their sources

All scenes are synthetic: text-to-speech dialogue mixed with real recorded sounds, room reverberation and
background noise. Train, dev and test share no conversations, no lines or phrasings except the common
ones below, no user or visitor voices, recording groups, TV speakers, rooms or noise recordings. The
replication split has its own conversations, recording groups and TV speakers, but takes its voices, rooms
and noise from the test split.

| Source | Train | Dev | Test | Replication |
|---|---|---|---|---|
| Conversations (topic, request, answer, follow-up) | 300 | 100 | 600 | 600 |
| Kokoro voices | 15 | 5 | 8 | test voices |
| Simulated rooms (RIRS_NOISES) | 240 | 60 | 100 | test rooms |
| Background-noise environments (DEMAND) | 10 | 3 | 4 | test environments |
| Phone-ring recordings / recording groups | 73 / 20 | 4 / 4 | 16 / 13 | 7 / 6 |
| Doorbell recordings / groups | 35 / 16 | 10 / 4 | 19 / 11 | 6 / 5 |
| Knock recordings / groups | 66 / 45 | 9 / 8 | 55 / 25 | 49 / 13 |
| Decoy recordings / groups | 1,536 / 650 | 380 / 158 | 60 / 30 | 22 / 12 |

- Event and decoy sounds come from FSD50K (single-source clips under CC0 or CC BY only), sorted into pools by
  fixed rules on their titles, tags and descriptions before any audio was used. Recordings were grouped by
  uploader and by "same sound" (audio fingerprints, with a match threshold calibrated on degraded copies and
  on clips of other classes); 88 clips that also occur in MUSAN (part of VoxTurn's training noise) were
  removed. Whole groups were assigned to splits by a seeded rule: knock and decoy groups so that test gets
  at least 40 clips from 25 uploaders and replication at least 20 from 12; phone and doorbell groups, too few
  for these minimums, in fixed proportions of the groups (test 30%, replication 15%, dev 10%, train 45%),
  with the actual numbers reported (table above). Four Wikimedia Commons recordings are used in the test
  split only.
- Footsteps (indoor surfaces only), doors (opening, closing) and cupboards or drawers came from FSD50K under
  the same rules (for doors, clips whose labels are only door-related sounds such as a squeak were also
  accepted, because the strict pool was small); test and replication add door recordings uploaded to
  Freesound after June 2023. An uploader already in a split keeps it; the remaining groups are split in the
  proportions above. The test split should hold at least 8 recording groups of each of these classes;
  otherwise its "stop, movement" results carry a "few recordings" note.
- TV and radio speech: half real recordings (VoxPopuli, European Parliament speech; speakers split train
  50%, dev 15%, test 20%, replication 15%), half Kokoro voices of the same split, so that "a real recording
  means TV" is not a shortcut; both pass through a loudspeaker response and the room, 5 to 20 dB below the
  user's speech.
- Voices are split by voice (stratified by accent and gender), rooms by room (stratified by size), noise by
  environment.
- Test and replication only (unseen material, section 3.9): Piper voices (40 test and 20 replication
  speakers), phone-vibration recordings, and recordings uploaded to Freesound after June 2023 (any that
  match an older source by fingerprint are excluded; uploader groups go to test and replication 2:1).
- **Recording-group caps:** within one part and one recording type, a single recording group may supply at
  most 30% of the dev units and 15% of the test and replication units; when a type has few groups, the cap
  is 1 / (number of groups), an even split.
- The dev split has only 4 phone and 4 doorbell groups, so dev choices are noisy. It was kept as it was:
  moving groups from training (20 phone, 16 doorbell groups) would weaken every trained system, the
  detector included. The cost is handled by the "best setting on test" rows (section 6.2) and by
  estimating the test precision before the test (section 7.3).

### 2.2 Lines and episode scripts

- Lines were generated with fixed prompts (gpt-6.1-sol) and checked by another model family (Claude, with
  the fixed prompts in `configs/checker_prompts_v1.md` and `configs/checker_prompts_v2.md`). The bar is
  "clearly", not "most likely": an answer line must clearly be for the caller or visitor, a line after a
  rejected call clearly for the assistant, a keep-going line or backchannel clearly a request to continue.
  Lines that fail are dropped.
- Answer lines and keep-going lines are pooled, normalised, deduplicated and split by phrasing (train 50%,
  dev 15%, test 20%, replication 15%), so test phrasings were never trained on. Minimums after the check:
  100 phone answers, 50 door answers, 60 keep-going phrasings (otherwise stop). Backchannels are a small
  closed set common to all splits (at least 8). Flip lines ("Hello?" and similar lines that fit either
  listener) are common to all splits, except 4 phone flip lines kept for the test split as an unseen subset.
  Other lines are deduplicated across splits.
- The conversations and the lines kept from version 0.3 (flip lines, "hold on" lines, family-call and
  side-talk lines, return lines) were generated and checked the same way (section 15.1). Conversations were
  deduplicated and split 300 / 100 / 600 / 600; each unique non-flip line belongs to one split (train 40%,
  dev 10%, test 25%, replication 25%); passing flip lines are common to all splits except a held-out set of
  test-only phrasings ("Hello?" always stays common).
- Episode scripts were generated with the same model for the types and shares of section 3.5, then
  checked in time order: at each decision point, the checker saw only what a person in the room could
  hear up to that point (never the caller's side) and answered "to the assistant / to someone else /
  cannot tell" (for stop points "stop / continue / cannot tell"). One batch never held two points of the
  same episode. If any point was unclear or disagreed with the script, the whole episode was dropped and
  regenerated. A second, differently worded prompt produced 100 test and 100 replication episodes (an
  unseen subset). Scripts of all splits were checked together, before splitting, the same way.

### 2.3 Rendering

- Each group or episode is laid out as a timeline from real TTS durations, mixed at 16 kHz, and cut into an
  8 s window ending at each decision point. Gemini listening hears an episode from its start to the point.
- Versions of a group are identical outside the element under test and its reverberation tail.
- User positions: near (about 1 m), across the room (3 to 5 m) and at the door, with the same share of far
  positions among "respond" and "do not respond" lines. Pauses before lines come from one distribution for
  all classes. Levels, reverberation and signal-to-noise ratios are drawn by fixed rules.
- Positions come from the simulated room responses, which list no source positions: each response's source
  distance is estimated from its direct-path delay and its direct-to-reverberant ratio; within each room,
  "near" is the closest 20% of its responses, "across" the 40th to 80th percentile and "door" the farthest
  10%. Footsteps walking away use responses of increasing distance.
- The assistant's speech is synthesised only to time its turns; it is not mixed into the microphone (except
  in the faint-echo subset).
- Transcripts, speaker tags and detector output are computed once and shared by all systems.

### 2.4 Sizes

Units actually rendered (a unit is one group of versions or one episode):

| Part | Train | Dev | Test | Replication |
|---|---|---|---|---|
| Respond, hearing (groups of 3) | 3,600 | 200 | 600 | 600 |
| Respond, reaction (groups of 2) | 3,600 | 150 | 400 | 400 |
| Stop, reaction (groups of 4) | 3,600 | 150 | 400 | 400 |
| Stop, movement (groups of 4) | 3,600 | 150 | 400 | 400 |
| Episodes | 875, each rendered 4 times | 194 | 580 | 582 |
| Floor checks (6 kinds) | 21,600 | 600 | 1,800 | 1,800 |
| Unseen subsets | — | — | 1,195 | 1,195 |
| Report-only cells | — | — | 800 | 800 |

- Training groups are 12 independent renders of one group per contrast and training conversation.
- Episodes are fewer than planned (200 dev, 600 test and replication) because scripts that failed the
  check were dropped.
- The dev split has 3,692 decision points; the test split has 10,401 outside its unseen and report-only
  subsets.

### 2.5 Data checklist ("step 0")

Before any training, the data stage followed a fixed checklist; the code cites its items as "step 0 item
N". This is the version 0.4 list; the list of version 0.3, which some data scripts cite, is in section
15.1.
1. new recording classes: counts, selection (indoor footsteps; doors opening and closing), comparison with
   MUSAN, grouping, split by uploader, minimum counts;
2. TV speech: licence, overlap with MUSAN and VoxTurn's training data, split by speaker, loudspeaker and
   room processing;
3. speaker-ID model: licence, enrolment method, accuracy on dev;
4. new lines: generation, check, normalisation, split by phrasing, minimum counts;
5. episode scripts (both prompts): generation, time-ordered check, share of ambiguous short lines and of
   each type;
6. renderer: timelines, windows, assistant state, positions, footsteps, ring looping and stopping,
   sample-identity checks;
7. situation-text format and the policy text;
8. interval method rewritten and calibrated (section 7.3);
9. newest available models, and whether Gemini returns token probabilities;
10. render train and dev; the shortcut check passes (section 4);
11. render test and replication, run automatic checks only, seal them;
12. the leak check passes (section 4).

## 3. Test parts and scoring

"Phone versions" use a phone or mobile ring; "door versions" use a doorbell or a knock, half each.

### 3.1 Respond, hearing

The assistant is silent; the same short line follows different sounds.

| Version | Sound, then the line | Correct |
|---|---|---|
| 1 | a ring (door versions: a knock or doorbell) | do not respond |
| 2 | no event | respond |
| 3 | a decoy (unrelated household sound) | respond |

- Lines: 8 common phone flip lines (4 more only in the test split, as an unseen subset) and 5 door lines
  ("Hello?", "Yes?", "Yeah?", "What's up?", "Who's that?").
- The sound ends 0.3 to 1.5 s before the line; the line comes at the start of the session or after at least
  3 s of assistant silence. Ring recordings are looped to the drawn length, so every recording can be used.
- Words and history are identical across versions: only the sound separates them.

### 3.2 Respond, reaction

The assistant is silent. A ring (or doorbell or knock), then:

| Version | The user's reaction | Correct |
|---|---|---|
| 1 | answers naturally ("Hi, this is Alex.", "Coming!") | do not respond |
| 2 | the same ring, cut off at the same moment (call rejected, or nobody goes to the door); the user goes on talking to the assistant | respond |

- About a third of the version-1 answers are common flip lines; the rest come from the answer pools split
  by phrasing.
- About a third of the version-2 lines open like an answer ("Hello? Are you still there? So, Thursday?"),
  so a rule such as "a line that starts with Hello is not for the assistant" fails.
- The two versions are identical up to the start of the line; the point is 0.2 s after the line.

### 3.3 Stop, reaction

The assistant is talking (for example reading a recipe). An event, then:

| Version | The user's reaction | Correct |
|---|---|---|
| 1 | answers (as in 3.2) | stop, do not respond |
| 2 | "Ignore that, keep going." (keep-going pool) | continue |
| 3 | a backchannel such as "mm-hm" | continue |
| 4 | stays silent | continue |

- Version 1 is scored on both decisions; versions 2 and 3 are asked both but scored on stop only; version 4
  has only stop, at the time of version 1's decision point.
- The ring state does not reveal the answer: version 1's ring stops when the call is picked up, version 4
  rings on (15 to 30 s), versions 2 and 3 ring on in half of the groups and stop at version 1's moment in the
  other half. Door versions ring or knock once.

### 3.4 Stop, movement

Door versions only; the assistant is talking; the user says nothing.

| Version | Sound | Correct |
|---|---|---|
| 1 | doorbell or knock, footsteps moving away, the front door opens | stop |
| 2 | the same doorbell or knock; nobody moves | continue |
| 3 | no doorbell; the same footsteps (same clips and positions), then a cupboard or drawer at the moment the door opened in version 1 | continue |
| 4 | no doorbell; speech from a TV or radio | continue |

- The decision point is 1 s after the door-opening sound starts, the same moment in every version.
- In half of the groups the visitor speaks before that point.
- Footsteps are one recording cut into pieces and played at 3 to 4 fixed positions; the visitor is another
  voice of the split, placed at the door.
- Also reported: version 4 split by real or synthetic TV voice, and the part split by door-sound source
  (FSD50K or post-2023 Freesound).

### 3.5 Episodes

Multi-turn scenes of 30 to 60 s that start with a conversation topic. About half of the events happen while
the assistant talks. Only the user's side of a phone call is audible; a visitor is audible.

| Type | Share | Content |
|---|---|---|
| Phone | 30% | ring, pick up, call (user side only), return |
| Visitor | 25% | doorbell or knock, answer or silently go, door opens, visitor talks, door closes, return |
| Very short | 10% | the user picks up, says one line ("Can't talk now, I'll call you back."), returns |
| Control: no event | 15% | conversation with the assistant only |
| Control: event ignored | 20% | the user ignores the event and keeps talking to the assistant |

- In phone and visitor episodes, about 30% have a question to the assistant in the middle of the call
  (respond), after which the user goes back to the call (do not respond); about 40% end without a goodbye;
  about 50% have two or more turns with the assistant after the return; returns vary ("Sorry, where were
  we?", the old topic, a new question).
- At least a third of the user's lines to callers and visitors are short lines that could also be for the
  assistant ("Yeah.", "Okay.", "What time?"); they are reported separately.
- Decision points: respond 0.2 s after each user line (and stop as well if the assistant is talking); stop
  1 s after the door opens when the user silently goes to the door; one stop point in each
  ignored-event control, at a delay drawn from the same distribution as the answering points. Visitors'
  lines have no decision point.
- Position and pause do not give the answer away: far positions occur equally in respond and
  do-not-respond lines, and pauses and call lengths come from shared distributions.
- An episode counts as right only if every scored decision in it is right. Also reported by point type
  (picking up, during the call, mid-call question, return and later, stop points, control points) and as
  the mean over point types.

### 3.6 Group scoring and the five-part mean

- In the four single-line parts a group counts as right only if every scored decision of every version is
  right. Episodes use whole-episode accuracy.
- **Main score: the five-part mean**, the unweighted mean of the four group accuracies and the episode
  accuracy, computed on the main body only (no unseen or report-only units).
- A missing score counts as wrong.
- Secondary: each part, the accuracy of each version, episode point types, the ambiguous short lines, the
  sound-only part (3.8), the floor checks (3.7) and the unseen part (3.9).

### 3.7 Floor checks

Control cases with a sound in which the user still talks to the assistant: a system must not stop or ignore
the user by mistake. They are not part of the main score.

| Check | Situation | Correct |
|---|---|---|
| Ignored event during a question (two kinds: phone, knock) | assistant silent; the phone rings or someone knocks while the user asks a question; the user ignores it and finishes | respond |
| "Hold on" while silent | assistant silent; after an event the user says "Hold on, someone's at the door." or similar | respond (briefly) |
| "Hold on" while talking | assistant talking; the same kind of line | stop and respond |
| Ordinary interruption | assistant talking; no event; the user asks "Wait, how much salt?" | stop and respond |
| Quiet | assistant talking; no event; nobody speaks; the point falls at a random time | continue |

Each check must reach at least 0.90 at a system's operating point (section 6.1). On the test split each is
reported with its interval; the decision table uses their mean (section 8.3).

### 3.8 Sound-only part

- All "respond, hearing" groups plus the "stop, movement" groups whose visitor does not speak before the
  point (test: 600 and 200 groups). In both, the text is identical across versions, so a text-only system
  scores about 0 by construction. Episodes are not included: they have no sample-identical controls.
- Metric: the mean of the two group accuracies.
- Uses: a design check (text only must score about 0 here, section 6.3) and the listen step of the decision
  table (section 8.2).
- Limitation stated in advance: the decisive sounds are detector classes, so detector labels are in
  principle enough here; this part measures how well a system hears, not information the labels cannot
  carry. The matched grounding test (section 10) removes this limitation.

### 3.9 Unseen subsets and report-only cells

Unseen subsets (test and replication only; reported separately; their pooled score enters one condition of
the decision table):
- phrasings that occur only in the test split: 4 phone flip lines in "respond, hearing" (100 groups); in
  the reaction parts the main body already uses unseen answer and keep-going phrasings;
- a second TTS engine (Piper voices): 100 groups per contrast and 100 episodes;
- an unseen event type (phone vibration): 100 "respond, hearing" groups;
- recordings uploaded after June 2023: 100 groups per contrast;
- episodes from the second script prompt: 100.

Unseen metric: for each part, the group (or episode) accuracy over all units of its unseen subsets pooled,
then the mean over the parts that have unseen subsets.

Report-only cells, outside every decision: a family member calls from another room and the user answers,
and the user turns to talk with someone in the room (300 groups each); a faint echo of the assistant, 25 to
35 dB below the user's speech (100 groups each in "stop, reaction" and "stop, movement").

## 4. Leakage controls

- **Disjoint splits** (section 2.1). Shared on purpose and stated: the common flip lines and backchannels,
  and the scene types and event classes (only the recordings differ). Conclusions are limited to these
  scenes; unseen phrasings, voices, event types, recordings and script prompts are reported separately.
- **Shortcut check** (before any training): a gradient-boosted classifier sees only generator parameters,
  never audio or text; trained on train, scored on dev. For every contrast its AUC (0.5 = chance) must be
  at most 0.55; otherwise the generator is fixed and the check repeated. Gating parameters: for "respond,
  reaction" ring length and cut time, line start, position, room, noise and level; for "stop, reaction" the
  same plus whether the ring still rings at the point and the time since the event; for "stop, movement"
  the point time, footstep positions and levels, door or cupboard level, TV level, room and noise; for
  episode lines the pause, position, level, room and noise; for silent episode points the time since the
  event, level, position, room and noise. Word count, position in the episode and time since the event are
  reported but never gate, because they belong to the content.
- **External models:** no event, new-class or TV recording comes from MUSAN or VocalSound (VoxTurn's
  training data). The speaker-ID model's training data (VoxCeleb) was checked against our voices: the user
  and assistant voices are synthetic, and the TV speakers' names were matched (exactly and loosely) against
  the VoxCeleb identities, with VGGFace2 as a broader reference; for test and replication only counts were
  printed. Units with a matched TV speaker, or one without a name, are left out of training, thresholds and
  scores. The training data of VoxTurn's own backbone is not public; the post-2023 recordings check this in
  part. The line generator (OpenAI), the checker (Anthropic) and the rivals (Google Gemini, Qwen) come from
  different model families.
- **Single use of the test split:** all choices on dev. Before the final run, test and replication received
  automatic checks only (counts, hashes, levels, times); nobody looked at their content or at system outputs
  on them. The loaders open them only in the final run (section 12.5).
- **Leak-check script** (must pass before training or testing): recording groups, voices, rooms, noise,
  lines and TV speakers stay in their split; recording-group caps hold (tolerance 0.05); the situation text
  built with and without the gold labels is identical; which answers are scored never enters an input.

## 5. Systems compared

### 5.1 Shared front end

Every system that uses one of these parts uses the same one:
- **Sound-event detector:** PANNs CNN14 (AudioSet weights), fully fine-tuned on the training split for
  seven classes (phone, doorbell, knock, footsteps, door, cupboard or drawer, decoy) with frame-level
  targets; epoch chosen on dev.
- **Transcripts:** Whisper-small (local systems), Whisper large-v3-turbo (Gemini systems).
- **Speaker tags:** ECAPA-TDNN; the user is enrolled with a separate passage of about 10 s in the same
  voice; the threshold is chosen on dev (best balanced accuracy) and shared by all systems.
- **Sentence embeddings:** all-MiniLM-L6-v2.
- **The situation text** of section 1.2.

### 5.2 Trained candidates and the main system

Trained on the training split, settings chosen on dev; gold answers are only labels, never inputs, and
respond enters the loss only where it is defined.
- **Same model, original read-out:** VoxTurn (frozen) with a new LoRA adapter on its language model and a
  new option head; input the 8 s window and the situation text; two multiple-choice questions in VoxTurn's
  own format.
- **Late fusion of frozen audio and text features:** a small network over detector embeddings and class
  scores, VoxTurn's five answers, sentence embeddings of the line and the history, speaker tags, the
  assistant's state, the previous answer (from the correct history), the time since each event class and
  the number of lines since the event.
- **Fine-tuned Whisper encoder + features:** the Whisper-base encoder reads the window, plus the late-fusion
  model's non-audio features.

The plan's rule took the best of these three on the dev five-part mean (floors met): the original
read-out (0.944). Before the freeze, the amendment replaced it by **ours**: the same fine-tuned model with a
retrained read-out, step 2d. The model is unchanged; only its last layer is new: one logistic regression
per decision over the last-layer states at the two answer options (2,048 numbers) and the audio encoder's
mean and max for each of the 8 seconds of the window (16,384 numbers). It was trained on data in which
each of five label families (ring-type events, decoys, footsteps, door, cupboard) was independently removed
or falsely added with probability 0.5, the false labels copied from the timing of real detections; gold
answers were unchanged. Thresholds by the operating-point rule: stop 0.139, respond 0.248.

How the main system was selected:
- There were three selection rounds. The first chose step 2b (with the whole-window audio summary) by a
  rule written after all dev numbers were known: grounded on "respond" (G1a lower bound above 0.5) and no
  lower five-part mean than the original read-out. The second and third rounds used rules written before
  their runs.
- Step 2c would have replaced it only if its stop grounding reached 0.55, its respond grounding 0.65 with a
  lower bound above 0.5, and its five-part mean 0.934 with all floors met. It failed and was not adopted.
- Step 2d had four conditions: respond grounding at least 0.65 with a lower bound above 0.5; stop grounding
  at least 0.70; five-part mean at least 0.934 with floors met; and, with the stop decision point moved to
  0.5, 1.5 and 2 s after the door opens, a mean gain in stop grounding of at least 0.10 over step 2c with
  the whole-window summary. All held, so step 2d became the main system. (Grounding here is the grounded
  group accuracy of section 10.)
- Stated limits: the bar of 0.934 is the original read-out's dev score minus 1 point; under the first
  round's rule (no loss at all), step 2d (0.938) would not have been chosen. The per-second idea came from a
  secondary row of a diagnostic whose primary summary failed, so the dev stop numbers of step 2d are
  optimistic.
- The other 12 read-out candidates also run on the test split, as reported rows outside the decision
  table; the cost of grounding (step 2 with the whole-window summary minus ours on the five-part mean) is
  reported with its interval.

### 5.3 Attribution controls

Trained like late fusion on our data; not in the main comparison:
- **Label reader trained on our data:** detector class scores and timing features, transcripts, speaker
  tags and the whole situation text, but no audio features. It also enters the listen step (section 8.2).
- **Text only:** the situation text without any detector content.
- **Text only, current line only:** only the transcript and speaker tag of the last line.

### 5.4 Rivals

Not trained on our labels; every setting, prompt and example count chosen on dev.
- **Tuned rule system:** a two-state rule ("with the assistant" / "away"). It enters "away" when an
  attention sound (phone, doorbell, knock) above a confidence threshold falls within T s before a line that
  is an answer phrase or is short without a to-the-assistant phrase, or when footsteps or a door follow a
  doorbell or knock within T2 s. It returns on a return or to-the-assistant phrase, a call-ending phrase in
  the previous line, a closing door, or a line close to the assistant's topic (each switch chosen on dev).
  It stops when "away" or when the user said a line that is not a backchannel or keep-going phrase; it
  responds when "with the assistant" and the line is tagged as the user's. Phrase lists are fixed lists
  plus 1- to 3-word phrases mined from training lines. All 2,488,320 settings are evaluated exactly on dev;
  three options were added in its favour before its results were used (speaker tags off, backchannels never
  enter "away", phrases mined from Whisper transcripts).
- **Gemini readers** (fast: gemini-4-hydrogen; strong: gemini-4-carbon) and the **local LLM reader**
  (Qwen3.5-4B, 4-bit): the prompt holds the full policy, K training examples (6, 24 or 48, cycled over parts
  and versions so that every kind of situation appears) and the case's situation text with detector
  confidences; the questions follow section 1.1; temperature 0. Token probabilities are the score where
  available, with thresholds chosen on dev in the rival's favour; otherwise the one-word answer is the score
  (1 or 0; 0.5 when the response has no answer word). Gemini: default thinking and no thinking both tried.
  Local LLM: thinking off, two reading positions (first token, or after an answer label), K and reading
  chosen on dev; a K that did not fit the 8 GB card would be dropped and logged.
- **Gemini listening to the audio** (gemini-4-carbon): hears the 8 s window (an episode from its start),
  with the same situation text and policy; 0 or 6 audio examples from the training split; its prompt says
  that the situation text may be wrong and that what it hears counts.

### 5.5 References

Reported only; never part of a choice or a verdict:
- **Gemini reader given perfect labels:** clean script words, true speakers and true sound labels
  (footsteps and door included), otherwise as the strong reader. It separates "heard wrong" from "reasoned
  wrong".
- **Off-the-shelf VoxTurn:** respond if its end-of-turn probability is at least 0.8; stop if a line just
  ended and it predicts an interruption.
- **Environment-blind gatekeeper:** stops whenever the user (by speaker tag) has just spoken, unless VoxTurn
  hears a backchannel; responds to every line tagged as the user's; ignores sounds and history.
- **Copy the previous decision:** repeats the previous correct answer in episodes; always "respond" and
  "continue" elsewhere.

## 6. Thresholds and settings (development set only)

### 6.1 Operating-point rule

For every score-based system, the stop and respond thresholds (and, where they exist, the prompt setting,
K or reading) are chosen on dev:
1. among all settings where every floor check is at least 0.90, take the one with the highest five-part
   mean;
2. if no setting meets the floors, take the one whose lowest floor check is highest (ties: higher five-part
   mean).

The threshold grid is the same for all systems: steps of 0.001 plus dense values near 0 and 1 (down to
10^-12 from either end), 1,183 candidates per threshold, because API probability scores lie very close to 0
and 1. Epochs of the fine-tuned models and the detector are chosen on dev; weight decays and network
sizes on dev or by held-out loss inside the training split.

### 6.2 Equal tuning of the rivals

- Every rival gets the same front end, the same situation text and the same operating-point rule. Uncertain
  choices are made in the rival's favour: the larger Whisper model for the Gemini systems, probability
  scores with thresholds tuned on dev, both thinking modes tried, extra settings for the rule system.
- Fairness additions, chosen by the author before the freeze although they make the rivals stronger: a note
  that the situation text comes from automatic tools that can be wrong (`configs/b2_note_v1.txt`) and the
  detector's measured reliability on dev by class and confidence band
  (`configs/b2_detector_reliability_v1.txt`) are candidate prompt settings for the Gemini readers. Gemini
  listening got the reliability table as an extra candidate (used in answers mode, section 9.1).
- Prompts were tuned on dev only and then fixed.
- Because the dev set is small, every rival is also run at all its candidate settings on the test split and
  reported at its best one ("best setting on test"). This row is not confirmatory and cannot change the
  verdict; if ours does not beat a rival's best test setting, that is stated.

### 6.3 Development checkpoint and basic checks

Before any API rival was run, every local system was reported on dev at its operating point (five-part
mean, parts, floors, sound-only part, episode point types, differences with intervals). Two basic checks
had to pass on dev (never on test): ours at least 30 points above off-the-shelf VoxTurn (otherwise look for a
bug), and text only about 0 on the sound-only part (above 10% would mean that the text leaks the answer,
and the generator would be fixed before the freeze). Two diagnostics fixed in advance were reported as
well: stop accuracy on the keep-going and backchannel versions of "stop, reaction", split by whether the
ring had already stopped; and respond accuracy on episode lines, split by the pause before the line and by
the door position.

## 7. Intervals

### 7.1 Clustering factors

Decisions are not independent: many units share a conversation or a recording. Each unit carries three
cluster labels:
- **conversation** (every unit is built on one);
- **main event recording group** (the group of its phone, doorbell or knock recording; a unit without an
  event is a cluster of its own);
- **second key recording group:** the decoy's group in "respond, hearing", the door sound's group in "stop,
  movement"; a cluster of its own elsewhere.

### 7.2 Combining one-way cluster bootstraps

- For each factor alone and for each intersection of factors, a cluster bootstrap draws one Poisson(1)
  weight per cluster (B = 2,000 replicates, seed 1) and recomputes the statistic; for a difference, both
  systems use the same weights.
- The variances are combined by inclusion and exclusion, the multiway clustering formula of Cameron,
  Gelbach and Miller: the three one-way variances, minus the three pairwise-intersection variances, plus
  the three-way-intersection variance. If the result is smaller than the largest one-way variance, the
  largest one-way variance is used.
- 95% interval = point ± 1.96 × SE. This applies to the five-part mean (resampled directly), each part,
  the floors, the sound-only part, the unseen part and every difference.
- Why this method: the earlier version multiplied one random weight per conversation and one per recording
  group; on simulated independent items that interval was 2.8 times too wide, against 1.1 times for this
  method.

### 7.3 Calibration and precision estimate

- Before the dev checkpoint, the method was run on simulated outcomes with the real dev unit structure.
  With independent units the bootstrap SE had to be 0.9 to 1.25 times the true SE; with clustering by the
  event group only, within 20% of the known spread. Otherwise the code would be fixed first. The same check
  on the matched grounding groups gave ratios of 1.11 and 0.96.
- After the systems were frozen and before the test, the SE of each main difference was estimated on dev
  and scaled to the test split once by the number of event groups and once by the number of conversations;
  the wider estimate was reported.

### 7.4 Variants

- **Five-factor sensitivity:** the background-noise recording and the user voice are added as cluster
  factors. Reported beside every main difference; a statement that an interval lies above 0 must also say
  when the five-factor interval includes 0. Differences per background-noise recording, and with each one
  left out, are reported as well.
- **t intervals** in the grounding test and the noise analysis (section 10.8).
- **Rivals scored on answers:** the SE of their dev answer loss is added (section 9.5).

## 8. Main comparison and decision rule

### 8.1 Main comparison

Ours minus each rival (tuned rule system, local LLM reader, Gemini reader fast, Gemini reader strong,
Gemini listening) on the five-part mean of the test split. Ours must beat every rival, not only the one
that is best on dev: with 4 phone and 4 doorbell groups the dev set cannot pick the strongest rival
reliably, and beating all of them is stricter. Because every comparison must pass, each uses its own 95%
interval with no multiple-comparison correction.

### 8.2 Listen step

Checked only when every rival is beaten: on the sound-only part, ours minus the label reader and minus
every rival must be at least 5 points with the interval above 0. Testing the main comparison first and the
listen step second keeps the overall error near 5% (at most about 10% with the single replication check of
row 4).

### 8.3 Decision table

Rows are checked in order; the first row that holds is the verdict. "Difference" is ours minus a rival on
the five-part mean; "beaten" means a difference of at least 5 points with the interval above 0.

| Row | Condition | Verdict |
|---|---|---|
| 1 | Every rival beaten; the mean of the floor checks at most 5 points below each rival's; unseen-part difference at least 0 (point) against each rival; the listen step holds | Clear win, and the gain comes from listening |
| 2 | As row 1, but the listen step fails | Clear win from training on paired data; the gap to the label reader is reported (if they tie: a trained label reader is enough) |
| 3 | Every rival beaten, but the floor or unseen condition fails for some rival | Not counted as a clear win; recorded as is and left to the author |
| 4 | Every difference at least 5 points (point), floor and unseen conditions hold, but some interval includes 0 | Check once on the replication split with frozen systems and judge again by this table; if that lands in row 4 again, rows 5 and 6 decide |
| 5 | Every rival either beaten or tied (interval lower bound above -3 points), and ours has a practical advantage over each tied rival (at least 10 times faster per decision, or the rival needs an API call and ours does not) | Tie with a practical advantage: a local model that matches the large-model readers while being much faster or needing no API call |
| 6 | Otherwise | Report the measurements and the benchmark only |

On the laptop timings no local rival is 10 times slower than ours, so row 5 is possible only if every local
rival is beaten and only API rivals tie.

### 8.4 How the code reads the table

`p1v4_test_report.py` prints these readings with every report:
- (R1) The floor mean counts the two kinds of "ignored event during a question" as one check (5 checks); the
  mean over all 6 kinds is reported too.
- (R2) Row 5 is applied as written (it checks neither floors nor unseen); the stricter reading with row 1's
  conditions is reported beside it.
- (R3) "Beaten" inside row 5 means the main-comparison condition only.
- (R4) Unseen subsets exist only in test and replication; on dev that condition is "not evaluable" and
  taken as met.
- (R5) The label reader "ties" when the interval of ours minus it includes 0.
- (R6) Practical advantage: API rivals always; local rivals by median laptop time per decision.
- (R7) The three factors of section 7.1 decide; five-factor intervals are a sensitivity row.
- (R8, R10) A rival scored on answers has its dev answer loss subtracted and the loss's SE added (section
  9.5); the unseen-part difference gets the same subtraction; raw values are kept.
- (R9) Ours has the original read-out's latency: its read-out is a linear map on the same forward pass.
- (R11) The table is also computed with the original read-out as "ours" (the plan's rule before the
  amendment) and reported beside the main table; differences per background-noise recording are described.
- (R12) A rival re-chosen after a failed drift check (section 9.3) runs at its re-chosen setting.

### 8.5 Rows reported beside the table

These do not change the verdict:
- per-part differences of ours minus every rival, with intervals (always listed);
- attribution: the three trained candidates among themselves; ours minus the label reader, text only, and
  text only (current line) on the five-part mean, each part and the sound-only part;
- every rival at its best setting on the test split (section 6.2);
- "stop, movement" by door source, and its TV version by real or synthetic voice;
- the references (section 5.5);
- latency per decision (median and p90, laptop GPU, front end included);
- the fixed extra rows of the grounding test (section 10.10).

## 9. Gemini systems on the test days

### 9.1 Probabilities or answers

Shortly before the freeze, the Gemini API began refusing token-probability requests ("Logprobs is not
enabled for this model"). The plan already said: use probabilities when available, otherwise answers. The
amendment fixed the details before the test split was opened. On the test day a probe sets one scoring mode
per Gemini system for the whole run. Settings for both modes were frozen; the answers-mode settings and
thresholds were re-chosen by the operating-point rule on the stored dev answers, without new calls.

| System | Answers mode | Probability mode |
|---|---|---|
| Gemini reader, fast | 48 examples, thinking, note and reliability table | the same setting |
| Gemini reader, strong | 48 examples, thinking, note and reliability table | 48 examples, no thinking, note and reliability table |
| Gemini listening | 6 audio examples, no thinking, reliability table | 6 audio examples, no thinking |
| Reference with perfect labels | 48 examples, no thinking | the same setting |

The "best setting on test" row of a rival uses the same mode as its main run, over all its planned
candidates (example counts × thinking modes), with the frozen prompt.

### 9.2 Order before the test split is opened

1. Check every hash of the freeze record; stop on any mismatch.
2. Probe: one probability request per model on a fixed dev question. Only the specific "not enabled"
   refusal sets a system to answers mode; any other failure stops the run for inspection. An existing mode
   file is never overwritten.
3. Drift check on dev questions (section 9.3); re-choice if it fails.
4. Open the test split and run all systems. While they run, only counts and errors are logged: no test
   metric is computed or printed.
5. Compute metrics only after every output (grounding test and noise analysis included) is complete.

### 9.3 Drift checks and re-choice

- A fixed sample of about 250 dev decision points, stratified by part (seed 11; 301 questions), is re-asked
  in the day's mode. The share of decisions that flip at the frozen thresholds must stay at or below a
  limit. Probability mode: 3% (fast reader), 2% (strong reader), 8% (listening), 2% (perfect-label
  reference), from earlier repeat calls. Answers mode: r + 3 points, where r is the flip rate measured on dev
  before the freeze (fast 1.7%, strong 0.7%, listening 4.0%, giving limits of 4.7%, 3.7% and 7.0%; the
  reference's limit was set the same way). 3 points is about two standard errors at this sample size; an r
  above 10% would have meant the model had changed, and its dev run would have been redone before the
  freeze.
- Each check is written to a new dated folder. A check with fewer answers than questions counts neither as
  a pass nor as a fail and is asked again. The first complete check of a day decides that day; it cannot be
  repeated until it passes.
- A rival that fails, before its first test answer, has its dev candidates re-asked in the day's mode and
  its setting and thresholds re-chosen by the operating-point rule (in answers mode, its answer loss is
  recomputed); the re-choice is logged and written to a file the final scripts read. Once a rival has test
  answers, no re-choice is allowed.
- On every later day a rival needs a complete, passed check of that day (at its current setting) before
  more test questions; if no check passes within 7 days, its remaining items are reported as missing.
- After a switch from probabilities to answers (section 9.4), only checks in the new mode count.

### 9.4 Incidents during the run

| Incident | Rule |
|---|---|
| A probability-mode rival loses probabilities | It switches entirely to answers mode at its answers setting and is re-asked from the start; probabilities already received are neither used nor mixed in. Only a reply that has an answer word but no probabilities triggers this |
| An answers-mode rival regains probabilities | Nothing changes |
| Transient errors (timeouts, rate limits) | Retry without limit |
| The model is withdrawn or renamed, or its run is not finished by a fixed date | The rival counts as "not beaten" in the decision table and the listen step (in its favour); partial results are reported; no other model replaces it |
| A bug in our code | Fix only the bug, re-run the dev rehearsal (identical outputs required), re-hash the fixed files (section 12.7), log it and continue; no test metric is looked at meanwhile |

### 9.5 Compensating answers-only scoring

Answers mode weakens a rival through no fault of its own, so that loss is given back:
- For a rival in answers mode, every difference "ours minus it" first subtracts the rival's dev loss from
  answers-only scoring (its dev score in probability mode minus its dev score in answers mode), only when the
  loss is positive. The point and the interval move together.
- The loss's SE (from the dev three-way bootstrap) is added: SE = sqrt(SE_test² + SE_loss²), and the
  interval is the point ± 1.96 × SE.
- The unseen-part difference gets the five-part loss as well; raw values are kept.

| Rival | Loss on the five-part mean (SE) | Loss on the sound-only part |
|---|---|---|
| Gemini reader, fast | 0.10 points (0.13) | 0.25 points |
| Gemini reader, strong | 0.59 points (1.25) | -1.78 points (not subtracted) |
| Gemini listening | 2.05 points (1.31) | -4.22 points (not subtracted) |

On dev the adjusted differences equal the probability-mode differences by construction, so every rival's
outcome is the same as with probabilities. Alternatives considered and rejected: report both scorings and
take the one better for the rival (the test run produces only one), or add a fixed extra margin (treats
three rivals with very different losses alike).

### 9.6 Missing answers

A score file is written only when every question has an answer. A single request is retried up to 12
times; then whole rounds of the missing questions are re-asked until none is missing; if some are still
missing, the run stops before any metric. A response without an answer word scores 0.5, as fixed in the
plan. In the grounding test a missing answer counts as wrong.

### 9.7 What happened on the test days

On each of the three test days the probe found no token probabilities, so all four Gemini systems ran in
answers mode, with the loss adjustment above. Every daily drift check passed; no setting was re-chosen.

## 10. Matched grounding test

The main comparison tells who scores higher. A higher score can come from good labels, from shortcuts in
the data, or from hearing. This test checks whether decisions follow the sound a system hears or the
detector's label. It was fixed before the test split was opened and ran on the test split with the main
run. Its criteria decide which statements about grounding may be made; they never change the verdict of
section 8. The first reviewed draft of the package was version 0.2; versions 0.3 to 0.5 adapted it to the
new read-outs; version 0.6 (its section 15) is final.

### 10.1 Groups

- **Respond:** the main body of "respond, hearing" (test 600 groups, dev 200); version 1 ring or knock,
  version 2 nothing, version 3 decoy; one respond decision per version.
- **Stop:** the "stop, movement" groups whose visitor does not speak (test 200, dev 72); versions 1 to 4;
  one stop decision per version.
- Criteria use the main body only. The unseen subsets are run by every system (API rivals included) and
  described per subset, without criteria. Phone-vibration groups are left out (the detector has no such
  class) and counted.

### 10.2 Conditions

- **Base:** every non-audio input of every version is replaced by version 2's: all transcripts and speaker
  tags, and all detector output before the point (the sound list and the frame scores). Version 2 has no
  event, so the base carries no trace of one.
- **Respond conditions:** event detections (phone, doorbell, knock) are removed and their frame scores
  capped at 0.1. "None" adds nothing. "Confidence c" (c = 0.3, 0.5, 0.7, 0.9, 0.99) adds one detection with
  version 1's event class and time span at confidence c, frames over the span set to c, in every version.
- **Stop conditions** (since version 0.6): "none", and false movement labels at confidence 0.5, 0.9 and
  0.99: on the base, footstep and door detections are removed and their frames capped at 0.1; then version
  1's footstep and door spans (merged when closer than 0.3 s, clipped at the point) are added at confidence
  c to every version. The label is true in version 1, false in versions 2 and 4, and half true in version 3
  (real footsteps, a cupboard instead of a door). Stop criteria average these four conditions; the six
  ring-label conditions on stop are a reported row.
- **Result:** within a group, every version gets the same situation text and the same non-audio features;
  only the audio and what is computed from it (detector embeddings, VoxTurn's answers, our model's audio
  path) differ. In a respond group the injected label is right in one version of three at every
  confidence, by construction, so this is a stress test, not a picture of normal use (natural inputs:
  section 10.9).
- **Natural:** the unchanged inputs, re-run by this test's own code as a baseline (section 10.4).

### 10.3 How each system reads the matched inputs

- Text readers (ours and every read-out on our model, the tuned rule system, the Gemini systems) read the
  matched situation text.
- Late fusion, the label reader and the fine-tuned Whisper model: features derived from the detector are
  recomputed from the matched detector output; text features use version 2's transcripts; features computed
  from the audio (detector embeddings, VoxTurn's answers, the Whisper encoding) stay each version's own.
  VoxTurn's questions use version 2's transcript of the line with each version's own audio.
- Our model is run again for every condition with its frozen weights and thresholds.
- The Gemini readers are asked once per group and condition (their inputs are identical across versions);
  Gemini listening is asked for every version.

### 10.4 Input-identity check and numerical noise floor

- Before scoring, within each group and condition, all situation texts must be identical strings and all
  non-audio features bit-identical. Failing groups are dropped and counted; if more than 2% fail, the test
  is invalid and only the reason is reported. The report code enforces this; it was rehearsed on a
  deliberately corrupted dev copy.
- In this test local models score one window at a time (no batching effects). The natural inputs are
  re-run the same way, and the maximum score difference and the number of flipped decisions against the
  main run (which uses batches of 8, section 12.2) are reported.
- API systems: about 300 questions per system (34 per condition) are asked a second time and the flip rate
  is reported.

### 10.5 Measures

- **Grounded group accuracy** (at frozen thresholds): a group counts only if every version is right: the
  system reacts to the real sound and does not react without it, whatever the label says. One value per
  condition; the summary is the mean over conditions (respond 6, stop 4).
- **Within-group paired concordance** (no threshold): in each group and condition the event version is
  paired with each other version (respond: 1 with 2 and 1 with 3; stop: 1 with 2, 3 and 4). The event score
  is the probability of "do not respond" or of "stop". A pair scores 1 if the event version scores higher,
  0.5 if they are equal (difference below 0.001), 0 if lower. A system that reads only text scores exactly
  0.5. The same is computed at the decision level (decisions instead of scores; equal decisions count 0.5);
  statements about decisions cite the decision-level value.
- **Event-score difference:** the event version's score minus each other version's, averaged over
  conditions; it shows the size of the effect, which concordance does not.
- **Follow curves:** the share of event versions and of no-event versions treated as "the event happened",
  by label confidence. Sound effect = the gap between the two curves at one confidence; label effect = the
  change of one curve from 0.3 to 0.99; sound effect without labels = the gap in the "none" condition, which
  separates "cannot hear" from "hears, but a label overrides it".
- Reported separately: phone and door groups; presence pairs (1 with 2) and identity pairs (1 with 3).

### 10.6 Read-outs compared

All read our frozen fine-tuned model; they differ in what they read and in their training labels.

| Read-out | Reads | Training labels | Role |
|---|---|---|---|
| Original read-out | last-layer states | natural | the status quo |
| Natural-label control (mean of 8 random starts) | last-layer states | natural | ladder rung 0 |
| Step 2, states only | last-layer states | ring-type events independent of the truth | ladder rung 1 |
| Step 2b, states only | last-layer states | all five families independent; false labels timed as in "respond" scenes | label-timing comparison |
| Step 2c, states only | last-layer states | as step 2b, false labels copied from the timing of real labels | ladder rung 2 |
| Step 2d (ours) | states + per-second audio summary | as step 2c | main system |
| Step 2c with the audio summary | states + whole-window audio summary | as step 2c | comparison for the time criteria |
| Shuffled-seconds read-out | as step 2d, with the 8 seconds in random order | as step 2c | comparison for G4c |
| Natural-label read-outs (8 seeds each) | as step 2d; as step 2c with the summary | natural | the 2×2 of label reliability × time detail |
| Step 2 and 2b with the audio summary; conditional read-outs | — | — | reported |
| Audio-encoder probe | whole-window audio summary | true events | does the encoder hear the event? |

The control is the mean of 8 fixed random starts because, with natural labels, nothing in training
constrains the direction "sound beyond the label", so a single fit depends on its random start.

### 10.7 Criteria

Unless stated otherwise, a criterion is computed on "respond, hearing" over its six conditions, with
score-level concordance. The suffix "-S" marks the same criterion on "stop, movement" over its four
conditions. The bars are our own choices, without a literature value: 0.20 is one fifth of the
concordance range from 0.5 (sound unused) to 1 (sound fully used); 0.10 is used for differences of grounded
group accuracy; 5 points is the decision table's win margin.

- **Ours is grounded (G1a, G1a-S):** ours' grounded group accuracy has a lower interval bound above 0.5.
- **Ours is more grounded than Gemini listening (G1b):** ours' decision-level concordance minus that of
  Gemini listening (the more grounded of its two settings, with and without the reliability table) is at
  least 0.20, with the interval above 0. The stop version is reported. Against the text-only rivals this
  holds by construction and is reported as a construction check, not as evidence.
- **The information is there, and the original read-out does not use it (G2a, G2a-S):** (step 2c states-only
  concordance - 0.5) - |original read-out concordance - 0.5| is at least 0.20, interval above 0.
- **Use depends on label reliability in training (G2b, G2b-S):** step 2c states-only concordance minus the
  natural-label control's is at least 0.20, interval above 0.
- **Rung 1 is about presence (G3a),** on presence pairs: relative, step 2 states-only minus the control at
  least 0.20 with the interval above 0; absolute, its own concordance minus 0.5 at least 0.20 with the
  interval above 0 (so that a low control cannot hand out the pass).
- **Rung 2 is about identity (G3b),** on identity pairs (ring against decoy): relative, step 2c minus step 2
  (states only) at least 0.20, interval above 0; absolute, step 2c's own concordance minus 0.5 at least 0.20,
  interval above 0.
- **Side conditions** (needed only for the words "and only on that judgement"): rung 1's rise on identity
  pairs below 0.20, and rung 2's drop on presence pairs below 0.20 (point estimates). They were written after
  the dev results were known; the symmetric version is reported too.
- **Label timing (G3c, described only):** step 2c minus step 2b (states only), decision-level concordance on
  "stop, movement".
- **Time detail (G4a):** ours minus step 2c with the whole-window summary, stop grounded group accuracy, at
  least +0.10 with the interval above 0. Both use the same training data, so the difference comes only from
  the per-second information.
- **Time detail with a moved decision point (G4b):** the same difference after moving the stop decision point
  to 0.5, 1.5 and 2.0 s after the door opens (instead of 1.0 s): test windows re-rendered with the frozen
  generator (the 1.0 s window must equal the stored one sample by sample), "none" condition, one window at a
  time; the mean over the three offsets at least +0.10 with the interval above 0, without the groups
  dropped by the identity check.
- **The order of the seconds matters (G4c):** ours minus the shuffled-seconds read-out, stop grounded group
  accuracy, at least +0.10 with the interval above 0.
- **Noise (G5):** section 11.
- **Label reliability matters with a direct audio path too (G6a, G6a-S):** ours minus the natural-label
  per-second read-out (mean of 8 seeds), grounded group accuracy, at least 0.20 with the interval above 0.
- **Appropriate reliance (G7a, G7b):** on natural "respond, hearing" points whose window holds an event
  detection of confidence at least 0.2, "follow" means do not respond, and the label is "true" when a real
  event is in the window. Within strata of confidence band (0.2-0.5, 0.5-0.7, 0.7-0.9, 0.9-0.99, above 0.99)
  × event class, the follow rate given a true label minus the follow rate given a false label is combined
  with Cochran-Mantel-Haenszel weights n_true × n_false / (n_true + n_false). G7a: ours' combined gap has a
  lower bound above 0. G7b: ours' gap minus the label reader's has a lower bound above 0. Both must pass
  before "appropriate reliance" is stated as a result.

### 10.8 Intervals, ties and several criteria

- Every criterion and reported row of this test uses point ± t(0.975, G - 1) × SE, where SE is the
  three-way combination of section 7.2 and G is the number of clusters of the factor with the largest
  one-way SE among the units involved. Cluster counts, the deciding factor and the normal interval are
  reported beside it. The t rule was added because the normal rule is too narrow with few clusters (dev
  "stop, movement" has 5 door-recording groups): in a simulation on that structure with a true rate of
  exactly 0.5, the t rule passed falsely in 1.5% of runs and the normal rule in 5.5% (nominal 2.5%).
- For display, proportions are clipped to [0, 1] and differences to [-1, 1]; criteria use the unclipped
  bounds.
- Bootstrap B = 2,000 with seed 1. Seeds 2 to 6 are a reported check; if they disagree, the result is
  called "within resampling error" and all six lower bounds are given.
- The two central statements are intersection tests: "label reliability decides whether the sound is used"
  needs G2b, G3a and G3b; "time detail matters" needs G4a and G4b (and G4c for the stronger wording). Each
  criterion is judged at 5%, which keeps each statement at 5% without correction. G1a, G1a-S, G1b, G2a,
  G2a-S, G2b-S and G5 each support one sentence and are judged at 5% each, without correction (stated with
  the results).

### 10.9 Other measurements

- **The words channel** (from the main run): on the items where the user's words decide (reaction parts,
  floor checks, episode points), two error types are counted: "following the event" (the words show the
  user is still with the assistant or wants it to continue, but the system acts as if the user left) and
  "ignoring the event" (the user turned to someone else, but the system acts as if not). These items have
  no no-event controls, so error rates are compared only between systems on the same items, with the
  intervals of section 7.
- **Natural front end** (main run, main body only): "respond, hearing" and "stop, movement" points split by
  what the detector did (event present with a label of at least 0.9, 0.5 to 0.9, 0.2 to 0.5, or missed; no
  event with a false label of at least 0.5, 0.2 to 0.5, or clean), by whether the speaker tag was right, and
  by background-noise recording; the hard subset (labels of 0.5 to 0.9) and the vibration subset
  separately.
- **Trust against precision:** for each system and confidence band, the share of decisions that act on the
  label, beside the detector's real precision in that band (measured on the test split) and the best
  label-only rule (follow where precision is above 0.5). Text readers that deviate from that rule over- or
  under-follow; a listening system can do better by following only the true labels. Bands carry intervals.
- **Inside our model:** concordance of every read-out and of the audio-encoder probe; AUC on the hard natural
  subset; in conflicts between a confident label (at least 0.9) and the probe (heard at probability 0.5 or
  more), whose side the decision takes.

### 10.10 Further fixed report rows

Reported with intervals but without a pass mark:
- **Perception gap:** the reference with perfect labels minus the strong Gemini reader on the five-part
  mean, both at the same setting (48 examples, no thinking). In answers mode two versions are reported (that
  same setting, taken from the best-setting-on-test run, and the strong reader's own answers setting);
  statements use the smaller one. The only other difference is that the perfect-label prompt has no
  reliability table.
- **Natural comparisons:** on the hard subset (respond points with labels of confidence 0.5 to 0.9), ours
  minus the original read-out, minus step 2 with the audio summary and minus each Gemini system; among
  no-event respond points with labels below 0.5, the share where ours and the original read-out act on an
  event that did not happen; ours minus the original read-out on the five-part mean and the sound-only
  part; the original read-out minus the label reader on the five-part mean.
- **Gemini listening, "hears but the label overrides it":** its sound effect without labels minus its mean
  sound effect at confidence 0.9 and above (respond: 0.9, 0.99; stop: the movement labels at 0.9, 0.99).
  Only if this is at least 0.20 with the interval above 0 may we say that it hears but confident labels
  override what it hears.
- **Others:** stop grounding over a grid of stop thresholds (0.05 to 0.60); grounding at confidence 0.9 and
  above, by phone and door and by door source; the stop table of every system; the ladder of the audio-summary
  read-outs; the 12 other read-out candidates on the five-part mean and the sound-only part; the 2×2 cells
  and second random starts; reliance tables for stop and at each noise level; the Gemini readers against
  their own reliability table; the Gemini readers' errors split by the front-end part that was wrong
  (transcript, speaker tag, detector label) and into "perception cost" and "reliance error".

### 10.11 Reporting rules

Statements on the test split are limited by these rules (dev numbers are background only):
- "Grounded" for a decision only if its G1a criterion passes; otherwise "partly grounded", with the
  numbers. Never an unqualified "both decisions are grounded"; the two contrast families are always named
  (respond: ring, nothing, decoy; stop: leaving, staying, cupboard, TV).
- For the original read-out only "its decisions did not change with the audio", and only if its
  decision-level concordance is between 0.45 and 0.55 and every per-condition gap is below 0.05; otherwise
  the direction of the change is stated.
- "Keeps natural accuracy" only if the lower bound of ours minus the original read-out is above -3 points;
  "better" only if the interval is above 0.
- Text readers "inherit front-end errors"; "right for the wrong reason" is used only for systems that
  receive the audio.
- A criterion that fails on the test split is reported as "not reproduced on the test set", never replaced
  by its dev value.

## 11. Noise analysis

Question: when the front end gets worse, do the label-reading systems lose more than ours?
- After the main run, the main body of "respond, hearing" and "stop, movement" (all versions) was
  re-rendered with the frozen generator with background noise 5 and 10 dB louder (all other sources, times
  and seeds unchanged); the clean level was re-run through the same path.
- The frozen detector, Whisper models and speaker model ran again on the new audio. Nothing was re-tuned:
  the speaker threshold, the Gemini readers' reliability table and every setting and threshold stayed
  frozen.
- Systems: ours, the original read-out, step 2b and step 2c with the whole-window audio summary, the tuned
  rule system, the label reader, late fusion, the fine-tuned Whisper model, both Gemini readers and Gemini
  listening (same scoring mode as their main run; clean and noisy questions asked in the same session).
- **Criterion (G5):** at +10 dB, how much the gap "ours minus a label-reading system" grows relative to the
  clean level, averaged over the four label-reading systems (original read-out, tuned rule system, both
  Gemini readers) and then equally over the two parts; at least +5 points with the t interval (section 10.8)
  above 0. It was written after a dev pilot, with the decision table's win margin as the bar.
- Reported without a bar: the +5 dB level; each system and part; Gemini listening (it hears, so it is not
  counted as a label reader); the label reader, late fusion and the Whisper model; detector recall, false
  alarms and precision by confidence at each level; the trust tables at each level; sensitivity rows (label
  reader added, original read-out removed, each noise recording alone and left out, five-factor interval).
  A missing file at any level stops the report.
- Stated limit: only louder background noise was tested; far field, extra reverberation and unseen sounds
  were not.

## 12. Freeze, rehearsal and the single test run

### 12.1 Code required before the freeze

All test-day code had to exist and be rehearsed on dev: the test front end; the main run with every
read-out candidate; the decision report reading every setting from the freeze record, with the answer-loss
adjustment; the grounding report with subset filtering, the identity rule and the decision-level
comparison with Gemini listening; API commands with the answers-only switch and frozen settings; no test
metric printed while running (the rule system's script used to print one); a test lock that only the final
run can open; the moved-decision-point check; the extra report rows; the noise analysis.

### 12.2 Batching

The main run scores every test window in batches of 8, in the same order as on dev, because the read-outs'
thresholds were chosen on batch-of-8 features. Automatic checks: the original read-out's scores from this
pass must equal those of its own scorer bit for bit, and the whole-window audio summary rebuilt from the
per-second values must match the stored one within 0.001. The grounding test scores one window at a time
(section 10.4).

### 12.3 Freeze record

A script writes `results/main_v04/freeze.json` with:
- the sha256 and size of every file the final run reads (193 files): the three rule documents, every script
  (found by following imports), model weights (adapter, heads, detector, read-outs, controls, probe), the
  rule system's chosen setting and phrase lists, the Gemini prompt, note, reliability table and 48 examples,
  the dev results and calibrations the final scripts read, the drift-check references, asset tables and
  downloaded model files;
- digests of two large folders (the TTS cache and the prompt examples' audio);
- the frozen settings that the scripts read instead of hard-coding them: the main system and its
  thresholds, every system's setting, both API modes with settings, losses and loss SEs, drift limits, the
  decision-table constants, the grounding bars, the moved-point offsets, the noise levels; external model
  names and versions; software and GPU versions.

It can be written only from a clean working tree, is never overwritten and must have no open fields. Every
final command checks all hashes first and stops on any mismatch. At the freeze, the sha256 of the record and
of the three rule documents was logged.

### 12.4 Rehearsal on the development set

The freeze required a full rehearsal of the test-day code on dev, reading a dev copy of the record: scores
within 0.001 of the stored ones and identical decisions (no dev metric changes); the original read-out
reproduced bit for bit; the moved-point check and the noise pilot reproduced; the decision table reproduced
in both API modes. A changed dev copy (some units relabelled as unseen subsets, some inputs corrupted)
confirmed that subsets stay out of the criteria and that corrupted groups are dropped, with more than 2%
making the test invalid.

### 12.5 Test lock and seals

- The loaders open the test and replication splits only when the freeze record exists and the command sets
  `P1V4_FINAL_RUN=1` (set for each command, never for a whole session).
- Each sealed split has a seal: the sha256 of its sorted window hashes, printed when it was rendered. When a
  sealed split opens, the freeze record is verified once per process and the seal recomputed; a separate
  command re-hashes every window file.
- Speaker enrolment audio for the test and replication voices was synthesised before the freeze from the
  voice lists alone, without reading any test data.

### 12.6 Test-day order

0. Freeze after the rehearsal passes; log the hashes.
1. Before opening: hash check, probe, drift checks (re-choice if needed).
2. Unlock and verify the seal.
3. Test front end.
4. Local systems, one process each (the read-outs need the fine-tuned model's and the Whisper model's
   scores first).
5. API systems, and the best-setting-on-test questions (same day, before any metric, because the
   perception gap reads them).
6. Grounding runs (with the API repeat samples and Gemini listening's second setting).
7. Noise analysis at its three levels.
8. Completeness check: no missing file or answer, every best-on-test candidate present, repeat samples
   present, scoring modes consistent, every mode switch logged.
9. Metrics, in this order: moved-point check, audio-encoder time diagnostic, noise report, decision
   report, grounding report.

Had the table landed in row 4, the replication split would have been run once through the same steps.

### 12.7 Bug fixes after the freeze

Only bug fixes are allowed: re-run the affected dev rehearsal (outputs must be identical), re-hash only the
fixed files with the `amend` command (it asserts that no frozen setting changes and keeps old and new
hashes) and log the reason. The final record holds two such entries:
- On the first test day, the local LLM reader's cache check (on the first 3 questions of each K, scores from
  the cached prompt prefix against an uncached pass) stopped the run at K = 24, with a difference of 0.056
  against a limit of 0.05. Dev-only diagnostics showed that both computations are valid and differ only by
  rounding (4-bit weights, bf16, different split points; up to 0.093 on dev), so the author set the limit
  to 0.15. The check decides only whether a run goes on; scores do not change. The dev run from scratch was
  byte-identical to the frozen one, and the test run continued.
- The new use of the replication split (section 13), with no frozen setting changed.

## 13. Reliability checks on the replication split

### 13.1 Background and why the split was available

- After the test, a further question was studied: does ours know when it cannot hear? First descriptively,
  on the stored outputs of the noise analysis (how confidence and confident errors change with noise), then
  on dev only, in steps whose criteria were written before each run (summarised in
  [iterations.md](iterations.md)): step 1 crossed labels (none, false, true) with background noise up to
  40 dB louder; steps 2 to 5 tried five versions of a fix. No frozen file, setting or threshold changed.
- The replication split had been sealed for one purpose: a single re-check if the decision table landed in
  row 4. On the test split the table landed in row 1, so that re-check was not needed. A further amendment,
  recorded in the freeze record without changing any frozen setting, made the split the confirmation set
  for the dev findings. The registration (frozen hashes, claim list, reading rules, run order) was committed
  before the split was opened, and the split was run once. Only local systems run (no API rival), on the
  main body only.

### 13.2 What was frozen

Nothing is fitted or chosen on the replication split. Frozen systems: ours (stop 0.139, respond 0.248); the
original read-out (0.286, 0.049); two confidence-aware versions developed on dev, each with its thresholds:
version 4 (a small confidence model using the score's distance to the threshold, the chosen side and an
audio-clarity probe, plus a read-out that takes structured device notifications: an incoming call, a
doorbell press, a front-door sensor) and version 5 (adds how much the score changes when labels are
removed, a classifier on the user's last line, and a piecewise clarity term); version 5's control that uses
only the label-free score; the clarity probe. A decision is "confident" when the chosen answer's
probability is at least 0.9. The hashes of every script, model and threshold file the check reads are kept
in its own freeze file, which is never overwritten.

### 13.3 Data and conditions

- Groups: the matched groups of section 10.1 on the replication split (respond: 3 versions; stop: 4
  versions, visitor silent), main body only.
- Front end: the first step of the frozen test-day front end, as on the test day.
- Sound: clean, and background noise 10, 20, 30 and 40 dB louder (frozen generator); transcripts stay those
  of the clean audio, so only the sound degrades.
- Seven input conditions (the letters are the condition ids used in the code); false event labels are added
  only to respond groups, false movement labels only to stop groups:

| Condition | Device notification | Detector label |
|---|---|---|
| A | on | event labels removed |
| B | on | false label, confidence 0.99 |
| B05 | on | false label, confidence 0.5 |
| C | unavailable | false label, confidence 0.99 |
| C05 | unavailable | false label, confidence 0.5 |
| D | unavailable | event labels removed |
| T | unavailable | true label, confidence 0.99 |

### 13.4 Fixed claims

A script computed every candidate claim on dev; all 59 whose dev interval lay entirely on the passing side
were registered, none picked by hand; 19 of them are key claims. Groups:
- (M1) with correct labels and masked sound, ours is more than 5 points below the original read-out
  (respond and stop at +20, +30 and +40 dB);
- (M2) more than 10% of ours' confident decisions are wrong (no label or false labels; +30 and +40 dB);
- (V4a, V4b) without labels, version 4 makes fewer confident errors than ours, but its AUROC for telling
  right from wrong decisions is within ±0.03 of ours;
- (V5s) version 5's AUROC is above ours with false labels and below ours with true labels;
- (TD1, TD2) trust trade-offs: version 5 minus version 4, and the label-free control minus version 5, are
  positive with false labels and negative with true labels;
- (V5n) with notifications on, accuracy on notified events above 95%;
- (V5o) in clean sound, version 5's share of confident decisions at most 10 points below ours;
- (V5k) with notifications on, knocks (which send no notification) under masked sound give more than 10%
  wrong confident decisions (a stated limitation).

How to read them, stated in advance: some claims hold partly by construction (version 5 reads a
label-free score, so under false labels it moves toward the no-label cells: it "does not count labels"
rather than "detects false labels"); the notification claims are close to a rule; the M1 claims measure
how far ours falls when its hearing is damaged; the trade-off claims also confirm a cost on true labels.

### 13.5 Reading rules and run order

- Intervals: 1,000 bootstrap resamples of conversation units, 95% percentile interval (the same kind used to
  select the claims on dev); statistics pooled over noise levels resample all levels together.
- Each claim on its own is confirmed (interval entirely on the passing side), uncertain (point passes,
  interval crosses the bar), not confirmed (point fails, interval crosses), overturned (interval entirely
  on the failing side) or not evaluable (fewer than 30 confident decisions, or no interval).
- The bars (5 points, 10%, 95%, ±0.03) are our own; ±0.03 was set after seeing the dev value. There is no
  overall pass verdict and no count over all 59 claims as a conclusion; the 19 key claims and each group
  are summarised separately. Claims that only just passed on dev are expected to fall back to "uncertain".
- Sensitivity rows (report only): Bonferroni-corrected intervals for the 19 key claims (level 1 - 0.05/19,
  10,000 resamples) and multi-factor intervals (section 13.6).
- Run order: on dev, the evaluation, the text check and a rehearsal of the renders and all seven inputs had
  to reproduce the stored results exactly; then the claims were registered, the amendment recorded in the
  freeze record, the check's own freeze file written, and the freeze check run (no change allowed). Only
  then, with the test lock opened per command: front end, groups, renders, forward passes and evaluation,
  run once. Automatic checks: every level complete; every stored output matches the current groups and
  texts by hash; the five levels hold the same windows; the noisy audio summaries differ from the clean
  ones. No intermediate result was looked at.
- Stated limits: clean transcripts at every level; the common flip lines are not new phrasings; version 5 is
  the fifth round on the same dev set; natural (unmatched) inputs are not part of this check.

### 13.6 Report-only and descriptive rows

- **Multi-factor intervals** for the 19 key claims, computed after the main result from the stored outputs:
  five cluster factors (conversation, event recording, second recording, background-noise recording, user
  voice), Poisson(1) weights, B = 2,000, seed 1, combined by inclusion and exclusion and floored at the
  largest one-way variance, with the t interval of section 10.8; verdicts by the same reading rule, report
  only.
- **Mirror against hearing damage** (descriptive; design written down before computing). For each noise
  type, level and decision: hearing damage x = ours' accuracy without labels at the clean level minus at
  that level; mirror y = ours minus the original read-out, both with true labels (the M1 statistic); label
  help u = ours with true labels minus ours without. Since y = (clean gap) - x + u by construction, the
  slope of y on x and the size of u carry the information, not the correlation.

## 14. Check with new noise types

- **Purpose:** the confirmation's background noise came from only 7 environments of one corpus (DEMAND),
  the largest source of uncertainty there. Do its relations hold for noise types the systems never heard?
- **Status:** a labelled check, not a confirmation, written down before it was run. It adds no criterion;
  results are described as "holds", "uncertain" or "does not hold", and the confirmation's conclusions do
  not change.
- **Data:** the same replication units as section 13 (main body of "respond, hearing" and "stop, movement",
  all versions). Conversations, events, decoys, voices, rooms and signal-to-noise settings are unchanged;
  only the background-noise recording is replaced, so the results compare directly with section 13.
- **Noise:** MS-SNSD at a pinned commit, only its CC0 Freesound files, four appliance types: air
  conditioner, vacuum cleaner, washer or dryer, copy machine. Left out: noise with speech (babble,
  announcements, neighbours: speech raises the question of who is talking to whom), sounds like our events
  (a door closing, a squeaking chair) or our decoys (typing, munching), and MS-SNSD files that come from
  DEMAND (our own noise corpus).
- **Screening, changed before any system output was seen:** the rule fixed in advance (drop a file whose
  detector frame probability reaches 0.5 for any target class) would have dropped 45 of 48 files, because
  the detector, trained with DEMAND noise only, fires on unfamiliar appliance noise. The labels in this check
  are injected by condition, so detector output does not enter the results; only near-identical files were
  dropped (one pair), and the detector's probabilities are recorded per file. Side finding: confident false
  labels also occur naturally on unfamiliar noise.
- **Rendering:** each type's files are set to the same loudness and joined with 1 s crossfades into one
  track; each unit gets a type (round robin over a seeded order, balanced per part) and a seeded start in
  that track (no looping), at its own signal-to-noise ratio and then 10, 20, 30 and 40 dB louder, rendered
  with the frozen generator.
- **Systems, conditions and statistics:** exactly those of section 13, with the code frozen there; the
  unit's own noise level plays the role of "clean".
- **What is looked at:** first, the 16 new cells (4 types × 4 louder levels) added to the mirror against
  hearing damage: do they fall near the earlier lines (slopes of about -0.8 to -0.9)? Reported plainly if a
  type damages hearing without a mirror, or gives a mirror without damage, and the slope fitted to the new
  cells. Second, the 19 key and 59 registered claims under the same reading rule, also per noise type
  (report only).
- **Limits:** all four types are steady appliance sounds; speech, music and sudden sounds were not tested.

## 15. Earlier task version and dated amendments

### 15.1 Version 0.3 sections still cited

The first version of the task had one decision (respond) and cells with three versions (event, no event,
decoy). Its data scripts and three of its other files are still used, so the code cites its plan:
- **Section 3:** that version's behaviour rules and the two-sided line check. A flip line passes only if,
  after an event, it is clearly for the caller or visitor and, without an event, natural to say to the
  assistant; requests to wait ("Just a second!") never pass, because after an event they fit the visitor
  and the assistant alike. A "hold on" line must clearly be for the assistant; a decoy sound may precede a
  flip line only if the line then clearly goes to the assistant. Each main cell needed at least 5 distinct
  passing flip lines, otherwise it became report-only.
- **Sections 3 to 4:** the item renderer and its data rules (versions sample-identical outside the event and
  its reverberation tail; levels relative to the user's speech; decoys generated exactly like events, from
  household sounds that are neither ring-like nor knock-like, not human and not alarming).
- **Section 4 (data):** conversations generated with fixed prompts, deduplicated and split by conversation
  (300 / 100 / 600 / 600); lines normalised and deduplicated across splits, flip lines common to all splits
  with a held-out test-only set; voices split by voice, plus a second TTS engine for the test split; event
  and decoy recordings from FSD50K (CC0 or CC BY only), grouped by uploader and by "same sound" before
  splitting, MUSAN matches removed, with minimum counts for test (at least 40 clips from 25 uploaders per
  event class and for the decoys) and replication (20 from 12), filled in that order; recordings uploaded
  to Freesound after June 2023 for test and replication only; rooms and noise split by room and by source.
- **Amendment A1 to version 0.3:** phone and doorbell groups were too few for the minimums, so they were
  split in fixed proportions of the groups (test 30%, replication 15%, dev 10%, train 45%; seeded), with the
  actual numbers reported; knock and decoy groups kept the minimums. Version 0.4's new classes reuse these
  proportions.
- **Section 5.1:** the fine-tuned speech model: VoxTurn with a new LoRA adapter answering one new question in
  VoxTurn's own format.
- **Section 6.4:** the unseen subsets (test-only phrasings, another TTS engine, unseen event types, new
  recordings) are reported separately and enter the decision table together, the rule kept in section 3.9.
- **Section 8 (its step 0 list):** item 1, our own forward pass must reproduce VoxTurn's official outputs
  (probability differences below 0.001 on 20 clips) and the new adapter must be able to overfit 64
  samples; item 6, the recordings (FSD50K counts and licences, grouping with a calibrated fingerprint
  threshold, MUSAN comparison, minimum counts, decoy classes, lengths and cropping); item 7, the post-2023
  Freesound recordings and the sources of the unseen event types; items 8 and 9, voices and the second TTS
  engine, rooms and noise (sources, licences, splits); item 11, line pools generated, checked both ways
  before splitting, normalised, deduplicated and assigned to splits.

### 15.2 Dated amendments

Some smaller changes were logged with a date and a reason before any affected result was used. Two are
cited by date in the code:
- 2026-11-14: the episode checker changed from Claude Opus 5.5 to Claude Sonnet 5.5 after the first 45
  batches (the model of every batch is recorded);
- 2026-11-15: the detector's planned "family member calling" class was dropped, because the training split
  has no such calls (they occur only in report-only cells); other people's speech reaches every system
  through transcripts and speaker tags.

Others are described in the code where they apply (for example the denser threshold grid, three settings
added to the rule system, and VoxTurn's text input limited to the line transcript).

## 16. References from the code

Comments in `src/` and `configs/` cite the rule documents by section. The table lists every distinct
reference and the heading of this file that explains it. A few comments in `p1v4_grounding.py` say "plan"
for the grounding package (its file was also called a plan); those are listed separately.

| Reference in the code | Explained in |
|---|---|
| plan section 2 | 1.1 The two decisions |
| plan 3.1 | 1.2 What every system receives |
| plan 3.2 | 1.3 Correct behaviour |
| plan sections 3-5 | 1 Task and decision points; 2 Data and splits; 3 Test parts and scoring |
| plan 4 | 3 Test parts and scoring |
| plan 4.2 (`p1v4_units.py`) | 3.2 Respond, reaction |
| plan 4.5, plan section 4.5 | 3.5 Episodes |
| "P1 v0.4, sections 4.4 and 5.2", "sections 4.4-4.5, 5.5" (TV speech, source positions) | 3.4 Stop, movement; 3.5 Episodes; 2.1 Splits and their sources; 2.3 Rendering |
| plan 4.6 | 3.7 Floor checks |
| plan 4.7 | 3.8 Sound-only part |
| plan 4.8 | 3.9 Unseen subsets and report-only cells |
| plan 4.9 | 3.9 Unseen subsets and report-only cells |
| plan 5.2 (including "relaxed"), "P1 v0.4, section 5.2" | 2.1 Splits and their sources |
| plan 5.3 | 2.2 Lines and episode scripts |
| plan 5.4 | 2.2 Lines and episode scripts |
| plan 5.5, "P1 v0.4, section 5.5" | 2.3 Rendering |
| plan 5.6 | 2.1 Splits and their sources (caps); 2.4 Sizes |
| plan 6.1 | 5.1 Shared front end |
| plan 6.2 | 5.2 Trained candidates and the main system |
| plan 6.3 | 5.3 Attribution controls |
| plan 6.4 | 5.4 Rivals; 6.2 Equal tuning of the rivals |
| plan 6.5 | 5.5 References |
| plan 7.1; "the plan's rule", "the plan's operating-point rule" | 6.1 Operating-point rule |
| plan 7.1 (intervals, calibration); "the plan's interval", "the plan's three-way bootstrap", "the plan's 3 factors" | 7.1 Clustering factors; 7.2 Combining one-way cluster bootstraps; 7.3 Calibration and precision estimate |
| plan 7.1 (unseen part) | 3.9 Unseen subsets and report-only cells |
| plan 7.1-7.5 | 3.6 Group scoring and the five-part mean; 6 Thresholds and settings; 7 Intervals; 8 Main comparison and decision rule |
| plan 7.2 (`p1v4_test_report.py`, `p1v4_dev_report.py`) | 8.1 Main comparison |
| plan 7.3 (listen step) | 8.2 Listen step |
| plan 7.4; "the plan's decision table"; rows 1 to 6 | 8.3 Decision table |
| readings R1 to R12 | 8.4 How the code reads the table |
| plan 7.5 | 6.2 Equal tuning of the rivals (best setting on test); 6.3 Development checkpoint and basic checks; 8.5 Rows reported beside the table |
| plan 8.1 step 3 | 6.3 Development checkpoint and basic checks |
| plan 8.1 step 4 | 12.3 Freeze record |
| plan 8.1 step 5 | 12.6 Test-day order |
| plan 8.2, 8.3, 8.4, 8.6 | 4 Leakage controls |
| "plan section 10 risks, with item 3" (`step0v4_speaker_overlap.py`) | 4 Leakage controls (external models); 2.5 Data checklist ("step 0") |
| plan 8.5 | 4 Leakage controls; 12.5 Test lock and seals |
| step 0 items of version 0.4 ("P1 v0.4": items 1 to 6, 10, 12; "step 0 v0.4 item 1") | 2.5 Data checklist ("step 0") |
| version 0.3 plan ("P1 v0.3", "v0.3 plan"): sections 3, 3-4, 4, 5.1, 6.4 and 8 ("section 8 item 6"); its step 0 items 1, 6, 7, 8-9 and 11; "plan section 3" in `step0_check_batches.py` and `step0_assign_lines.py`; "Frozen rule (v0.3)" | 15.1 Version 0.3 sections still cited |
| "amendment A1 of v0.3" | 15.1 Version 0.3 sections still cited; 2.1 Splits and their sources |
| amendment 1.1, 1.2 (main system) | 5.2 Trained candidates and the main system |
| amendment 1.4 (other candidates) | 5.2 Trained candidates and the main system; 10.10 Further fixed report rows |
| amendment 2.2 (steps 2, 4, 6), 2.2-2.4, 2.2-2.6 | 9.1 Probabilities or answers; 9.2 Order before the test split is opened; 9.3 Drift checks and re-choice |
| amendment 2.3 (including "first row") | 9.4 Incidents during the run |
| amendment 2.4 | 9.1 Probabilities or answers |
| amendment 2.5 | 9.3 Drift checks and re-choice |
| amendment 2.6 | 9.5 Compensating answers-only scoring; 6.2 Equal tuning of the rivals (reliability table for Gemini listening) |
| amendment section 3, amendment 3.3 | 10 Matched grounding test (3.3: 10.7 Criteria, side conditions) |
| amendment 4.1 | 12.2 Batching |
| amendment 4.2 (items 1, 2, 3, 5, 6) | 12.1 Code required before the freeze |
| amendment 4.3 | 12.3 Freeze record |
| amendment 4.5 | 12.4 Rehearsal on the development set |
| amendment 11, 11.1 | 10.10 Further fixed report rows (perception gap) |
| amendment 11.2 | 10.10 Further fixed report rows (natural comparisons) |
| amendment 11.3 | 10.10 Further fixed report rows (Gemini listening) |
| amendment 12, amendment section 12 | 11 Noise analysis |
| amendment 13, amendment section 13 | 7.4 Variants; 9 Gemini systems on the test days; 12 Freeze, rehearsal and the single test run |
| amendment 13.1 | 9.5 Compensating answers-only scoring; 9.6 Missing answers; 7.4 Variants; 8.4 How the code reads the table (R10, R11) |
| amendment 13.2, "amendment 13.2 supplement" | 9.2 Order before the test split is opened; 9.3 Drift checks and re-choice; 12.3 Freeze record; 12.5 Test lock and seals |
| amendment 13.4 | 9.3 Drift checks and re-choice; 12.6 Test-day order |
| "test-day runbook step 8", "runbook step N" | 12.6 Test-day order |
| amendment section 14 (test-day notes) | 12.7 Bug fixes after the freeze |
| amendment section 15 | 13 Reliability checks on the replication split |
| "registered checks summarised in docs/evaluation_protocol.md"; registration 1 | 13 Reliability checks on the replication split; 13.2 What was frozen |
| registration 3 | 13.4 Fixed claims |
| registration 4, "registration section 4" (reading rule, Bonferroni, "only reported" rows) | 13.5 Reading rules and run order; 13.6 Report-only and descriptive rows |
| "amendment 13 / package 15.7" in `p1v4_reliability_cluster.py` | 13.6 Report-only and descriptive rows |
| the new-noise plan (`p1v4_newnoise.py`) | 14 Check with new noise types |
| "logged amendment, 2026-11-15"; "amendment 2026-11-14" | 15.2 Dated amendments |
| package draft v0.2, v0.3, v0.5, v0.6; "package 15" | 10 Matched grounding test |
| package 4.1 | 10.1 Groups |
| package 4.3 | 10.3 How each system reads the matched inputs |
| package 4.4; measurement 1 | 10.5 Measures |
| package 4.5 | 10.4 Input-identity check and numerical noise floor |
| package 4.6 | 10.4 Input-identity check and numerical noise floor |
| package 5; measurement 2 | 10.9 Other measurements (the words channel) |
| package 6, 6.2; measurement 3 | 10.9 Other measurements (natural front end, trust against precision) |
| package 6.3 | 10.10 Further fixed report rows (natural comparisons) |
| package 6.4 | 11 Noise analysis |
| package 7, 7.3; measurement 4 | 10.6 Read-outs compared; 10.9 Other measurements (inside our model) |
| package 9.2 | 10.7 Criteria; 10.8 Intervals, ties and several criteria |
| package 9.3, 15.9 | 10.11 Reporting rules |
| package 15.1 | 10.2 Conditions |
| package 15.2 | 10.8 Intervals, ties and several criteria |
| package 15.3 | 10.7 Criteria (G4b, G4c); 10.6 Read-outs compared |
| package 15.4 | 10.7 Criteria (G6a); 10.10 Further fixed report rows (2×2, second starts) |
| package 15.5 | 10.7 Criteria (G7a, G7b); 10.10 Further fixed report rows |
| package 15.6 | 10.7 Criteria (G1b); 10.10 Further fixed report rows (Gemini listening) |
| package 15.7 | 11 Noise analysis |
| package 15.8 | 10.10 Further fixed report rows |
| package 15.12 | 10.8 Intervals, ties and several criteria; 10.11 Reporting rules |
| "plan 4.1-4.2", "plan 4.2" in `p1v4_grounding.py` (package sections) | 10.1 Groups; 10.2 Conditions |
| "plan 4.6" in `p1v4_grounding.py` (package section) | 10.4 Input-identity check and numerical noise floor |
| "plan 7.2" in `p1v4_grounding.py` (package section) | 10.6 Read-outs compared |
| "docs/evaluation_protocol.md, section 17" in the `bb2_*.py` scripts; the second-backbone plan ("plan" in the freeze part of `bb2_test.py`, which names the plan's file in its code and in its "not run" notes) | 17 Second backbone (Voxtral Mini 3B) |

Two kinds of references point outside this file. In the dev-only reliability scripts
(`p1v4_reliability_dev.py`, `p1v4_reliability_fix*.py`, the two `*_facts.py` scripts), "plan section 5", "plan
section 8", "plan 1.2", "the plan's four criteria" and "criteria 1-6" refer to the plans of those dev steps,
summarised in [iterations.md](iterations.md) ("plan summarised in docs/iterations.md"). Labels such as "review
M5", "review item 3" or "Review 1" are findings of the independent AI reviews of the plans and the code (see
[ATTRIBUTION.md](../ATTRIBUTION.md)), not rule sections.

## 17. Second backbone (Voxtral Mini 3B)

Every section above uses one pretrained speech model, VoxTurn (built on Qwen3-ASR-0.6B). Does the method
depend on that model? The whole recipe was repeated once on a speech LM from another family: the training
recipe, read-outs, threshold rule, report code and criteria of sections 5 to 12 stayed the same (17.2); only
the model interface changed (17.3). The plan was written on 2026-11-24, before training; its test-run part
on 2026-11-25, before any test data were read for this backbone. "Backbone 1" is VoxTurn, "backbone 2"
Voxtral. Code: the `bb2_*.py` scripts ([code_map.md](code_map.md)); commands: [SETUP.md](../SETUP.md),
section 8.

### 17.1 Model and choice

- **Model:** Voxtral Mini 3B (`mistralai/Voxtral-Mini-3B-2507`, revision 3060fe3, Apache-2.0, about 4.7B
  parameters): a Whisper large-v3 encoder (50 frames per second), a projector that joins 4 frames into one
  audio token (12.5 per second) and the Ministral 3B language model (30 layers, width 3,072).
- **Why:** every part differs from backbone 1 (AuT encoder, Qwen3-0.6B language model), so a result found on
  both is less likely to belong to one model; the transformers version in use supports it natively.
- **Check first, rule written before it ran:** the linear probe that had checked backbone 1's audio encoder
  (phone, doorbell, knock, a similar household sound, no event; 200 training and 80 dev windows per class)
  had to reach a dev accuracy at most 10 points below backbone 1's encoder, otherwise a named fallback model
  would be checked. Voxtral passed. The probe script is not in this repository.

### 17.2 What stayed the same

- Training split, situation texts (most recent 1,024 tokens), questions and phrasings, built by backbone 1's
  code; the fine-tuning recipe of the original read-out (LoRA rank 16, alpha 32, on every linear layer of the
  language model; 2 epochs; micro-batch 1 × 8 accumulation; AdamW 1e-4 without weight decay; 300 warm-up
  steps, then linear decay; clipping 1.0; bf16; one phrasing per sample; epoch kept by section 6.1's rule).
- The decorrelated training labels of steps 2 to 2d (section 5.2) and every read-out with its own fit code:
  ours, the natural-label control and its 8 starts, the other read-outs of section 10.6, the audio-encoder
  probe. Thresholds by the operating-point rule; batching as in sections 10.4 and 12.2.
- The report code and every criterion and bar of sections 8 and 10, the decision table of section 8.3
  included. Rivals and references do not depend on the backbone, so their stored outputs were reused; on
  test the report stops unless every rival number equals backbone 1's test report.

### 17.3 What had to change (interface only)

Each change follows from Voxtral's architecture; none is a new choice.
- **Input:** Voxtral's own chat-template format (start, instruction and audio-start tokens, the audio
  tokens, then the situation text and the question). Its processor pads every input to 30 s, so the 8 s
  window is padded to 30 s: 375 audio tokens, the first 100 of which are the window.
- **Answer head:** Voxtral has none, so the new head (as on backbone 1, one score per option) is trained
  from scratch, starting at zero; on backbone 1 it was copied from VoxTurn's own head.
- **Adapter:** the new LoRA is the only adapter (on backbone 1, VoxTurn's own LoRA stays active under it).
- **Ears:** as on backbone 1, the audio tokens the language model reads, here the projector output (width
  3,072), only the window's 100 tokens; a token belongs to the second that holds its centre, so a second has
  12 or 13 tokens (backbone 1: 13).
- **Attention:** scaled-dot-product instead of eager; the mask that keeps two questions apart is unchanged.

### 17.4 Checks before the run

- CPU, float32: our hand-built input matched Voxtral's native forward pass at the answer positions to
  rounding error; our log-mel equalled Voxtral's feature extractor; both questions in one pass matched two
  separate passes.
- Every input set (training, dev, the ten grounding conditions, the dev noise grid of section 13.1, later
  the test sets) was regenerated by backbone 1's code and matched item by item to backbone 1's stored inputs
  (ids, questions, answers, text hashes where stored); the moved-point sets reuse backbone 1's windows.
- Run on backbone 1's own files, the redirected report code reproduced all 36 thresholds of backbone 1's
  rehearsal record and its moved-point result exactly. Each report lists what it read from backbone 1's
  folders; on test it stops if any file depends on the backbone.
- A GPU smoke run of the test-run code on 64 items of every dev set equalled the stored dev outputs. One
  independent AI review of that code preceded the freeze; its findings were fixed or listed in 17.6.

### 17.5 Order

1. Training and the training and dev forward passes on one 48 GB GPU; then the read-out fits.
2. Dev evaluation with backbone 1's report code (thresholds, grounding test, moved decision points, dev
   noise grid) and two descriptive dev diagnostics of the differences found: does the whole-window audio
   summary keep the decision time, and how much audio information do the answer-option states hold without
   labels?
3. Freeze: a separate record (checkpoint, base-model files, read-outs, settings, dev results, code); its
   settings may differ from backbone 1's only for backbone 2's own model and read-outs. Written once;
   changed only by the amend command of section 12.7.
4. One test run, 2026-11-25: inputs built under the test lock (section 12.5) and checked, forward passes on
   the GPU, then read-out scores, the moved-decision-point check, the decision report and the grounding
   report.

### 17.6 Not run for backbone 2

- The noise analysis of section 11 (G5) and the reliance table that reads it; the reports say "not run".
- Speed. Row 5's practical advantage (section 8.3) reads backbone 1's laptop timings (it matters only for
  local rivals; API rivals always count).
- The audio-encoder time diagnostic on test (step 9 of section 12.6); it ran on dev only.
- The replication-split checks of sections 13 and 14.
- Known difference besides the backbone: the moved-decision-point features are stored in float16 (as on
  dev).
