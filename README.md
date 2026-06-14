# Driver-Monitoring-System-for-ADAS-Level

> Python · MediaPipe · OpenCV · CARLA 0.9.15 · ISO 26262-aligned FMEA

 **The driver must always remain ready to retake control**. This project builds the safety bridge: a real-time driver monitoring system that detects distraction and drowsiness, escalates alerts through four tiers, and commands gentle braking in a CARLA simulation when the driver does not respond.

---

## Demo

> _Screenshot / GIF placeholder — record a 15s clip of the HUD overlay with a MILD → DISTRACTED → CRITICAL escalation and replace this line._
>
> Suggested workflow: `python hud_overlay.py --demo` → record with OBS or `ffmpeg`, export as GIF.

---

## What it does

| Signal | Method | Output |
|---|---|---|
| Head pose | MediaPipe FaceMesh + OpenCV solvePnP | Yaw + pitch (degrees) |
| Gaze zone | Iris landmark ratio, 7-frame EWMA | LEFT / CENTER / RIGHT |
| Blink rate | Eye Aspect Ratio, 30 s rolling window | Blinks per minute |
| **Attention score** | Weighted fusion (pose 40%, gaze 35%, blink 25%) | **0–100** |

The score drives a four-tier state machine — ATTENTIVE → MILD WARNING → DISTRACTED → CRITICAL — with hysteresis, dwell timers, and cooldowns to suppress false positives. On CRITICAL, a linear brake ramp is applied to the CARLA vehicle if the driver does not respond within 5 seconds.

---

## Project structure

```
dms-project/
├── m1_perception/
│   ├── verify_install.py       # Dependency smoke test
│   ├── head_pose.py            # Yaw/pitch via solvePnP
│   ├── gaze_zone.py            # Iris ratio gaze classifier
│   ├── blink_rate.py           # EAR blink counter + rate
│   └── attention_score.py      # Weighted fusion → score 0-100
│
├── m2_alerts/
│   ├── alert_state.py          # 4-tier state machine + audio
│   ├── audio_alert.py          # Synthesized pygame tones
│   └── hud_overlay.py          # Full-frame OpenCV HUD
│
├── m3_carla/
│   ├── carla_client.py         # Connection + map load + spawn
│   ├── spawn_autopilot.py      # Town04 autopilot scenario
│   ├── disengagement.py        # L2 disengagement event (3 trigger modes)
│   └── brake_control.py        # Linear ramp VehicleControl + SharedDMSState
│
├── m4_artifacts/
│   ├── logger.py               # 30 Hz CSV logger (background queue thread)
│   ├── DMS_PRD_001.docx        # Product Requirements Document
│   └── DMS_FMEA_001.docx       # Failure Mode and Effects Analysis (20 entries)
│
└── logs/                       # Session CSVs written here at runtime
```

---

## Quickstart

### 1. Prerequisites

- Python **3.10** (CARLA wheels require 3.10; MediaPipe 0.10.x supports 3.8–3.11)
- [CARLA 0.9.15](https://github.com/carla-simulator/carla/releases/tag/0.9.15) downloaded and extracted (optional — needed for M3 only)
- Webcam for live mode (M1/M2 work without CARLA)

### 2. Create a Python 3.10 virtual environment

```bash
python3.10 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install mediapipe opencv-python numpy pandas pygame
```

### 3. Download the MediaPipe model (once)

```bash
curl -L https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task \
     -o face_landmarker.task
```

### 4. Verify your environment

```bash
python verify_install.py
```

All lines should print `OK`. The webcam line prints `WARN` on headless machines — that is expected.

### 5. Run the attention scorer (M1 complete pipeline)

```bash
python attention_score.py
```

Opens a webcam window showing the live attention score, all three signal bars, and the tier badge. Press `Q` to quit.

### 6. Run the HUD demo (no webcam required)

```bash
python hud_overlay.py --demo
```

Cycles through all four alert tiers with a simulated score profile. Good for a quick screenshot or GIF recording.

### 7. Run the CARLA scenario (M3 — requires running CARLA server)

```bash
# Start CARLA in a separate terminal first:
./CarlaUE4.sh -quality-level=Low -fps=30

# Then in another terminal:
python disengagement.py --trigger time --duration 12
```

Spawns a Tesla Model 3 on Town04, drives for 12 seconds, shows a 3-second countdown, disengages autopilot, and applies a brake ramp if no driver response.

---

## Install CARLA Python package

```bash
# Option A: PyPI (Python 3.8–3.11, functionally identical for this project)
pip install carla==0.9.16

# Option B: exact 0.9.15 wheel from the release archive
pip install CARLA_0.9.15/PythonAPI/carla/dist/carla-0.9.15-cp310-cp310-linux_x86_64.whl
```

---

## Key design decisions

**Why a weighted penalty fusion instead of a classifier?**
Rule-based fusion is transparent and tunable without a labelled dataset. The three weights (40/35/25) are documented in the PRD and can be adjusted per deployment context. A CNN-based drowsiness classifier is a planned Phase 2 enhancement (see FMEA FM-10).

**Why a dwell timer on alert escalation?**
A brief glance away (mirror check, lane change) should not trigger MILD WARNING. The 1.5-second dwell requirement ensures the score must stay below the threshold for a sustained period before the tier commits. This is the single most important false-positive suppressor in the system.

**Why a one-way brake ratchet?**
Once the brake value increases, it never decreases within a session. This is a safety invariant: even if the driver briefly opens their eyes (temporarily improving the attention score), the vehicle does not surge forward. Documented in FMEA FM-16.

**Why SharedDMSState instead of direct function calls across threads?**
The DMS perception loop runs at 30 fps; the CARLA control loop runs at 20 Hz. A `threading.Lock`-protected shared state container decouples the two loops cleanly, allowing either to be swapped or scaled independently.

---

## TPM artifacts

This project includes a complete TPM artifact set, as would be expected for a production ADAS program:

| Artifact | Description |
|---|---|
| `DMS_PRD_001.docx` | 11-section PRD: problem statement, 6 user stories, 13 functional requirements (P0/P1/P2), NFRs, milestones, success metrics |
| `DMS_FMEA_001.docx` | FMEA with 20 failure modes across 7 functional areas. 5 HIGH-priority items (RPN ≥ 120). ISO 26262-aligned severity ratings |
| Architecture diagrams | Five-layer structural overview + data-flow diagram (in project wiki / README images) |

The FMEA directly references module-level controls — for example, the gate in `BrakeController` that prevents braking while autopilot is active (FM-16, RPN 36) and the EAR threshold gap that allows microsleep to go undetected (FM-10, RPN 280, recommended fix: PERCLOS).

---

## Session log analysis

Every run writes a 15-field CSV to `logs/`. Load and analyse with:

```python
from logger import load_session, print_analysis

df = load_session("logs/dms_20260615_143022.csv")
print_analysis(df)

# Time in each tier (seconds)
df.groupby("tier")["elapsed_s"].count() / 30

# All alert events
df[df["alert_fired"]].groupby("tier").size()

# Score distribution when distracted
df[df["tier"] == "DISTRACTED"]["score"].describe()
```

---

## FMEA highlights

| ID | Failure Mode | RPN | Status |
|---|---|---|---|
| FM-10 | Microsleep not detected (partial eye closure) | 280 | Phase 2: PERCLOS |
| FM-08 | Phone-in-lap gaze not detected | 252 | Phase 2: pitch proxy |
| FM-05 | Systematic yaw bias from off-centre camera | 175 | Mitigated by threshold offset |
| FM-06 | False distraction during mirror check | 144 | Mitigated by dwell timer |
| FM-03 | Landmark jitter causes score oscillation | 120 | Mitigated by EWMA + smoothing |

Full table with S/O/D ratings, root causes, current controls, and recommended actions in `DMS_FMEA_001.docx`.

---

## Tech stack

| Layer | Technology |
|---|---|
| Face tracking | MediaPipe FaceMesh 0.10.x (478-point model, Tasks API) |
| Head pose | OpenCV solvePnP + RQDecomp3x3 |
| Gaze / blink | MediaPipe iris landmarks + Eye Aspect Ratio |
| Simulation | CARLA 0.9.15, Town04 map, Traffic Manager |
| HUD | OpenCV frame drawing |
| Audio | pygame synthesized tones (no WAV files required) |
| Logging | pandas + CSV, background thread queue |
| Language | Python 3.10 |

---

## Roadmap (Phase 2)

- [ ] PERCLOS drowsiness metric to address FM-10 (microsleep detection gap)
- [ ] Pitch-based gaze proxy to address FM-08 (phone-in-lap blind spot)
- [ ] Per-session EAR calibration to address FM-09 (anatomy-dependent threshold)
- [ ] nuScenes dataset integration for attention score validation
- [ ] CNN-based drowsiness classifier (Closed Eyes in the Wild dataset)
- [ ] Multi-driver profile support

---

## References

- Soukupova & Cech (2016) — Real-Time Eye Blink Detection Using Facial Landmarks, CVWW 2016
- ISO 26262:2018 — Road vehicles: Functional safety
- MediaPipe FaceLandmarker — [ai.google.dev](https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker)
- CARLA Simulator — [carla.org](https://carla.org)
- NHTSA Traffic Safety Facts (2024) — Distraction-Affected Crashes
- IEA Clean Energy Transitions Programme (2026) — Autonomous Vehicles Update

---

## License

MIT — free for portfolio and research use. Not approved for production vehicle deployment.
