# BCI-VR: Integrating CVSA and Motor Imagery

Brain-Computer Interface system for VR-based neural rehabilitation and research at **ISR (Institute for Systems and Robotics)**, Lisbon. The system decodes two complementary EEG signals — Motor Imagery (MI) and Covert Visual Spatial Attention (CVSA) — and fuses them in real time to drive a VR scene on a Pico 4 headset.

---

## Table of Contents

1. [Paradigms](#1-paradigms)
2. [Event codes](#2-event-codes)
3. [EEG signal flow](#3-eeg-signal-flow)
4. [FBCSP feature extraction](#4-fbcsp-feature-extraction)
5. [sLDA classification](#5-slda-classification)
6. [Bayesian fusion (hybrid)](#6-bayesian-fusion-hybrid)
7. [Leaky integrator and VR feedback](#7-leaky-integrator-and-vr-feedback)
8. [Calibration vs evaluation](#8-calibration-vs-evaluation)
9. [Quick start](#9-quick-start)
10. [Offline analysis and model training](#10-offline-analysis-and-model-training)
11. [Workspace structure](#11-workspace-structure)

---

## 1. Paradigms

### Motor Imagery (MI)

The user **imagines moving their left or right hand** without any actual movement. This modulates the **sensorimotor rhythms** (8–30 Hz) over the motor cortex:

- **ERD** (Event-Related Desynchronization): power **decrease** over contralateral motor cortex (e.g., right-hand imagery → ERD over left C3/CP1)
- **ERS** (Event-Related Synchronization): power **increase** over ipsilateral motor cortex

The classifier captures this spatial-spectral pattern and outputs the probability of each imagined movement class.

**VR feedback**: the cube moves **up** (class 1 wins) or **down** (class 2 wins) proportionally to the normalised integrator output.

### Covert Visual Spatial Attention (CVSA)

The user **covertly attends to a stimulus on the left or right visual field** (eyes can remain central). This modulates the **alpha band** (8–13 Hz) in parieto-occipital areas:

- Attending **left** → alpha suppression in right occipital, alpha increase in left occipital
- Attending **right** → opposite pattern

CVSA is most informative at **trial onset** (first ~2.5 s of continuous feedback), before fatigue or distraction sets in.

**VR feedback**: the cube becomes **more solid** (less transparent) when the classifier is confident.

### Hybrid

Both signals are active simultaneously. The fuser combines them using a **Bayesian LOP** where:
- CVSA dominates **early in the trial** (α=1 at t=0) — fast, attention-driven onset
- MI takes over **progressively** (α→0 over 2.5 s) — sustained, movement-driven output
- When classifiers **disagree** the LOP products cancel near-uniformly → integrator naturally stalls (no explicit gate needed)

The hybrid paradigm is designed to exploit the complementary temporal profiles: CVSA is faster but not sustained; MI is slower to build up but robust at steady-state.

---

## 2. Event codes

All events are stored in the GDF recording (BIOSIG format) and on the ROS `/events/bus` topic (neuromask events).

| Code | Paradigm | Meaning |
|------|----------|---------|
| **769** | MI | Class 1 cue onset (left-hand imagery) |
| **770** | MI | Class 2 cue onset (right-hand imagery) |
| **730** | CVSA | Class 1 cue onset (left spatial attention) |
| **731** | CVSA | Class 2 cue onset (right spatial attention) |
| **750** | Hybrid | Class 1 cue onset |
| **751** | Hybrid | Class 2 cue onset |
| **781** | All | Continuous Feedback (CF) window starts — integrator resets to 0.5 |
| **897** | All | Hit — integrated output ≥ threshold within CF window |
| **898** | All | Miss — CF window ended without hit |
| **899** | All | Timeout — trial exceeded maximum duration |
| **33549** | All | Trial end marker |

**Trial sequence (evaluation mode)**:
```
onset_code (769/730/750)
  → [rest/preparation period]
  → 781  (CF starts, cube animates, integrator resets)
  → [CF window: 20 Hz frames, integrator accumulates]
  → 897 (hit) or 898/899 (miss/timeout)
  → 33549 (trial end)
  → next trial
```

---

## 3. EEG signal flow

```
LiveAmp amplifier (32-ch EEG, 500 Hz)
   │
   │  Lab Streaming Layer (LSL, TCP)
   ▼
rosneuro_acquisition_lsl
   │  publishes /neurodata  (rosneuro_msgs/NeuroFrame)
   │  at 20 Hz in chunks of 25 samples
   │
   ├──────────────────────────────────────────────────────────┐
   │  processing_bci (MI)          processing_bci (CVSA)     │
   │  6 bands: 8–12, 10–14,        3 bands: 7–11, 9–13,     │
   │           12–16, 14–18,                 11–15 Hz        │
   │           16–22, 20–30 Hz                               │
   │  11 motor channels            8 occipito-parietal ch    │
   │  [C3, Cz, C4, FC5, …]         [O1, Oz, O2, P7, …]      │
   │         │                             │                 │
   │  /mi/eeg_fbcsp (20 Hz)     /cvsa/eeg_fbcsp (20 Hz)     │
   │         │                             │                 │
   │  slda_bci (MI)             slda_bci (CVSA)             │
   │         │                             │                 │
   │  /mi/neuroprediction/raw   /cvsa/neuroprediction/raw   │
   │  [P(c1), P(c2)] at 20 Hz   [P(c1), P(c2)] at 20 Hz   │
   └──────────────────────────────────────────────────────────┘
                       │                   │
                       └────────┬──────────┘
                                │         artifacts_bci
                                │         /artifact_presence
                                ▼
                      rosneuro_integrator
                      (seq-sync + Bayesian LOP + WTA buffer)
                                │
                   ┌────────────┴────────────┐
                   ▼                         ▼
   /hybrid/neuroprediction/     /hybrid/neuroprediction/
      integrated/raw               integrated/normalized
   [0,1] per class              [0,1] per class, rescaled
                                        │
                              ROS-TCP-Endpoint (TCP)
                                        │
                                  Unity VR (Pico 4)
                            cube height (MI) + transparency (CVSA)
```

---

## 4. FBCSP feature extraction

Each 25-sample chunk (50 ms at 500 Hz) passes through the following causal pipeline, **replicated per frequency band**:

```
raw chunk [25 × 32]
   │
   │ 1. CAR (Common Average Reference)
   │    subtract per-sample mean of the 30 non-EOG channels
   │    (Fp1, Fp2 excluded — contaminated by eye blinks)
   ▼
chunk_car [25 × 32]
   │
   │ 2. Bandpass (per band, e.g. 8–12 Hz)
   │    2a. LP at 12 Hz:  butter(4, 12/250, 'low')  + lfilter, state zi_lp carried across chunks
   │    2b. HP at 8 Hz:   butter(4,  8/250, 'high') + lfilter, state zi_hp carried across chunks
   │    Filter order 4, ba-form (matches ROS rtfilter and MATLAB butter+filter)
   │    Zero initial conditions — filter fills from cold start
   ▼
chunk_bp [25 × 32]  (filtered in the 8–12 Hz band)
   │
   │ 3. Ring buffer (500 samples = 1 second)
   │    NaN-initialised shift register; valid after 20 chunks (1 s warmup)
   │    buf[:, selected_channels, band] ← [old[25:], chunk_bp[:, sel_ch]]
   ▼
buf [500 × 11 sel_channels]  (for MI; 8 for CVSA)
   │
   │ 4. CSP spatial filter
   │    W is [n_comp × n_sel_ch], trained per subject (Ledoit-Wolf regularised)
   │    csp_out = buf × W.T   →  [500 × n_comp]
   ▼
csp_out [500 × 4 components]  (typically 4 per band)
   │
   │ 5. Log mean power
   │    power = sum(csp_out²) / 500   →  [n_comp]  (mean over 500 samples)
   │    feature = log(power)
   ▼
features [n_comp × n_bands]  →  flattened to [n_comp*n_bands] → /eeg_fbcsp
```

**Example (MI, 6 bands × 4 components = 24 features per frame)**:

```
Band  8–12 Hz:  comp0=−2.31  comp1=−1.87  comp2=−3.05  comp3=−2.44
Band 10–14 Hz:  comp0=−2.18  comp1=−1.95  comp2=−2.88  comp3=−2.51
...
```

A left-hand trial would show strong ERD in bands 8–14 Hz for the right-motor CSP components (contralateral), which the sLDA discriminates from right-hand trials.

---

## 5. sLDA classification

The classifier is a **Shrinkage LDA** trained per subject from calibration data:

```
features (log mean power) [24]
   │
   │  score = W · features + b      (linear projection)
   │  where W = [1×24] trained weights, b = scalar intercept
   │
   │  P(c2) = σ(a·score + b_platt)  (Platt calibration, a≈1 if not fitted)
   │  P(c1) = 1 − P(c2)
   ▼
[P(c1), P(c2)] published at 20 Hz on /*/neuroprediction/raw
```

The model is trained in `src/slda_bci/create_slda/create_slda.ipynb` and saved as a YAML file loaded at runtime. Feature selection (mRMR/Fisher/miBIF) can reduce the 24 features to a smaller stable subset.

**Typical output range**: 0.3–0.7 at rest, 0.65–0.85 during strong MI or clear attention.

---

## 6. Bayesian fusion (hybrid)

The integrator receives three streams synchronised by sequence number:
- `p_MI = [P_MI(c1), P_MI(c2)]`
- `p_CVSA = [P_CVSA(c1), P_CVSA(c2)]`
- `has_artifact` (boolean)

### Temporal decay of CVSA influence

The parameter `cvsa_influence` (default 3.0 s) controls how long CVSA shapes the fusion. A cosine curve provides a smooth, monotonic transition — no abrupt plateau, strong CVSA at onset, slow early decay then faster mid-trial:

```
α(t) = 0.5 × (1 + cos(π × t / T))    T = cvsa_influence = 3 s

 t      α       MI weight   Description
 0.0   1.000      0%        CVSA fully active (spatial attention reliable at onset)
 0.5   0.933      7%        ← slow start: CVSA still dominant
 1.0   0.750     25%        ← CVSA still weighs 3× more than MI
 1.5   0.500     50%        ← crossover: MI and CVSA equally weighted
 2.0   0.250     75%        ← MI dominant
 2.5   0.067     93%        ← near pure MI
 3.0   0.000    100%        ← pure MI (CVSA ignored)
```

Rate of change of α: zero at t=0 (flat start), maximum at t=T/2 (steepest transition), zero again at t=T. This gives CVSA a "long plateau" at the start and ensures no abrupt switches.

### Logarithmic Opinion Pool (LOP)

```
P_prior(c) ∝ P_CVSA(c)^α          (temper CVSA by α; at α=0 → uniform prior)
P_out(c)   ∝ P_MI(c) × P_prior(c)  (Bayes update, normalised to sum=1)
```

When α=1 the prior equals P_CVSA. When α=0 the prior is uniform and P_out = P_MI.

### Worked examples

---

#### Example A — Agreement: both classifiers say LEFT (α = 1, t = 0 s)

```
P_MI   = [0.82, 0.18]   (MI: 82% LEFT)
P_CVSA = [0.78, 0.22]   (CVSA: 78% LEFT)
α = 1.0

P_prior ∝ [0.78^1, 0.22^1] = [0.78, 0.22]
P_out   ∝ [0.82×0.78, 0.18×0.22] = [0.640, 0.040]
P_out (normalised) = [0.941, 0.059]   ← well above either input alone
```

LOP amplifies agreement: P_out(c1) = 0.941 > P_MI(c1) = 0.82 > P_CVSA(c1) = 0.78.

How P_out evolves as α decays (same P_MI and P_CVSA throughout):

```
 t(s)    α       P_prior(c1)   P_out(c1)   Δ vs P_MI
  0.0   1.000      0.780         0.941       +0.121  ← peak LOP boost
  1.0   0.750      0.833         0.903       +0.083
  1.5   0.500      0.883         0.873       +0.053
  2.0   0.250      0.935         0.847       +0.027
  3.0   0.000      1.000         0.820        0.000  ← pure MI
```

The boost fades as α→0 but the output always matches or exceeds P_MI alone.

---

#### Example B — Disagreement: MI says LEFT, CVSA says RIGHT (α = 1, t = 0 s)

```
P_MI   = [0.70, 0.30]   (MI: LEFT)
P_CVSA = [0.30, 0.70]   (CVSA: RIGHT — user's attention drifted)
α = 1.0

P_prior ∝ [0.30, 0.70]
P_out   ∝ [0.70×0.30, 0.30×0.70] = [0.21, 0.21]
P_out (normalised) = [0.500, 0.500]   ← products cancel exactly
```

How P_out recovers as α decays (same classifiers throughout):

```
 t(s)    α       P_prior(c1)   P_out(c1)   step (SOFT)   buffer Δ/s
  0.0   1.000      0.300         0.500       0.000          0.00  ← frozen
  0.5   0.933      0.317         0.507       0.001          0.01
  1.0   0.750      0.346         0.553       0.005          0.10
  1.5   0.500      0.396         0.605       0.011          0.21
  2.0   0.250      0.447         0.653       0.016          0.31
  3.0   0.000      0.500         0.700       0.020          0.40  ← pure MI
```

At t=0 the opposing CVSA prior exactly cancels MI and the buffer freezes. As α decays, the output converges toward P_MI = [0.70, 0.30] and the buffer slowly starts accumulating.

---

#### Example C — CVSA uncertain (α = 0.5, mid-trial)

```
P_MI   = [0.78, 0.22]   (MI confident)
P_CVSA = [0.52, 0.48]   (CVSA near-chance — attention not established)
α = 0.5

P_prior ∝ [0.52^0.5, 0.48^0.5] = [0.721, 0.693]  (normalised: [0.510, 0.490])
P_out   ∝ [0.78×0.510, 0.22×0.490] = [0.398, 0.108]
P_out (normalised) = [0.787, 0.213]

Result: P_out ≈ P_MI  (near-uniform prior has almost no effect)
```

When CVSA is uncertain, its prior approaches uniform regardless of α. The fusion reduces to near pure MI automatically.

---

#### Example D — Full trial: two scenarios compared (T = 3 s)

Parameters: `buffer_size=40`, `k_gain=2`, `framerate=20 Hz`, `threshold_c1=0.95`.

SOFT step formula: `step = min(|P_out − 0.5| × 2 × k_gain, 1) / buffer_size`

**Scenario 1 — Agreement** (P_MI=[0.75,0.25], P_CVSA=[0.70,0.30] throughout):

```
Frame   t(s)    α      P_out(c1)   step    buffer(c1)
  1     0.00   1.000   [reset]      —       0.500
  2     0.05   0.999   0.875       0.025    0.525   ← max step (LOP amplifies)
 10     0.45   0.918   0.860       0.025    0.725
 19     0.90   0.794   0.841       0.025    0.950   → HIT (event 897) at 0.9 s
```

P_out stays ≥ 0.84 throughout (LOP boost), so step is always capped at 0.025 (max). HIT in under 1 second.

**Scenario 2 — Disagreement** (P_MI=[0.70,0.30], P_CVSA=[0.30,0.70] throughout):

```
Frame   t(s)    α      P_out(c1)   step    buffer(c1)
  1     0.00   1.000   [reset]      —       0.500
  2     0.05   0.999   0.500       0.000    0.500   ← frozen (LOP cancels)
 10     0.45   0.918   0.518       0.002    0.505   ← barely moving
 20     0.95   0.772   0.548       0.005    0.540
 30     1.45   0.451   0.615       0.012    0.622   ← α past midpoint, MI taking over
 40     1.95   0.210   0.662       0.016    0.761
 51     2.50   0.067   0.692       0.019    0.950   → HIT at 2.5 s (barely within window)
```

Even with a completely opposing CVSA, the buffer eventually reaches threshold — but at 2.5 s instead of 0.9 s. If the CF window is shorter, this trial would timeout.

---

## 7. Leaky integrator and VR feedback

### SOFT leaky WTA integrator

At each non-artifact frame the buffer updates with a **velocity-scaled step**:

```
p_in = P_out(c1)   (fused probability of class 1)

step = min( |p_in − 0.5| × 2 × k_gain,  1 ) / buffer_size

if p_in > 0.5:   buffer(c1) += step   (increase class-1 evidence)
                 buffer(c2) -= step   (decrease class-2 evidence)
else:            symmetric in the other direction

buffer(c) clipped to [0, 1]
reset() → buffer = [0.5, 0.5]  (at every event 781)
```

**Default parameters** (from `evaluation.launch`):
- `buffer_size = 40` (40 frames = 2 s to fill from 0.5 to 1.0 at full certainty)
- `k_gain = 2.0` (doubles the step at full certainty → fills in ~1 s)
- `thresholds = [0.95, 0.75]` (class 1 needs higher confidence than class 2)

**Why asymmetric thresholds**: class 2 (right hand / right attention) is often easier to decode, so a lower threshold compensates, making both classes equally reachable within the CF window.

### Normalised output → VR

```
raw_i     = buffer(class_i)
normalized = p_rest + (raw − p_rest) × (1 − p_rest) / (threshold − p_rest)
           = raw mapped so that [p_rest=0.5 … threshold] → [0.5 … 1.0]

offset    = max(0,  (normalized − 0.5) × 2 )   (Unity: [0.5, 1.0] → [0, 1])
```

The cube reaches its **goal position/full opacity** exactly when `raw = threshold`, i.e., when `normalized = 1.0`. This gives the user a direct visual cue that threshold was crossed.

---

## 8. Calibration vs evaluation

### Calibration mode

```bash
roslaunch launchers_bci calibration.launch paradigm:=mi subject:=S01
```

- `training_node` publishes **autopilot signals** (LinearPilot for active classes, SinePilot for rest) to `/*/integrated/raw` and `/*/integrated/normalized`.
- No real sLDA output is used — the autopilot drives the VR cube to demonstrate the feedback.
- The BCI amplifier is still recording, and bag_bci saves the GDF + rosparam YAML.
- Purpose: subject learns the task, calibration data is collected for model training.

### Evaluation mode

```bash
roslaunch launchers_bci evaluation.launch paradigm:=hybrid subject:=S01 trials:=10
```

- `rosneuro_integrator` owns the pipeline: real EEG → FBCSP → sLDA → fusion → buffer → VR.
- `training_node` subscribes to `/*/integrated/raw` and detects hits (`raw[i] ≥ threshold[i]`).
- On hit: publishes event 897, advances trial counter.
- On timeout: publishes 898/899, moves to next trial.
- `bag_bci` records GDF + YAML alongside.

### Key launch parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `paradigm` | `cvsa` | `mi` / `cvsa` / `hybrid` |
| `subject` | — | Subject ID (used in filenames) |
| `trials` | — | Trials per class |
| `samplerate` | `500` | EEG sample rate (Hz) |
| `framerate` | `20` | Processing rate (Hz) |
| `filters_band_mi` | `8.0 12.0; 10.0 14.0; 12.0 16.0; 14.0 18.0; 16.0 22.0; 20.0 30.0` | MI frequency bands |
| `filters_band_cvsa` | `7.0 11.0; 9.0 13.0; 11.0 15.0` | CVSA frequency bands |
| `bufferSize` | `40` | Integrator buffer size (frames) |
| `k_gain` | `2.0` | SOFT step gain |
| `thresholds` | `[0.95, 0.75]` | Hit thresholds per class |

---

## 9. Quick start

### Setup environment

```bash
# Build container image (once)
cd distrobox && podman build -t bci_image .

# Create and enter the Distrobox container
distrobox create --name bci_cvsaMiVr --image localhost/bci_image
distrobox enter bci_cvsaMiVr

# Source ROS + workspace
source /opt/ros/noetic/setup.bash
source /home/paolo/bci_vr_ws/devel/setup.bash
```

### Build

```bash
catkin_make                          # full workspace
catkin_make --pkg processing_bci     # single package
catkin_make -DCMAKE_BUILD_TYPE=Debug # with debug symbols
```

### Multi-PC setup (EEG PC + VR PC)

```bash
# on both machines:
export ROS_MASTER_URI=http://<eeg-pc-ip>:11311
export ROS_IP=<this-machine-ip>
```

Set the same master IP in Unity's `Robotics` / `ROS_manager` component.

### Run evaluation session

```bash
# Hybrid paradigm, 10 trials per class
roslaunch launchers_bci evaluation.launch \
  paradigm:=hybrid subject:=S01 trials:=10

# MI only
roslaunch launchers_bci evaluation.launch paradigm:=mi subject:=S01 trials:=12

# Override thresholds for this session
roslaunch launchers_bci evaluation.launch \
  paradigm:=hybrid subject:=S01 trials:=10 thresholds:=[0.90,0.70]
```

### Offline pipeline test (no hardware)

```bash
# Replay a CSV file through the full pipeline
roslaunch test_pipeline test_pipeline.launch paradigm:=mi

# Replay a GDF file over LSL
roslaunch test_pipeline test_pipeline_lsl.launch \
  paradigm:=mi gdf_file:=/path/to/file.gdf
```

---

## 10. Offline analysis and model training

### Model training (after calibration)

```bash
cd src/slda_bci/create_slda
jupyter notebook create_slda.ipynb
```

The notebook:
1. Loads calibration GDF(s) → applies CAR + bandpass
2. Runs grid search over N_CSP_COMPONENTS and K features
3. Cross-validates sLDA + optional feature selection (mRMR/Fisher/miBIF)
4. Saves CSP YAML (`processing_bci/cfg/csp/`) and sLDA YAML (`slda_bci/models/`)
5. Saves diagnostic figures to `<gdf_dir>/images/<subject>_<paradigm>_*.png`

### Offline simulation (MATLAB)

After collecting evaluation GDF files, analyse performance in MATLAB:

```matlab
cd src/analysis_bci/matlab_simulation

% Inspect a single recording (per-trial plot of integrator signal)
main_simulate

% Full pipeline on one or more GDFs → metrics + ERD/ERS + topoplots + SVG
main_session_overview

% Matched hybrid-vs-MI-vs-CVSA comparison (saved-trial analysis)
main_hybrid_advantage

% Interactive scrollable viewer: classifier probabilities + integrator output
main_browse_gdf
```

`main_session_overview` produces per-session SVG figures in `<gdf_dir>/results/<basename>/`:
- `01_metrics.svg` — trial accuracy (897/898/899), sLDA frame accuracy per class, time-to-hit
- `02_erd_classXXX.svg` — ERD/ERS heatmap [time × channel] per class, per frequency band
- `03_spatial_erd.svg` — toposcatter maps: static (mean CF) + time snapshots

---

## 11. Workspace structure

All `src/` packages are Git submodules (`PaoloFori/*` on GitHub, branch-tracked):

| Package | Language | Role |
|---|---|---|
| `processing_bci` | C++ | CAR → Butterworth BP → ring buffer → CSP → log mean power (FBCSP) |
| `artifacts_bci` | C++ | EOG + peak artifact detection; gates the classifier pipeline |
| `slda_bci` | Python | Shrinkage LDA + Platt calibration; training notebook; model YAMLs |
| `feedback_bci_vr` | C++/Python | Protocol state machine (calibration/evaluation) + Unity scene manager |
| `rosneuro_integrator` | C++ | Seq-sync + cosine-annealed Bayesian LOP (no agreement gate) + artifact gating |
| `rosneuro_integrator_buffer` | C++ | Leaky WTA buffer plugin (HARD/SOFT step modes, `dynamic_reconfigure`) |
| `rosneuro_acquisition_lsl` | C++ | LiveAmp LSL stream → `/neurodata` (NeuroFrame) |
| `rosneuro_filters_car` | C++ | Common Average Reference spatial filter |
| `launchers_bci` | XML | Top-level launch files for calibration and evaluation |
| `test_pipeline` | C++/Python/MATLAB | Offline replay + MATLAB integration tests (MAE < 1e-6 vs ROS) |
| `analysis_bci` | MATLAB | Per-session evaluation + ERD/ERS + topoplots + cross-session comparison |
| `bag_bci` | Python | Paradigm-aware GDF + ROS bag + rosparam-dump recording |

```bash
# Clone with all submodules
git clone --recursive <URL>

# Update all submodules to their tracked branch HEAD
git submodule update --remote
```

Key branches in use: `fbcsp_vr` (processing), `cvsa_mi` (integrator, test_pipeline, analysis), `lsl_vr` (launchers), `no_eog` (CAR filter).
