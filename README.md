# BCI-VR: Integrating CVSA and Motor Imagery

This repository serves as the **Meta-Repository** (Parent Repo) for a Brain-Computer Interface (BCI) system integrated with Virtual Reality (VR). The project is currently being conducted at the **ISR (Institute for Systems and Robotics)** in Lisbon.

---

## 🎯 Project Objective
The core of this research is to investigate the integration of two distinct BCI paradigms to enhance user control and performance:
1.  **Covert Visual Spatial Attention (CVSA)**
2.  **Motor Imagery (MI)**

The study hypothesizes that using CVSA as a complementary signal can facilitate the modulation of Motor Imagery, resulting in more robust and intuitive control within a virtual environment.

---

## 🕹️ VR Feedback Mechanism & Setup
The user interacts with a VR environment where the feedback is represented by a **dynamic cube**:
* **Transparency Axis (CVSA):** The cube materializes or dematerializes based on the focus of spatial attention.
* **Vertical Axis (MI):** The cube moves up or down based on the activation of Motor Imagery.

The system is designed to validate this dual approach, allowing for experimental sessions in both **MI-only** mode and combined **CVSA + MI** mode.

### Unity & VR Configuration
* **Software Version:** Unity 2022 LTS.
* **Tested Hardware:** Fully tested with the **Pico 4** VR headset.
* **VR Runtime:** SteamVR (v2.9.6).
* **Project Settings:**
    1. Go to `Edit` -> `Project Settings` -> `XR Plugin Management`.
    2. In the **Windows (PC)** tab, ensure **OpenXR** is selected.
    3. Under OpenXR, add the correct **Controller Profile** (e.g., *Oculus Touch Controller Profile*).
* **Audio Check:** To hear the spatial auditory feedback within the Unity Editor during tests, ensure you deactivate the **"Mute Audio"** button (or select "Unmute") located at the top of the **Game** view window.
* **Scene Adjustments:** Minor environmental tweaks might be necessary depending on the subject's anatomy. For instance, you can easily adjust the vertical position of the **"Finish bar"** in the Inspector for the **MI** and **Hybrid** paradigms to ensure it is comfortably reachable visually.
* **Connection:** Ensure the correct IP address is set in both the **'Robotics'** component and the **'ROS_manager'** script within the Unity scene.

---

## 🌐 Network & Multi-PC Setup
If the system is running across two different machines (e.g., one for signal processing and one for VR), you must configure the ROS environment variables:

1.  **Environment Variables:**
    ```bash
    export ROS_MASTER_URI=http://[MASTER_IP]:11311
    export ROS_IP=[LOCAL_IP]
    ```
2.  **IP Synchronization:** * Set the correct Master IP in the `ROS_CONNECTOR` settings.
    * Double-check that the IP matches in the Unity **'Robotics'** and **'ROS_manager'** fields to allow the TCP endpoint to communicate correctly.

---

## 🧠 Technical Stack & Signal Processing
The architecture relies on a professional BCI pipeline:
* **Communication:** Data streaming is handled via **LSL (Lab Streaming Layer)** for high-precision synchronization.
* **Framework:** The system is built on **ROS (Robot Operating System)**, specifically leveraging the **ROSNeuro** ecosystem for neural data acquisition and processing.
* **Feature Extraction:** * Signal power is extracted using the **Hilbert Transform** within the frequency bands of interest for both paradigms.
    * This provides a smooth power envelope used for real-time control.

---

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

---

## 📦 Reproducibility & Development Environment (Distrobox)

To ensure that the system runs with the exact same dependencies and ROS configuration used during development, this repository includes a dedicated **`distrobox/`** folder. 

Inside, you will find:
* **`Dockerfile`**: A complete "recipe" that builds an image with ROS Noetic, ROSNeuro (via official PPA), LibLSL, and all required C++/Python dependencies.
* **`README.md`**: Detailed instructions on how to build the image and create the container.

### Why use this?
* **Zero Configuration**: No need to manually install ROSNeuro or complex BCI libraries on your host OS.
* **Isolation**: Keeps your main operating system clean from specific BCI drivers and PPA repositories.
* **Consistency**: Guarantees that signal processing (Hilbert Transform, filters) behaves identically across different machines.

To get started with the environment, navigate to the folder and follow the instructions:
```bash
cd distrobox
# Read the README.md inside distrobox folder for setup steps
```