# BCI-VR: Integrating CVSA and Motor Imagery

This repository serves as the **Meta-Repository** (Parent Repo) for a Brain-Computer Interface (BCI) system integrated with Virtual Reality (VR). The project is currently being conducted at the **ISR (Institute for Systems and Robotics)** in Lisbon.

## 🎯 Project Objective
The core of this research is to investigate the integration of two distinct BCI paradigms to enhance user control and performance:
1.  **Covert Visual Spatial Attention (CVSA)**
2.  **Motor Imagery (MI)**

The study hypothesizes that using CVSA as a complementary signal can facilitate the modulation of Motor Imagery, resulting in more robust and intuitive control within a virtual environment.

## 🕹️ VR Feedback Mechanism
The user interacts with a VR environment where the feedback is represented by a **dynamic cube**:
* **Transparency Axis (CVSA):** The cube materializes or dematerializes based on the focus of spatial attention.
* **Vertical Axis (MI):** The cube moves up or down based on the activation of Motor Imagery.

The system is designed to validate this dual approach, allowing for experimental sessions in both **MI-only** mode and combined **CVSA + MI** mode.

## 🧠 Technical Stack & Signal Processing
The architecture relies on a professional BCI pipeline:
* **Communication:** Data streaming is handled via **LSL (Lab Streaming Layer)** for high-precision synchronization.
* **Framework:** The system is built on **ROS (Robot Operating System)**, specifically leveraging the **ROSNeuro** ecosystem for neural data acquisition and processing.
* **Feature Extraction:** * Signal power is extracted using the **Hilbert Transform** within the frequency bands of interest for both paradigms.
    * This provides a smooth power envelope used for real-time control.

## 🏗️ Workspace Structure (Submodules)
This workspace organizes the project through several ROS packages managed as Git Submodules:

* **`processing_bci`**: Filters and feature extraction nodes.
* **`analysis_bci`**: Data analysis and classification logic.
* **`launchers_bci`**: Configuration files to toggle between MI and CVSA+MI scenarios.
* **`feedback_bci_vr`**: Bridge for communication with Unity/VR.
* **`ros_tcp_endpoint`**: Dedicated ROS node for Unity integration.

---

## 🚀 Quick Start
To clone the entire workspace, including all submodules and dependencies:

```bash
git clone --recursive [URL_OF_THIS_REPO]
```