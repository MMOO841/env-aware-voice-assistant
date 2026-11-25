# Line and script check, prompts v2 (P1 v0.4, sections 5.3-5.4; step 0 items 4-5)

Checker: Claude Opus 5.5 via fresh subagents. Anthropic is not the family of any competing system (B2 and B3 are
Gemini, B2-local is Qwen) or of the generator (OpenAI). Each item is answered once, with these exact questions. The
subagent reads only its batch file and this text, writes only its answer file, never sees system outputs and is never
told which split an item belongs to. The bar is "clearly" (plan 5.3), so every question has an option for "could also
be meant the other way", and only the "clearly" answer passes.

Shared settings, prefixed to the items as stated:
- **S0:** "At home, a person is in the middle of a conversation with a voice assistant (a smart speaker). Right now
  the assistant is not speaking."
- **S1:** "At home, a person's voice assistant (a smart speaker) is in the middle of talking to them, for example
  reading out a recipe or directions."

## Line items (step 0 item 4)

**phone_answer.** "[S0] Their phone starts ringing and they pick it up. Right after picking up, the person says:
«{line}» Question: who is this said to? A = clearly the caller; B = it could also be meant for the voice assistant;
C = clearly the voice assistant." **Pass:** A.

**door_answer.** "[S0] {event}. Before opening the door, the person calls out: «{line}» Question: who is this said to?
A = clearly the person at the door; B = it could also be meant for the voice assistant; C = clearly the voice
assistant." Events: "Someone knocks on the front door" or "The doorbell rings" (the line's generation event).
**Pass:** A.

**keep_going.** "[S1] {event}. The person says: «{line}» Question: what does the person want the assistant to do?
A = clearly to keep talking; B = to stop, pause or wait; C = unclear, or the line is meant for someone else."
Events: "Their phone starts ringing", "Someone knocks on the front door" or "The doorbell rings" (the line's
generation event). **Pass:** A.

**backchannel.** "[S1] {event}. The person does not pick up the phone and does not go to the door, and only says:
«{line}» Question: A = the person is clearly just listening to the assistant and letting it go on; B = the person is
answering the phone or the door, or talking to someone else; C = cannot tell." Every backchannel is asked once per
event (phone, knock, doorbell). **Pass:** A for all three events.

**reject_phone.** "[S0] Earlier the person asked the assistant: «{request}» The assistant answered: «{answer}» Then
the person's phone rang, and the ringing stopped. Someone in the room cannot tell whether the person picked up or
declined the call. Right after the ringing stopped, the person says: «{line}» Question: who is this said to?
A = clearly the voice assistant; B = it could be the caller; C = clearly the caller." **Pass:** A.

**reject_door.** "[S0] Earlier the person asked the assistant: «{request}» The assistant answered: «{answer}» Then
someone knocked on the front door or rang the doorbell, and the person did not go to the door. The person says:
«{line}» Question: who is this said to? A = clearly the voice assistant; B = it could be the person at the door;
C = clearly the person at the door." **Pass:** A.

## Episode items (step 0 item 5; exact strings in scripts/step0v4_episodes.py)

Checker model: Claude Opus 5.5 for the first 45 r0 batches, Claude Sonnet 5.5 for all later batches (amendment
2026-11-14 20:16; per batch in data/episodes_v04/check_models.json). Subagent prompt: read only the batch file, write
only the answer file, run no code, answer every item independently, judge only from what the item shows (the caller
is never heard), "clearly" means a typical listener would not hesitate, borderline items get B; the reply gives counts
only and never quotes or describes an item.

Every item starts with HEAD: "At home, a person uses a voice assistant (a smart speaker). Below is everything someone
in the room could hear so far, in order. If the person is on the phone, the caller cannot be heard." Then the scene up
to the decision point, one line per script turn: `Person: «...»`, `Assistant: «...»`, `Another person (a different
voice, at the front door): «...»`, sounds as `[the phone rings]` (with ", while the assistant is still talking" when
the script marks the overlap), `[the person is quiet for a few seconds, listening]` or `[a short pause]`; at S points
the line `[at this moment the assistant is still talking]`.

- **Question R** (r0, r1): "Question R: who is the person's last line said to? A = clearly the voice assistant; B = it
  could be either; C = clearly someone else (the caller or the person at the door)." Pass: A for respond, C for not.
- **Question S, policy-blind** (r0 only; its answers are not used since option A): "Question S: what should the
  assistant do right now? A = clearly stop talking; B = unclear; C = clearly keep talking."
- **Question S with the household rule** (s1, r1; option A, user 2026-11-14): RULE_S, "Household rule for the
  assistant: while it is talking, it stops when the person answers the phone or the door, silently goes to open the
  door, or says something to the assistant other than a short listening sound or a request to go on; it keeps talking
  when the person does not react to a sound, only makes a short listening sound (like 'mm-hm'), tells it to go on, or
  when voices come from a TV or radio.", then QS_RULE, "Question S: by this rule and from what could be heard so far,
  what should the assistant do right now? A = clearly stop talking; B = unclear; C = clearly keep talking." Pass: A for
  stop, C for continue. Probes without a line add "Nobody has said anything since." before the rule.
- **Ignored-event probes** (s2, and the same points in r1; probe-wording option 1, author 2026-11-14): the scene up to the event, then
  "[a few seconds pass; the person does not react: nobody says anything and nobody moves]" and "[at this moment the
  assistant is still talking]", then RULE_S2 (RULE_S with "it keeps talking when the person does not react to a sound"
  read as "it keeps talking as long as the person has not reacted to a sound, and when the person ...") and QS_RULE.
  Pass: C. s2 answers replace s1 answers for these points.
- **Away lines, text only** (r0_amb, r1_amb): "Question: could a person plausibly say this exact line, on its own, to a
  voice assistant? Line: «...» Y = yes, it could; N = no." Counted for the ambiguous-line share (plan 5.4), not for
  dropping.

A batch never holds two points of one episode (rounds: the k-th point of every episode). Any point answered B or
against the script drops the whole episode (plan 5.4).

## Output format

Line items: one JSON object per item, in the batch's order: `{"id": "...", "answer": "A"}`. Episode items:
`{"id": "...", "R": "A"}` or `{"id": "...", "S": "A"}`; away lines `{"id": "...", "answer": "Y"}`. No free text.
