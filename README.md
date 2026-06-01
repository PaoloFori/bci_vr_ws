# BCI-VR: Integrating CVSA and Motor Imagery

This repository is the **meta-repository** (parent repo) for a Brain-Computer Interface (BCI) system integrated with Virtual Reality (VR). The project is conducted at the **ISR (Institute for Systems and Robotics)** in Lisbon.

---

## Project Objective

The core of this research is to investigate the integration of two BCI paradigms to enhance user control and performance:

1. **Motor Imagery (MI)** — imagined limb movement modulates a vertical axis
2. **Covert Visual Spatial Attention (CVSA)** — covert spatial attention modulates transparency
3. **Hybrid** — both signals fused via Bayesian integration (cosine-annealed LOP + agreement gate)

The study hypothesises that using CVSA as a complementary signal at trial onset (where attention is most informative) can facilitate MI modulation, resulting in more robust and intuitive VR control.

---

## VR Feedback Mechanism

The user interacts with a VR scene where feedback is represented by a **dynamic cube**:

- **Vertical axis (MI):** cube moves up/down based on imagined movement
- **Transparency axis (CVSA):** cube materialises/dematerialises based on spatial attention

### Unity configuration

- **Software:** Unity 2022 LTS
- **Hardware:** Pico 4 VR headset (tested)
- **VR runtime:** SteamVR v2.9.6 — enable OpenXR under XR Plugin Management (Windows tab)
- **Audio:** unmute the Game view to hear spatial auditory feedback
- **Connection:** set the correct IP in the `Robotics` component and `ROS_manager` script

---

## Network / Multi-PC Setup

```bash
export ROS_MASTER_URI=http://<master-ip>:11311
export ROS_IP=<this-machine-ip>
```

Also set the matching IP in the Unity `ROS_CONNECTOR` / `Robotics` fields.

---

## Signal Processing Pipeline

```
LSL (LiveAmp EEG)
  → /neurodata  (rosneuro_msgs/NeuroFrame, 500 Hz)
      ├─ processing_bci (MI)   → /mi/eeg_fbcsp     FBCSP features
      ├─ processing_bci (CVSA) → /cvsa/eeg_fbcsp   FBCSP features
      └─ artifacts_bci         → /artifact_presence

/*/eeg_fbcsp → slda_bci → /*/neuroprediction/raw   (softmax P per class)

/mi/raw + /cvsa/raw + /artifact_presence
  → rosneuro_integrator  → /*/neuroprediction/integrated/{raw,normalized}

/*/integrated/normalized → ROS-TCP-Endpoint → Unity VR
/*/integrated/raw        → training_node (hit/miss/timeout detection)
```

### FBCSP feature extraction (`processing_bci`)

For each frequency band and each 25-sample chunk (at 500 Hz / 20 Hz = 25 samples/chunk):

1. **CAR** — subtract mean of non-EOG channels (`Fp1`, `Fp2` excluded)
2. **Bandpass** — causal IIR Butterworth order 4, LP then HP (ba-form, zero ICs)
3. **Ring buffer** — 500-sample NaN-initialised shift register (fills after 20 chunks)
4. **CSP** — `buf_selected × W.T`, weights trained per subject and paradigm
5. **Log mean power** — `log(mean(x²))` per component per band → sLDA input

### Bayesian fusion (hybrid mode)

CVSA influence decays cosine-annealed from α=1 to α=0 over 2.5 s from event 781:

```
P_fused ∝ P_MI × P_CVSA^α   (LOP)
```

An agreement gate blends the LOP output toward uniform when MI and CVSA disagree and α > 0.

---

## Workspace Structure

All `src/` packages are Git submodules (`PaoloFori/*` on GitHub):

| Package | Language | Role |
|---|---|---|
| `processing_bci` | C++ | CAR → Butterworth BP → ring buffer → CSP → log mean power (FBCSP) |
| `artifacts_bci` | C++ | EOG + peak artifact detection; gates the classifier |
| `slda_bci` | Python | Shrinkage LDA classifier; model training notebook (`create_slda/`) |
| `feedback_bci_vr` | C++/Python | Protocol state machine (calibration/evaluation) + Unity scene manager |
| `rosneuro_integrator` | C++ | Sequence-sync + Bayesian fusion + artifact gating + leaky WTA buffer |
| `rosneuro_integrator_buffer` | C++ | Buffer plugin: winner-take-all leaky integrator (HARD/SOFT modes) |
| `rosneuro_acquisition_lsl` | C++ | Reads EEG from LSL, publishes `/neurodata` |
| `rosneuro_filters_car` | C++ | Common Average Reference spatial filter |
| `launchers_bci` | XML | Top-level launch files wiring the full pipeline |
| `test_pipeline` | C++/Python/MATLAB | Offline CSV/GDF replay + MATLAB integration tests (MAE ~1e-6 vs ROS) |
| `analysis_bci` | MATLAB | Offline simulation and cross-session metric evaluation |
| `bag_bci` | Python | Paradigm-aware ROS bag + GDF + rosparam-dump recording |

---

## Quick Start

```bash
# Clone with all submodules
git clone --recursive <URL>

# Update all submodules to their tracked branch
git submodule update --remote
```

### Environment (Distrobox / Podman)

```bash
cd distrobox && podman build -t bci_image .
distrobox create --name bci_cvsaMiVr --image localhost/bci_image
distrobox enter bci_cvsaMiVr
source /opt/ros/noetic/setup.bash && source devel/setup.bash
```

### Build

```bash
catkin_make                          # full workspace
catkin_make --pkg processing_bci     # single package
```

### Run

```bash
# Evaluation session
roslaunch launchers_bci evaluation.launch paradigm:=hybrid subject:=S01 trials:=10

# Offline integration test (CSV replay)
roslaunch test_pipeline test_pipeline.launch paradigm:=mi
```

---

## Offline Analysis

The `analysis_bci` package provides MATLAB tools for evaluating recorded sessions:

- **`matlab_simulation/`** — chunk-by-chunk replay of the full ROS pipeline (FBCSP → sLDA → artifact → integrator), per-trial plots, batch metric evaluation, cross-paradigm comparison
- **`analysis_gdf/`** — quick EEG visualisation scripts (`check_psd.m`, `eeglab_gdf.m`)

See `src/analysis_bci/README.md` for details.

---

## Model Training

CSP + sLDA models are trained from calibration GDF files using the Jupyter notebook:

```
src/slda_bci/create_slda/create_slda.ipynb
```

Pipeline: CAR → LP→HP bandpass (ba-form `lfilter`, zero ICs, matching ROS `rtfilter`) → 1-s sliding windows at CF segments → CSP per band → shrinkage LDA with optional feature selection and Platt calibration. Trained models are saved as YAML files loaded by the ROS nodes at runtime.

---

## Reproducibility Note

This repository includes a `distrobox/` folder with a Dockerfile that installs ROS Noetic, ROSNeuro (PPA), LibLSL, and all C++/Python dependencies, guaranteeing identical behaviour across machines.
