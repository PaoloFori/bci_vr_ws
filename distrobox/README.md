# BCI VR Workspace (ROS Noetic + ROSNeuro)

This repository contains a ROS-based workspace for Brain-Computer Interface (BCI) and Virtual Reality (VR) applications. The environment is fully containerized using **Distrobox** and **Podman/Docker** to ensure reproducibility and easy setup across different Linux distributions.

---

## 📋 Prerequisites

Before starting, ensure you have the following installed on your host system:
1. **Podman** (recommended) or **Docker**.
2. **Distrobox** (tested on version 1.4.1+).

---

## 🚀 Environment Setup

Depending on your Distrobox version, choose one of the following options to build the environment.

### Option A: Manual Build via Podman (Recommended for Distrobox < 1.5.0)
If your version does not support the direct `--dockerfile` flag:

1. **Build the local image:**
   ```bash
   podman build -t bci_image .
   ```
2. **Create the Distrobox container:**
   ```bash
   distrobox create --name bci_env --image localhost/bci_image
   ```

### Option B: Direct Build via Distrobox (Newer versions)
If you have a recent version of Distrobox:
   ```bash
   distrobox create --name bci_env --dockerfile ./Dockerfile
   ```

---

## 🛠️ Compilation and Usage
Once the container is created, follow these steps to enter and compile the workspace:
1. **Enter the container:**
   ```bash
   distrobox enter bci_env
   ```
2. **Initialize the ROS environment (inside the container):**
   ```bash
   source /opt/ros/noetic/setup.bash
   ```
3. **Compile the Workspace:**
Navigate to your workspace folder and run:
   ```bash
   cd /path/to/your/bci_vr_ws
   rm -rf build devel          # Clean previous artifacts
   catkin_make
   ```
4. **Source the compiled workspace:**
   ```bash
   source devel/setup.bash
   ```

---

## 💡 Key Features & Troubleshooting

- **LSL Streaming:** The container shares the host network stack. This means LSL (Lab Streaming Layer) streams from other PCs or local amplifiers will be detected automatically without complex network mapping.
- **Graphical Support (VR/GUI):** Distrobox automatically handles X11/Wayland integration and GPU passthrough. This provides the hardware acceleration required for real-time EEG visualizers, Neurodraw, or VR interfaces.
- **Persistence:** All changes made within the workspace folder (`src`, `recordings`, etc.) are persistent since the directory is physically located on your host system and only "mounted" inside the container.
- **Isolation:** Using this container prevents "dependency hell" on your main OS, keeping your development environment clean and reproducible.

---

## 📦 Image Contents

The provided Dockerfile automatically sets up a complete BCI development stack:

* **Core:** ROS Noetic Desktop Full (Ubuntu 20.04 base).
* **BCI Framework:** ROSNeuro (installed via official PPA, including `rosneuro_msgs`, `rosneuro_filters`, `rosneuro_acquisition`, etc.).
* **Signal Communication:** LibLSL v1.16.0.
* **System Dependencies:** * `librtfilter`, `libfftw3` (Signal processing).
    * `libqcustomplot`, `libsdl2` (Visualization).
    * `libxdffileio`, `libpcap`, `libbluetooth` (Data acquisition and storage).
* **Python Stack:** `numpy`, `scipy`, `scikit-learn`, `pylsl`.

---

## 🤝 Contributing

If you encounter issues with the container setup or find missing dependencies, please open an Issue or a Pull Request.

