FROM osrf/ros:noetic-desktop-full

ENV DEBIAN_FRONTEND=noninteractive

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

RUN wget https://github.com/sccn/liblsl/releases/download/v1.16.0/liblsl-1.16.0-focal_amd64.deb -O /tmp/lsl.deb && \
    apt-get update && apt-get install -y /tmp/lsl.deb && rm /tmp/lsl.deb

RUN curl -s --compressed -o /etc/apt/sources.list.d/rosneuro.list "https://rosneuro.github.io/ppa/ubuntu/rosneuro.list" && \
    curl -s --compressed "https://rosneuro.github.io/ppa/ubuntu/rosneuro-key.gpg" | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/rosneuro-key.gpg >/dev/null && \
    apt-get update && \
    apt-get install -y ros-noetic-rosneuro-full

RUN echo "yaml https://rosneuro.github.io/ppa/ubuntu/rosdep-rosneuro-packages.yaml" > /etc/ros/rosdep/sources.list.d/50-rosneuro-packages.list && \
    rosdep update

RUN apt-get update && apt-get install -y bash-completion

RUN echo "source /usr/share/bash-completion/bash_completion" >> /etc/bash.bashrc && \
    echo "source /opt/ros/noetic/setup.bash" >> /etc/bash.bashrc && \
    curl -L https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.bash -o /etc/bash_completion.d/git-completion.bash
    
RUN echo "source /opt/ros/noetic/setup.bash" >> /root/.bashrc

WORKDIR /home/paolo/bci_vr_ws
CMD ["bash"]
