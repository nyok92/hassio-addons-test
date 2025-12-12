ARG BUILD_FROM
FROM $BUILD_FROM

ARG BUILD_ARCH
ARG PACKAGEPLATFORM
ARG WIREGUARD_RELEASE
ARG QBITTORRENT_VERSION
ARG QBT_CLI_VERSION

ARG LSIOWN_VERSION="v1"
ADD --chmod=755 "https://raw.githubusercontent.com/linuxserver/docker-mods/mod-scripts/lsiown.${LSIOWN_VERSION}" "/usr/bin/lsiown"

RUN \
  echo "**** install runtime packages ****" && \
  apk add --no-cache \
    alpine-release \
    bash \
    ca-certificates \
    catatonit \
    coreutils \
    curl \
    findutils \
    jq \
    netcat-openbsd \
    procps-ng \
    shadow \
    tzdata && \
  echo "**** create abc user and make our folders ****" && \
  groupmod -g 1000 users && \
  useradd -u 911 -U -d /config -s /bin/false abc && \
  usermod -G users abc && \
  mkdir -p \
    /app \
    /config \
    /defaults \
    /lsiopy && \
  echo "**** cleanup ****" && \
  rm -rf \
    /tmp/*

############################################ Linux wireguard repo #################################
## Install Wireguard & dependencies
RUN \
  if [ -z ${WIREGUARD_RELEASE+x} ]; then \
  WIREGUARD_RELEASE=$(curl -sL "http://dl-cdn.alpinelinux.org/alpine/v3.23/main/x86_64/APKINDEX.tar.gz" | tar -xz -C /tmp \
    && awk '/^P:wireguard-tools$/,/V:/' /tmp/APKINDEX | sed -n 2p | sed 's/^V://'); \
  fi && \
  echo "**** install dependencies ****" && \
  apk add --no-cache \
    nginx \
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
  echo "**** clean up ****" && \
  rm -rf \
    /etc/nginx \
    /tmp/*
############################################ Linux docker qbittorrent repo #################################
# build environment settings
ENV HOME="/config" \
XDG_CONFIG_HOME="/config" \
XDG_DATA_HOME="/config"

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
    QBITTORRENT_VERSION=$(curl -sL "http://dl-cdn.alpinelinux.org/alpine/v3.23/community/x86_64/APKINDEX.tar.gz" | tar -xz -C /tmp \
    && awk '/^P:qbittorrent-nox$/,/V:/' /tmp/APKINDEX | sed -n 2p | sed 's/^V://'); \
  fi && \
  apk add -U --upgrade --no-cache \
    qbittorrent-nox==${QBITTORRENT_VERSION}

RUN echo "***** install qbitorrent-cli ****" && \
  mkdir /qbt && \
  if [ -z ${QBT_CLI_VERSION+x} ]; then \
    QBT_CLI_VERSION=$(curl -sL "https://api.github.com/repos/fedarovich/qbittorrent-cli/releases/latest" \
    | jq -r '. | .tag_name'); \
  fi && \
  if [ "${BUILD_ARCH}" == "aarch64" ]; then PACKAGEPLATFORM=arm64; \
  elif [ "${BUILD_ARCH}" == "amd64" ]; then PACKAGEPLATFORM=x64; \
  fi && \
  echo installing with qbittorent "${QBT_CLI_VERSION}" on "${PACKAGEPLATFORM}" && \
  curl -o \
  /tmp/qbt.tar.gz -L \
  "https://github.com/fedarovich/qbittorrent-cli/releases/download/${QBT_CLI_VERSION}/qbt-linux-alpine-${PACKAGEPLATFORM}-net6-${QBT_CLI_VERSION#v}.tar.gz" && \
  tar xzvf \
  /tmp/qbt.tar.gz -C \
  /qbt
RUN echo "**** cleanup ****" && \
  rm -rf \
  /root/.cache \
  /tmp/*

# add local files
COPY root/ /

RUN chmod a+x /etc/s6-overlay/s6-rc.d/init-wireguard-confs/run \
    /etc/s6-overlay/s6-rc.d/init-wireguard-module/run \
    /etc/s6-overlay/s6-rc.d/svc-wireguard/run \
    /etc/s6-overlay/s6-rc.d/svc-wireguard/finish \
    /etc/s6-overlay/s6-rc.d/init-qbittorrent-config/run \
    /etc/s6-overlay/s6-rc.d/svc-qbittorrent/run \
    /etc/s6-overlay/s6-rc.d/svc-coredns/run \
    /etc/s6-overlay/s6-rc.d/svc-nginx/run \
    /etc/s6-overlay/s6-rc.d/svc-nginx/finish \
    /etc/s6-overlay/s6-rc.d/init-nginx-config/run \
    /etc/s6-overlay/s6-rc.d/init-folders/run \
    /etc/s6-overlay/s6-rc.d/init-openvpn/run \
    /etc/s6-overlay/s6-rc.d/init-qbit-manage/run \
    /etc/s6-overlay/s6-rc.d/init-monitor/run \
    /etc/s6-overlay/s6-rc.d/svc-monitor/run \
    /etc/s6-overlay/s6-rc.d/init-migrations/run \
    /etc/s6-overlay/s6-rc.d/init-envfile/run \
    /etc/s6-overlay/s6-rc.d/init-adduser/run \
    /etc/s6-overlay/s6-rc.d/init-crontab-config/run \
    /etc/s6-overlay/s6-rc.d/init-custom-files/run \
    /etc/s6-overlay/s6-rc.d/init-device-perms/run \
    /etc/s6-overlay/s6-rc.d/svc-cron/run

VOLUME /config

# ports and volumes
EXPOSE 51820/udp 8080 6881 6881/udp
