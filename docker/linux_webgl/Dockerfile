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
    && rm -rf /var/lib/apt/lists/*


# Stage 1: Build
FROM build AS builder

# Define variables for Qt versions
ARG QT_VERSION=5.15
ARG QT_SUBVERSION=5.15.13
ARG QT_WEBPLUGIN_NAME=qtwebglplugin-everywhere-opensource-src

# Build WebGL plugin
WORKDIR /usr/local/src
RUN wget https://download.qt.io/official_releases/qt/${QT_VERSION}/${QT_SUBVERSION}/submodules/${QT_WEBPLUGIN_NAME}-${QT_SUBVERSION}.zip \
    && unzip ${QT_WEBPLUGIN_NAME}-${QT_SUBVERSION}.zip \
    && mv *-${QT_SUBVERSION} qtwebglplugin-everywhere \
    && cd qtwebglplugin-everywhere \
    && qmake \
    && make

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

# Copy WebGL plugin to the appropriate location
COPY --from=builder /usr/local/src/qtwebglplugin-everywhere/plugins/platforms/libqwebgl.so /usr/lib/x86_64-linux-gnu/qt5/plugins/platforms/libqwebgl.so

# Create wrapper script with profile-aware restart handling
# Exit code 42: Profile change - restart QZ
# Exit code 0: Normal exit - stop container
# Other exit codes: Error/crash - stop container
RUN echo '#!/bin/bash\n\
export QZ_CONTAINER_MODE=1\n\
echo "[$(date)] Starting QDomyos-Zwift container wrapper..."\n\
\n\
while true; do\n\
    echo "[$(date)] Starting QDomyos-Zwift with WebGL..."\n\
    /usr/local/bin/qdomyos-zwift -qml -platform webgl:port=8080 2>&1 &\n\
    QZ_PID=$!\n\
    echo "[$(date)] QZ started with PID $QZ_PID"\n\
    \n\
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
done' > /usr/local/bin/qz-wrapper.sh && \
    chmod +x /usr/local/bin/qz-wrapper.sh

# Use wrapper script to handle restarts
CMD ["/usr/local/bin/qz-wrapper.sh"]
