# 1. Base: ROS Noetic Desktop Full
FROM osrf/ros:noetic-desktop-full

# Evita interruzioni durante l'installazione
ENV DEBIAN_FRONTEND=noninteractive

# 2. Strumenti di sistema e dipendenze base
RUN apt-get update && apt-get install -y \
    python3-pip \
    python3-rosdep \
    python3-catkin-tools \
    git \
    wget \
    curl \
    gnupg2 \
    lsb-release \
    libeigen3-dev \
    librtfilter-dev \
    libqcustomplot-dev \
    libbluetooth-dev \
    libpcap-dev \
    libxdffileio-dev \
    libsdl1.2-dev \
    libsdl2-dev \
    libgl1-mesa-dev \
    libfftw3-dev \
    && rm -rf /var/lib/apt/lists/*

# 3. Installazione LSL (fondamentale per BCI)
RUN wget https://github.com/sccn/liblsl/releases/download/v1.16.0/liblsl-1.16.0-focal_amd64.deb -O /tmp/lsl.deb && \
    apt-get update && apt-get install -y /tmp/lsl.deb && rm /tmp/lsl.deb

# 4. Configurazione PPA ROSNeuro e installazione a sistema
RUN curl -s --compressed -o /etc/apt/sources.list.d/rosneuro.list "https://rosneuro.github.io/ppa/ubuntu/rosneuro.list" && \
    curl -s --compressed "https://rosneuro.github.io/ppa/ubuntu/rosneuro-key.gpg" | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/rosneuro-key.gpg >/dev/null && \
    apt-get update && \
    apt-get install -y ros-noetic-rosneuro-full

# 5. Configurazione rosdep per ROSNeuro
RUN echo "yaml https://rosneuro.github.io/ppa/ubuntu/rosdep-rosneuro-packages.yaml" > /etc/ros/rosdep/sources.list.d/50-rosneuro-packages.list && \
    rosdep update

# 6. Setup Ambiente Finale (opzionale ma comodo)
RUN echo "source /opt/ros/noetic/setup.bash" >> /root/.bashrc

WORKDIR /home/paolo/bci_vr_ws
CMD ["bash"]
