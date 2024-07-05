ARG DEBIAN_FRONTEND=noninteractive

ARG BASE_IMAGE=debian:11
ARG SLIM_BASE=debian:11-slim

FROM ${BASE_IMAGE} AS base

FROM --platform=${BUILDPLATFORM} debian:11 AS base_host

FROM ${SLIM_BASE} AS slim-base

FROM slim-base AS wget
ARG DEBIAN_FRONTEND
RUN apt-get update \
    && apt-get install -y wget xz-utils \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /rootfs

FROM base AS nginx
ARG DEBIAN_FRONTEND
ENV CCACHE_DIR /root/.ccache
ENV CCACHE_MAXSIZE 2G

RUN --mount=type=tmpfs,target=/tmp --mount=type=tmpfs,target=/var/cache/apt \
    --mount=type=bind,source=docker/main/build_nginx.sh,target=/deps/build_nginx.sh \
    --mount=type=cache,target=/root/.ccache \
    /deps/build_nginx.sh

FROM scratch AS go2rtc
ARG TARGETARCH
WORKDIR /rootfs/usr/local/go2rtc/bin
ADD --link --chmod=755 "https://github.com/AlexxIT/go2rtc/releases/download/v1.9.2/go2rtc_linux_${TARGETARCH}" go2rtc

FROM wget AS tempio
ARG TARGETARCH
RUN --mount=type=bind,source=docker/main/install_tempio.sh,target=/deps/install_tempio.sh \
    /deps/install_tempio.sh

FROM base_host AS ov-converter
ARG DEBIAN_FRONTEND

COPY docker/main/requirements-ov.txt /requirements-ov.txt
RUN apt-get -qq update \
    && apt-get -qq install -y wget python3 python3-dev python3-distutils gcc pkg-config libhdf5-dev \
    && wget -q https://bootstrap.pypa.io/get-pip.py -O get-pip.py \
    && python3 get-pip.py "pip" \
    && pip install -r /requirements-ov.txt

RUN --mount=type=bind,source=docker/main/build_ov_model.py,target=/build_ov_model.py \
    mkdir /models && cd /models \
    && wget http://download.tensorflow.org/models/object_detection/ssdlite_mobilenet_v2_coco_2018_05_09.tar.gz \
    && tar -xvf ssdlite_mobilenet_v2_coco_2018_05_09.tar.gz \
    && python3 /build_ov_model.py

FROM wget AS models

RUN wget -qO edgetpu_model.tflite https://github.com/google-coral/test_data/raw/release-frogfish/ssdlite_mobiledet_coco_qat_postprocess_edgetpu.tflite
RUN wget -qO cpu_model.tflite https://github.com/google-coral/test_data/raw/release-frogfish/ssdlite_mobiledet_coco_qat_postprocess.tflite
COPY labelmap.txt .

COPY --from=ov-converter /models/ssdlite_mobilenet_v2.xml openvino-model/
COPY --from=ov-converter /models/ssdlite_mobilenet_v2.bin openvino-model/
RUN wget -q https://github.com/openvinotoolkit/open_model_zoo/raw/master/data/dataset_classes/coco_91cl_bkgr.txt -O openvino-model/coco_91cl_bkgr.txt && \
    sed -i 's/truck/car/g' openvino-model/coco_91cl_bkgr.txt

RUN wget -qO - https://www.kaggle.com/api/v1/models/google/yamnet/tfLite/classification-tflite/1/download | tar xvz && mv 1.tflite cpu_audio_model.tflite 
COPY audio-labelmap.txt .


FROM wget AS s6-overlay
ARG TARGETARCH
RUN --mount=type=bind,source=docker/main/install_s6_overlay.sh,target=/deps/install_s6_overlay.sh \
    /deps/install_s6_overlay.sh


FROM base AS wheels
ARG DEBIAN_FRONTEND
ARG TARGETARCH

RUN apt-get -qq update \
    && apt-get -qq install -y \
    apt-transport-https \
    gnupg \
    wget \
    && wget -qO- "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0xA4285295FC7B1A81600062A9605C66F00D6C9793" | \
    gpg --dearmor > /usr/share/keyrings/debian-archive-bullseye-stable.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/debian-archive-bullseye-stable.gpg] http://deb.debian.org/debian bullseye main contrib non-free" | \
    tee /etc/apt/sources.list.d/debian-bullseye-nonfree.list \
    && apt-get -qq update \
    && apt-get -qq install -y \
    python3.9 \
    python3.9-dev \
    build-essential cmake git pkg-config libgtk-3-dev \
    libavcodec-dev libavformat-dev libswscale-dev libv4l-dev \
    libxvidcore-dev libx264-dev libjpeg-dev libpng-dev libtiff-dev \
    gfortran openexr libatlas-base-dev libssl-dev\
    libtbb2 libtbb-dev libdc1394-22-dev libopenexr-dev \
    libgstreamer-plugins-base1.0-dev libgstreamer1.0-dev \
    gcc gfortran libopenblas-dev liblapack-dev && \
    rm -rf /var/lib/apt/lists/*

RUN update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.9 1

RUN wget -q https://bootstrap.pypa.io/get-pip.py -O get-pip.py \
    && python3 get-pip.py "pip"

COPY docker/main/requirements.txt /requirements.txt
RUN pip3 install -r /requirements.txt

COPY docker/main/requirements-wheels.txt /requirements-wheels.txt
RUN pip3 wheel --wheel-dir=/wheels -r /requirements-wheels.txt


FROM scratch AS deps-rootfs
COPY --from=nginx /usr/local/nginx/ /usr/local/nginx/
COPY --from=go2rtc /rootfs/ /
COPY --from=tempio /rootfs/ /
COPY --from=s6-overlay /rootfs/ /
COPY --from=models /rootfs/ /
COPY docker/main/rootfs/ /

FROM slim-base AS deps
ARG TARGETARCH

ARG DEBIAN_FRONTEND
ARG APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=DontWarn

ENV NVIDIA_VISIBLE_DEVICES=all
ENV NVIDIA_DRIVER_CAPABILITIES="compute,video,utility"

ENV PATH="/usr/lib/btbn-ffmpeg/bin:/usr/local/go2rtc/bin:/usr/local/tempio/bin:/usr/local/nginx/sbin:${PATH}"

RUN --mount=type=bind,source=docker/main/install_deps.sh,target=/deps/install_deps.sh \
    /deps/install_deps.sh

RUN --mount=type=bind,from=wheels,source=/wheels,target=/deps/wheels \
    python3 -m pip install --upgrade pip && \
    pip3 install -U /deps/wheels/*.whl

COPY --from=deps-rootfs / /

EXPOSE 5000
EXPOSE 8554
EXPOSE 8555/tcp 8555/udp

ENV S6_LOGGING_SCRIPT="T 1 n0 s10000000 T"
ENV S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0

ENTRYPOINT ["/init"]
CMD []

HEALTHCHECK --start-period=120s --start-interval=5s --interval=15s --timeout=5s --retries=3 \
    CMD curl --fail --silent --show-error http://127.0.0.1:5000/api/version || exit 1

FROM deps AS devcontainer

COPY docker/main/fake_frigate_run /etc/s6-overlay/s6-rc.d/frigate/run

RUN mkdir -p /opt/frigate \
    && ln -svf /workspace/frigate/frigate /opt/frigate/frigate

RUN curl -SLO https://deb.nodesource.com/nsolid_setup_deb.sh && \
    chmod 500 nsolid_setup_deb.sh && \
    ./nsolid_setup_deb.sh 20 && \
    apt-get install nodejs -y \
    && rm -rf /var/lib/apt/lists/* \
    && npm install -g npm@10

WORKDIR /workspace/frigate

RUN apt-get update \
    && apt-get install make -y \
    && rm -rf /var/lib/apt/lists/*

RUN --mount=type=bind,source=./docker/main/requirements-dev.txt,target=/workspace/frigate/requirements-dev.txt \
    pip3 install -r requirements-dev.txt

HEALTHCHECK NONE

CMD ["sleep", "infinity"]


FROM --platform=$BUILDPLATFORM node:20 AS web-build

WORKDIR /work
COPY web/package.json web/package-lock.json ./
RUN npm install

COPY web/ ./
RUN npm run build \
    && mv dist/BASE_PATH/monacoeditorwork/* dist/assets/ \
    && rm -rf dist/BASE_PATH

FROM scratch AS rootfs

WORKDIR /opt/frigate/
COPY frigate frigate/
COPY migrations migrations/
COPY --from=web-build /work/dist/ web/

FROM deps AS frigate

WORKDIR /opt/frigate/
COPY --from=rootfs / /
