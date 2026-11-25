# Two-sided line check, prompts v1 (P1 v0.3, section 3; step 0 item 11)

Checker: Claude Opus 5.5 via fresh subagents. Anthropic is not the family of any competing system (B2 and B3 are
Gemini, B2-local is Qwen) or of the generator (OpenAI). Each item is answered once, with these exact questions. The
subagent sees only the batch file and this text, never system outputs.

Shared setting, prefixed to every item:
> At home, a person is in the middle of a conversation with a voice assistant (a smart speaker). Right now the
> assistant is not speaking.

## Flip lines (types "phone_flip" and "door_flip")

Every candidate line is asked in two separate items.

**E item.** "[setting] {event}. Right after that, the person says: «{line}»
Question: who is the person most likely talking to?
A = the voice assistant; B = the caller on the phone, or the person at the door; C = cannot tell."

Events:
- phone_flip: "Their phone starts ringing."
- door_flip: "Someone knocks on the front door, or the doorbell rings."

**N item.** "[setting] Nothing else happens: no phone rings, nobody knocks, no doorbell. The person says: «{line}»
Question: is this a natural thing to say to the voice assistant?
Y = natural; N = odd or unlikely."

**Pass:** the E answer is B and the N answer is Y.

## H-E lines (types "he_door" and "he_phone")

"[setting] {event}. Right after that, the person says: «{line}»
Q1: who is the person most likely talking to?
A = the voice assistant; B = the caller or the person at the door; C = cannot tell.
Q2: does this line only make sense if the assistant is talking at that moment (for example, it asks the assistant to
stop, pause or be quiet)? Y / N."

Events:
- he_door: "Someone knocks on the front door, or the doorbell rings."
- he_phone: "Their phone starts ringing."

**Pass:** Q1 = A and Q2 = N.

## Decoy x line (decoy classes x flip lines that passed)

"[setting] In the room there is a short sound of {decoy description}. Right after that, the person says: «{line}»
Question: who is the person most likely talking to?
A = the voice assistant; B = someone else (a caller, a person at the door, or someone else in the home);
C = cannot tell."

**Pass:** A. A failing combination is never generated.

## Output format

One JSON object per item, `{"id": ..., "answer": "...", "q1": ..., "q2": ...}` as applicable. No free text is needed.
