ARG BUILD_FROM
FROM $BUILD_FROM

ARG BUILD_ARCH
ARG PACKAGEPLATFORM
ARG BUILD_DATE
ARG VERSION
ARG WIREGUARD_RELEASE
ARG QBITTORRENT_VERSION
ARG QBT_CLI_VERSION

LABEL build_version="Linuxserver.io version:- ${VERSION} Build-date:- ${BUILD_DATE}"
LABEL maintainer="nyok92"

# build environment settings
ENV HOME="/config" \
XDG_CONFIG_HOME="/config" \
XDG_DATA_HOME="/config"

RUN if [ "${BUILD_ARCH}" == "aarch64" ]; then BUILD_ARCH=arm64; \
    elif [ "${BUILD_ARCH}" == "amd64" ]; then BUILD_ARCH=amd64; \
    fi && \
    echo "build arch ${BUILD_ARCH}"

# download package plateform
RUN PACKAGEPLATFORM=$(case ${BUILD_ARCH} in \
        "amd64")        echo "x64"    ;; \
        "aarch64")      echo "arm64"  ;; \
        *)              echo ""       ;; esac) && \
  echo "platform ${PACKAGEPLATFORM} build arch ${BUILD_ARCH}"

## Install Wireguard & dependencies
RUN \
  if [ -z ${WIREGUARD_RELEASE+x} ]; then \
  WIREGUARD_RELEASE=$(curl -sL "http://dl-cdn.alpinelinux.org/alpine/v3.22/main/x86_64/APKINDEX.tar.gz" | tar -xz -C /tmp \
    && awk '/^P:wireguard-tools$/,/V:/' /tmp/APKINDEX | sed -n 2p | sed 's/^V://'); \
  fi && \
  echo "**** install dependencies ****" && \
  apk add --no-cache \
    bc \
    coredns \
    grep \
    iproute2 \
    iptables \
    ip6tables \
    iputils \
    kmod \
    libcap-utils \
    libqrencode-tools \
    net-tools \
    nftables \
    openresolv \
    wireguard-tools==${WIREGUARD_RELEASE} && \
  echo "wireguard" >> /etc/modules && \
  sed -i 's|\[\[ $proto == -4 \]\] && cmd sysctl -q net\.ipv4\.conf\.all\.src_valid_mark=1|[[ $proto == -4 ]] \&\& [[ $(sysctl -n net.ipv4.conf.all.src_valid_mark) != 1 ]] \&\& cmd sysctl -q net.ipv4.conf.all.src_valid_mark=1|' /usr/bin/wg-quick && \
  rm -rf /etc/wireguard && \
  ln -s /config/wg_confs /etc/wireguard && \
  printf "Linuxserver.io version: ${VERSION}\nBuild-date: ${BUILD_DATE}" > /build_version && \
  echo "**** clean up ****" && \
  rm -rf \
    /tmp/*

## Install Qbittorrent & dependencies
# install runtime packages and qbitorrent-cli
RUN \
  echo "**** install packages ****" && \
  apk add --no-cache \
    grep \
    icu-libs \
    p7zip \
    python3 \
    qt6-qtbase-sqlite && \
  if [ -z ${QBITTORRENT_VERSION+x} ]; then \
    QBITTORRENT_VERSION=$(curl -sL "http://dl-cdn.alpinelinux.org/alpine/v3.22/community/x86_64/APKINDEX.tar.gz" | tar -xz -C /tmp \
    && awk '/^P:qbittorrent-nox$/,/V:/' /tmp/APKINDEX | sed -n 2p | sed 's/^V://'); \
  fi && \
  apk add -U --upgrade --no-cache \
    qbittorrent-nox==${QBITTORRENT_VERSION} && \
  echo "***** install qbitorrent-cli ****" && \
  mkdir /qbt && \
  if [ -z ${QBT_CLI_VERSION+x} ]; then \
    QBT_CLI_VERSION=$(curl -sL "https://api.github.com/repos/fedarovich/qbittorrent-cli/releases/latest" \
    | jq -r '. | .tag_name'); \
  fi
RUN echo installing with qbittorent version: "${QBT_CLI_VERSION}" on arch: "${PACKAGEPLATFORM}" && \
  curl -o \
  /tmp/qbt.tar.gz -L \
  "https://github.com/fedarovich/qbittorrent-cli/releases/download/${QBT_CLI_VERSION}/qbt-linux-alpine-${PACKAGEPLATFORM}-net6-${QBT_CLI_VERSION#v}.tar.gz" && \
  tar -xzf \
  /tmp/qbt.tar.gz -C \
  /qbt
RUN printf "Linuxserver.io version: ${VERSION}\nBuild-date: ${BUILD_DATE}" > /build_version && \
  echo "**** cleanup ****" && \
  rm -rf \
  /root/.cache \
  /tmp/*

# Image specific modifications
# hadolint ignore=DL4006
RUN \
    # Set download folder to /share
    sed -i 's|/downloads/|/share/qBittorrent/|g' /defaults/qBittorrent.conf \
    \
    # Remove fixed folders, allows connection to webUI
    && sed '11,13d' /defaults/qBittorrent.conf \
    && echo 'WebUI\HostHeaderValidation=false' >> /defaults/qBittorrent.conf \
    && echo 'WebUI\LocalHostAuth=false' >> /defaults/qBittorrent.conf \
    \
    # Add vuetorrent
    && curl -f -s -S -O -J -L "$(curl -f -s -L https://api.github.com/repos/WDaan/VueTorrent/releases | grep -o "http.*vuetorrent.zip" | head -1)" >/dev/null \
    && unzip -o vuetorrent.zip -d / >/dev/null \
    && rm vuetorrent.zip >/dev/null

# Global LSIO modifications
ADD "https://raw.githubusercontent.com/alexbelgium/hassio-addons/master/.templates/ha_lsio.sh" "/ha_lsio.sh"
ARG CONFIGLOCATION="/config"
RUN chmod 744 /ha_lsio.sh && if grep -qr "lsio" /etc; then /ha_lsio.sh "$CONFIGLOCATION"; fi && rm /ha_lsio.sh

# ADD QBIT_MANAGE
#################

ENV QBM_DOCKER=True
ENV QBT_SKIP_QB_VERSION_CHECK=True
ENV qbt_skip_qb_version_check=true
ENV PYTHONUNBUFFERED=1

# halodint disable=DL3019,SC3010
RUN \
    apk add --no-cache git && \
    git clone --branch master https://github.com/StuffAnThings/qbit_manage.git /qbit_manage && \
    sed -i "/qbittorrent-api/d" /qbit_manage/pyproject.toml && \
    \
    # Install dependencies
    ## Allow python specific arch
    BUILD_ARCH=$(uname -m) && \
    echo "${BUILD_ARCH}" && \
    BUILD_ARCH=${BUILD_ARCH:-x86} && \
    if test "${BUILD_ARCH}" != "*amd64*"; then apk add --no-cache python3-dev; else apk add --no-cache python3; fi && \
    ln -sf python3 /usr/bin/python && \
    apk add --no-cache py3-pip gcc musl-dev && \
    cd /qbit_manage && \
    pip3 install --break-system-packages --user .

# add local files
COPY root/ /

# install unrar
#COPY --from=unrar /usr/bin/unrar-alpine /usr/bin/unrar

VOLUME /config

# ports and volumes
EXPOSE 51820/udp 8080 6881 6881/udp
