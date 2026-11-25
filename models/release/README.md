# Detector checkpoint (GitHub Release asset)

The fine-tuned sound-event detector is larger than GitHub's 100 MB limit for a file in a repository, so
it is attached to this repository's GitHub Release instead of being committed.

| File | Size | sha256 |
|---|---|---|
| `sed.pt` | 323,234,382 bytes (308.3 MiB) | `58be2ddacc795e5801bd252ee7c46ba87733c89a8709d759701a5a01ec49d2d3` |

## Download

1. Download the asset `sed.pt` of release `v1.0`:
   <https://github.com/MMOO841/env-aware-voice-assistant/releases/download/v1.0/sed.pt>
   (or open the repository's Releases page).
2. Save it here, as `models/release/sed.pt` (the default path of `src/demo.py`; elsewhere, pass
   `--detector <path>`).
3. Check it. `src/demo.py` compares the sha256 itself and stops on a mismatch; to check by hand:

```bash
python -c "import hashlib; print(hashlib.sha256(open('models/release/sed.pt', 'rb').read()).hexdigest())"
```

Keep the downloaded file out of commits: it is too large for the repository.

## What it is

- PANNs CNN14 with decision-level attention (Kong et al., 2020), started from the AudioSet checkpoint
  `Cnn14_DecisionLevelAtt_mAP=0.425.pth` (Zenodo record 3987831, CC BY 4.0). The whole network is
  fine-tuned on the training split, and a new attention head covers seven classes: phone, doorbell,
  knock, footsteps, door, cabinet (cupboard or drawer) and decoy (an unrelated household sound).
- Format: a PyTorch `state_dict` holding every weight of the network. Load it with
  `src/p1v4_sed.py` (`load`), or with `load_detector` in `src/demo.py`, which does not need the
  AudioSet file.
- Training: `python -X utf8 src/p1v4_sed.py train data/p1v4/runs/sed`. That is 3 epochs, batch 16,
  AdamW at learning rate 1e-4, 500 warm-up steps then cosine decay, bf16 autocast. The loss is
  binary cross-entropy on the frame outputs (100 per second) and on the clip outputs.
- Choice: epoch 3 of 3, picked by window-level macro F1 over the seven classes on the development set
  (0.876).
- To use it in the full pipeline instead of training it, copy it to `data/p1v4/runs/sed/sed.pt`.
