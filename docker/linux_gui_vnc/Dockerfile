# Define build image
FROM ubuntu:latest AS build

# Install essential build dependencies
ARG DEBIAN_FRONTEND=noninteractive
RUN apt update && apt upgrade -y \
    && apt install --no-install-recommends -y \
        git \
        ca-certificates \
        qtquickcontrols2-5-dev \
        qtconnectivity5-dev \
        qtbase5-private-dev \
        qtpositioning5-dev \
        libqt5charts5-dev \
        libqt5networkauth5-dev \
        libqt5websockets5-dev \
        qml-module* \
        libqt5texttospeech5-dev \
        qtlocation5-dev \
        qtmultimedia5-dev \
        g++ \
        make \
        wget \
        unzip \
    && rm -rf /var/lib/apt/lists/*


# Define runtime image        
FROM ubuntu:latest AS runtime

# Install essential runtime dependencies
ARG DEBIAN_FRONTEND=noninteractive
RUN apt update && apt upgrade -y \
    && apt install --no-install-recommends -y \
        libqt5bluetooth5 \
        libqt5widgets5 \
        libqt5positioning5 \
        libqt5xml5 \
        libqt5charts5 \
        qt5-assistant \
        libqt5networkauth5 \
        libqt5websockets5 \
        qml-module* \
        libqt5texttospeech5 \
        libqt5location5-plugins \
        libqt5multimediawidgets5 \
        libqt5multimedia5-plugins \
        libqt5multimedia5 \
        qml-module-qtquick-controls2 \
        libqt5location5 \
        bluez \
        dbus \
        tigervnc-standalone-server \
        tigervnc-tools \
        libgl1-mesa-dri \
        xfonts-base \
        x11-xserver-utils \
        tigervnc-common \
        net-tools \
    && rm -rf /var/lib/apt/lists/*


# Stage 1: Build
FROM build AS builder

# Clone the project and build it
WORKDIR /usr/local/src
RUN git clone --recursive https://github.com/cagnulein/qdomyos-zwift.git
WORKDIR /usr/local/src/qdomyos-zwift
RUN git submodule update --init src/smtpclient/ \
    && git submodule update --init src/qmdnsengine/ \
    && git submodule update --init tst/googletest/
WORKDIR /usr/local/src/qdomyos-zwift/src
RUN qmake qdomyos-zwift.pro \
    && make -j4


# Stage 2: Runtime
FROM runtime

# Copy the built binary to /usr/local/bin
COPY --from=builder /usr/local/src/qdomyos-zwift/src/qdomyos-zwift /usr/local/bin/qdomyos-zwift

# VNC configuration
RUN mkdir -p ~/.vnc && \
    echo "securepassword" | vncpasswd -f > ~/.vnc/passwd && \
    chmod 600 ~/.vnc/passwd

# .Xauthority configuration
RUN touch /root/.Xauthority
ENV DISPLAY=:99

# Create xstartup script with profile-aware restart handling
# Exit code 42: Profile change - restart QZ
# Exit code 0: Normal exit - stop container
# Other exit codes: Error/crash - stop container
RUN mkdir -p ~/.vnc && \
    echo '#!/bin/bash\n\
# Redirect output to Docker logs\n\
exec > /proc/1/fd/1 2>&1\n\
\n\
export QZ_CONTAINER_MODE=1\n\
export DISPLAY=:99\n\
echo "[$(date)] Starting QDomyos-Zwift container wrapper..."\n\
echo "QZ_CONTAINER_MODE=$QZ_CONTAINER_MODE"\n\
\n\
# Wait for X server to be ready\n\
sleep 2\n\
\n\
while true; do\n\
    echo "[$(date)] Starting QDomyos-Zwift..."\n\
    /usr/local/bin/qdomyos-zwift 2>&1 &\n\
    QZ_PID=$!\n\
    echo "[$(date)] QZ started with PID $QZ_PID"\n\
    \n\
    # Wait for the process\n\
    wait $QZ_PID\n\
    EXIT_CODE=$?\n\
    \n\
    echo "[$(date)] QZ exited with code $EXIT_CODE"\n\
    \n\
    # Exit code 42: Profile change requested - restart\n\
    if [ $EXIT_CODE -eq 42 ]; then\n\
        echo "[$(date)] Profile change detected (exit code 42), restarting QZ in 2 seconds..."\n\
        sleep 2\n\
        continue\n\
    fi\n\
    \n\
    # Exit code 0: Normal exit - stop container\n\
    if [ $EXIT_CODE -eq 0 ]; then\n\
        echo "[$(date)] Normal exit (code 0), stopping container..."\n\
        exit 0\n\
    fi\n\
    \n\
    # Any other exit code: Error/crash - stop container\n\
    echo "[$(date)] Error exit (code $EXIT_CODE), stopping container..."\n\
    exit $EXIT_CODE\n\
done' > ~/.vnc/xstartup && \
    chmod +x ~/.vnc/xstartup

# Start VNC in foreground with QZ wrapper script
CMD ["vncserver", ":99", "-depth", "24", "-localhost", "no", "-fg"]

