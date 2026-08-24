# Astronaut-Activity-Recognition-System
AI-powered astronaut activity recognition system for validating experimental procedures in real time using computer vision, object detection, pose estimation, and hand–object interaction analysis.

<div align="center">

# AEGIS · AI-HAR

**AI Human Activity Recognition for On-board BAS Experiments**

Smart India Hackathon 2026 · Problem Statement **26174** · ISRO / Department of Space

*Recognises and validates the sequence of a pre-defined experiment, on-board,
offline, with voice guidance — so science does not wait for a round trip to Earth.*

</div>

---

## The 60-second version

An astronaut performing a scientific protocol has no ground support in real time.
AEGIS watches the payload camera, recognises what the operator is doing, checks it
against the expected protocol, tells them what to do next, and shouts if a step is
skipped or performed out of order. Everything runs locally. Every decision is
logged with its evidence.

```
INSTALL.bat  →  CALIBRATE_ZONES.bat  →  START.bat
```

That is the whole path from a fresh laptop to a working console.

---

## Why this design

Most attempts at this problem gate everything behind a custom-trained object
detector. That means **nothing works until the dataset is finished** — which, in a
hackathon, is the night before. AEGIS is built as a **capability ladder** where
every rung actually runs:

| Tier | Evidence used | Setup cost | Accuracy |
|:---:|---|---|---|
| **0** | Rack-relative zones + grip aperture + dwell, derived from your protocol YAML | **none** — works the moment you plug in a webcam | demonstrable |
| **1** | GRU over 32-frame windows of ~90 rack-relative features, ONNX, offline | ~30 min of your own clips | good |
| **2** | Custom YOLO object classes *(optional)* | a labelling afternoon | best |

Tier 1 overrides Tier 0 only when it is confident; otherwise the heuristic keeps
the session moving. **The active tier is displayed in the GUI**, so a jury can see
exactly which evidence path validated each step.

### Orientation-agnostic, without the weight of HMR

The statement notes that ground-based posture models fail because astronauts have
no fixed "up". The fix is not to guess gravity — it is to stop depending on it.

Four ArUco markers on the payload rack give a homography into **rack coordinates**,
recovered every frame. Every feature is *relational within that frame*: distances,
angles, ratios between the operator and the rack. Rotate the operator 180° and the
features are unchanged.

> Verified, not asserted: `tests/test_rack_frame.py` renders the scene at
> 0°, 30°, 90°, 180° and 270° and asserts a fixed physical point keeps the same
> rack coordinates. Occlusion is covered too — losing the markers behind an arm
> holds the last good frame instead of collapsing.

---

## Quick start

### Requirements

* Windows 10/11 (Linux and macOS work; the `.bat` files are Windows)
* **64-bit Python 3.11** — tick **Add python.exe to PATH** *and* **tcl/tk and IDLE**
* Any USB webcam
* No GPU required

### Install and run

| Step | Command | What it does |
|---|---|---|
| 1 | `INSTALL.bat` | Creates `.venv`, installs everything, runs diagnostics |
| 2 | `MAKE_ARUCO_MARKERS.bat` | Prints a rack marker sheet *(optional but recommended)* |
| 3 | `CALIBRATE_ZONES.bat` | Click the rack corners, then draw your work zones |
| 4 | `START.bat` | **Launch the mission console** |

Then press **START SESSION**.

### Make it yours

Edit `configs/protocol.yaml`. It is the only file you need to change to re-target
the system at a different experiment — no Python edits, no retraining of the
perception stack.

```yaml
- id: 5
  name: "Transfer sample to tray"
  action: transfer_sample          # snake_case; the verb drives the Tier-0 rules
  instruction: "Transfer the sample from the red box into the sample tray."
  zone: sample_tray                # a zone you drew during calibration
  min_confidence: 0.68
  dwell_s: 0.7
  safety_critical: true            # skipping this HALTS guidance
  allow_manual_skip: false
```

### Train your own model (the real deliverable)

| Step | Command | Time |
|---|---|---|
| 5 | `RECORD_DATASET.bat` | ~30 min |
| 6 | `TRAIN_MODEL.bat` | 2–10 min |

Restart the app; it loads the model automatically and the tier indicator flips to
**Tier 1**. See [`docs/TRAINING.md`](docs/TRAINING.md) for the recipe that
actually produces a good model.

---

## The console

```
┌──────────────────────────────────────────┬───────────────────────────┐
│  AEGIS   AI-HAR                    ● ● ● │  NEXT STEP                │
├──────────────────────────────────────────┤  STEP 5 — Transfer sample │
│                                          │  Transfer the sample from │
│         live annotated video             │  the red box into the     │
│      skeletons · rack quad · zones       │  sample tray.             │
│         HUD · alerts · timestamp         │  ▓▓▓▓▓▓▓░░░  4/8 verified │
│                                          ├───────────────────────────┤
│                                          │  PROTOCOL                 │
│                                          │  ● 01 Open outer   OK     │
│                                          │  ● 02 Retrieve red OK     │
│                                          │  ● 05 Transfer    > NOW   │
├──────────────────────────────────────────┤  ● 06 Close red    CRIT   │
│ FPS  LATENCY  RECOGNITION  OBSERVING     ├───────────────────────────┤
│ 28.4  34.2ms   T1-learned   grasp 0.82   │  EVENT LOG                │
└──────────────────────────────────────────┴───────────────────────────┘
  START · STOP · ACKNOWLEDGE · CONFIRM · SKIP · RECORD · STREAM · VOICE
```

Six status lamps (camera, model, rack, voice, record, stream) mean you can tell at
a glance which subsystem is degraded — instead of wondering why nothing happens.

---

## What gets produced

Every session writes to `logs/`:

| File | For |
|---|---|
| `<session>.log` | Humans — fixed-width, `UTC \| ELAPSED \| EVENT \| STEP \| STATUS \| CONF \| DETAIL` |
| `<session>.jsonl` | Machines — one JSON object per line, ready for downlink |
| `<session>_report.txt` | The verdict — per-step outcomes, deviations, `PASS`/`FAIL` |

Plus `outputs/recordings/<session>.mp4` with the HUD burned in, so the archive
shows what the system believed and when.

```
Verdict         : FAIL
Steps completed : 4/8
Steps failed    : 1  (incl. 1 bypassed safety-critical)

ID  STEP                          STATUS      CONF    DUR(s)   NOTE
1   Open outer container          DONE        0.81    4.20     
5   Transfer sample to tray       BLOCKED     0.00    0.00     BYPASSED - safety critical
```

---

## Streaming to ground

Two modes, because *"stream to a specific IP"* means different things depending on
who opens the connection.

**Pull (default).** An MJPEG endpoint. The ground station opens
`http://<payload-pc>:8090/` in a browser or VLC. No extra software either end.

**Push (optional).** Set `stream_push_url: rtsp://10.0.0.5:8554/bas` and frames
are piped through `ffmpeg`.

Only the latest frame is ever held, so a slow client throttles itself rather than
stalling the perception loop.

---

## Everything else

| | |
|---|---|
| `DIAGNOSTICS.bat` | **Run this first when anything misbehaves.** Checks every subsystem and prints a concrete fix for each failure |
| `RUN_TESTS.bat` | 45 tests, no camera or GPU needed |
| `FETCH_MODELS.bat` | Downloads MediaPipe bundles if your wheel needs them |
| `START_LIVE.bat` | Straight into a running session with streaming on |

---

## Repository layout

```
configs/
  app.yaml              runtime settings, fully commented
  protocol.yaml         THE EXPERIMENT — edit this
  zones.json            written by CALIBRATE_ZONES.bat
src/aegis/
  capture/              threaded source, recorder, MJPEG streamer
  perception/
    rack_frame.py       ArUco homography — the orientation-agnostic core
    landmarks.py        MediaPipe, 3 interchangeable backends
    zones.py            rack-relative polygons
    features.py         ~90 relational features per frame
    recognizer.py       Tier 0 heuristic + Tier 1 learned + fusion
  protocol/
    spec.py             protocol dataclasses and validation
    engine.py           SEQUENCE VALIDATION — pure, fully unit-tested
  outputs/              voice, logbook, HUD overlay
  gui/                  Tkinter mission console
  tools/                calibrate · record · train · diagnostics · markers
  pipeline.py           orchestration
tests/                  45 tests
docs/
  SIH_REQUIREMENTS.md   requirement-by-requirement traceability
  TRAINING.md           how to get a model that actually works
  ARCHITECTURE.md       design decisions and why
  JURY_DEMO.md          a 5-minute demo script
```

---

## Design notes worth defending

**The protocol engine is pure.** It takes observations and a timestamp, returns
events. No camera, no audio, no disk. That is why the safety-critical logic is
*unit-tested* rather than merely demonstrated — 13 tests cover skip detection,
out-of-sequence repeats, safety blocking, recovery, dwell gating and timeouts,
none of which need hardware.

**Freshness beats completeness.** Capture, recording and streaming all drop frames
under load rather than queue them. A warning that arrives late is worse than a
warning computed from a slightly older frame.

**Landmarks, not pixels.** A 21-point hand plus a 33-point body is a ~160-number
description invariant to lighting, skin tone and sleeve colour. Training on that
needs hundreds of clips, not hundreds of thousands — the only reason a custom
protocol is trainable inside a hackathon.

**Validation holds out an operator, not random windows.** Consecutive sliding
windows from one clip are near-duplicates; a random split reports ~99% and then
collapses on a new person. The number in the model card is the honest one.

**Degradation is explicit, never silent.** Every fallback — motion-only
perception, uncalibrated zones, held rack frame, missing model — is surfaced in the
GUI and written to the session log. The system never pretends to be more capable
than it is.

---

## Limitations

Stated plainly, because a jury will ask:

1. Out of the box, recognition is heuristic-grade. Tier 1 needs your clips.
2. Actions that differ only by *which object is held* are the weak point — that is
   what optional Tier 2 is for.
3. One camera geometry per calibration.
4. Microgravity invariance is proven mathematically and in synthetic rotation
   tests, not in freefall.
5. Voice depends on an OS speech engine; the visual alert path is independent.

---

<div align="center">

**[Requirements traceability](docs/SIH_REQUIREMENTS.md)** ·
**[Training guide](docs/TRAINING.md)** ·
**[Architecture](docs/ARCHITECTURE.md)** ·
**[Demo script](docs/JURY_DEMO.md)**

</div>
