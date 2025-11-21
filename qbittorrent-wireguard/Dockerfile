ARG BUILD_FROM
FROM $BUILD_FROM

ARG BUILD_ARCH
ARG PACKAGEPLATFORM
ARG WIREGUARD_RELEASE
ARG QBITTORRENT_VERSION
ARG QBT_CLI_VERSION

RUN if [ "${BUILD_ARCH}" == "aarch64" ]; then BUILD_ARCH=arm64; \
    elif [ "${BUILD_ARCH}" == "amd64" ]; then BUILD_ARCH=amd64; \
    fi && \
    echo "build arch is ${BUILD_ARCH}"

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
  echo "**** clean up ****" && \
  rm -rf \
    /tmp/*

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
RUN if [ "${BUILD_ARCH}" == "aarch64" ]; then PACKAGEPLATFORM=arm64; \
    elif [ "${BUILD_ARCH}" == "amd64" ]; then PACKAGEPLATFORM=x64; \
    fi && \
  echo installing with qbittorent version: "${QBT_CLI_VERSION}" on arch: "${PACKAGEPLATFORM}" && \
  curl -o \
  /tmp/qbt.tar.gz -L \
  "https://github.com/fedarovich/qbittorrent-cli/releases/download/${QBT_CLI_VERSION}/qbt-linux-alpine-${PACKAGEPLATFORM}-net6-${QBT_CLI_VERSION#v}.tar.gz" && \
  tar -xzf \
  /tmp/qbt.tar.gz -C \
  /qbt
RUN echo "**** cleanup ****" && \
  rm -rf \
  /root/.cache \
  /tmp/*

# install unrar
#COPY --from=unrar /usr/bin/unrar-alpine /usr/bin/unrar

# add local files
COPY root/ /

VOLUME /config

# ports and volumes
EXPOSE 51820/udp 8080 6881 6881/udp
