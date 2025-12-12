ARG BUILD_FROM
FROM $BUILD_FROM

ARG BUILD_ARCH
ARG PACKAGEPLATFORM
ARG WIREGUARD_RELEASE
ARG QBITTORRENT_VERSION
ARG QBT_CLI_VERSION

#RUN if [ "${BUILD_ARCH}" == "aarch64" ]; then BUILD_ARCH=arm64; \
#    elif [ "${BUILD_ARCH}" == "amd64" ]; then BUILD_ARCH=amd64; \
#    fi && \
#    echo "build arch is ${BUILD_ARCH}"
############################################ Linux wireguard repo #################################
## Install Wireguard & dependencies
RUN \
  if [ -z ${WIREGUARD_RELEASE+x} ]; then \
  WIREGUARD_RELEASE=$(curl -sL "http://dl-cdn.alpinelinux.org/alpine/v3.22/main/x86_64/APKINDEX.tar.gz" | tar -xz -C /tmp \
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
    QBITTORRENT_VERSION=$(curl -sL "http://dl-cdn.alpinelinux.org/alpine/v3.22/community/x86_64/APKINDEX.tar.gz" | tar -xz -C /tmp \
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
  tar -xzf \
  /tmp/qbt.tar.gz -C \
  /qbt
RUN echo "**** cleanup ****" && \
  rm -rf \
  /root/.cache \
  /tmp/*

# add local files
COPY root/ /

############################################ Alexbelgium qbittorrent repo #################################

# Set S6 wait time
ENV S6_CMD_WAIT_FOR_SERVICES=1 \
    S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0 \
    S6_SERVICES_GRACETIME=0

# Environment variables
ENV \
    DEBIAN_FRONTEND="noninteractive"\
    LANG="C.UTF-8" \
    PS1="$(whoami)@$(hostname):$(pwd)$ " \
    S6_BEHAVIOUR_IF_STAGE2_FAILS=2 \
    S6_CMD_WAIT_FOR_SERVICES=1 \
    TERM="xterm-256color" \
    HOME="/config" \
    XDG_CONFIG_HOME="/config" \
    XDG_DATA_HOME="/config"

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

# Uses /bin for compatibility purposes
# hadolint ignore=DL4005
RUN if [ ! -f /bin/sh ] && [ -f /usr/bin/sh ]; then ln -s /usr/bin/sh /bin/sh; fi && \
    if [ ! -f /bin/bash ] && [ -f /usr/bin/bash ]; then ln -s /usr/bin/bash /bin/bash; fi

# Modules
#ARG MODULES="00-banner.sh 01-custom_script.sh 00-local_mounts.sh 00-smb_mounts.sh 90-dns_set.sh"

# Automatic modules download
#ADD "https://raw.githubusercontent.com/alexbelgium/hassio-addons/master/.templates/ha_automodules.sh" "/ha_automodules.sh"
#RUN chmod 744 /ha_automodules.sh && /ha_automodules.sh "$MODULES" && rm /ha_automodules.sh

#RUN if [ -f /etc/s6-overlay/s6-rc.d/init-adduser ]; then for SCRIPTS in /etc/cont-init.d/*; do [ -e "$SCRIPTS" ] || continue \
#    && mkdir -p /abscripts && sed -i 's|/usr/bin/with-contenv bashio|/usr/bin/env bashio|g' "$SCRIPTS" \
#    && SCRIPTSNAME="$(basename "$SCRIPTS")" && mkdir /etc/s6-overlay/s6-rc.d/"$SCRIPTSNAME" && mv "$SCRIPTS" /etc/s6-overlay/s6-rc.d/"$SCRIPTSNAME"/run \
#    && echo "oneshot" > /etc/s6-overlay/s6-rc.d/"$SCRIPTSNAME"/type && echo "/etc/s6-overlay/s6-rc.d/$SCRIPTSNAME/run" > /etc/s6-overlay/s6-rc.d/"$SCRIPTSNAME"/up \
#    && chmod a+x /etc/s6-overlay/s6-rc.d/$SCRIPTSNAME/* ; done; fi

# Manual apps
ARG PACKAGES=""

# Automatic apps & bashio
ADD "https://raw.githubusercontent.com/alexbelgium/hassio-addons/master/.templates/ha_autoapps.sh" "/ha_autoapps.sh"
RUN chmod 744 /ha_autoapps.sh && /ha_autoapps.sh "$PACKAGES" && rm /ha_autoapps.sh

# Switch scripts from s6 v2 to v3
#RUN if [ -f /etc/s6-overlay/s6-rc.d/init-custom-file/run ]; then mkdir "/custom-cont-init.d" && for SCRIPTS in /etc/cont-init.d/*; do [ -e "$SCRIPTS" ] || continue \
#    && mv "$SCRIPTS" /custom-cont-init.d && chmod a+x /custom-cont-init.d/*; done; fi

################
# 4 Entrypoint #
################

#RUN chmod +x /usr/local/bin/* /usr/local/sbin/*

# Add entrypoint
ENV S6_STAGE2_HOOK=/ha_entrypoint.sh
ADD "https://raw.githubusercontent.com/alexbelgium/hassio-addons/master/.templates/ha_entrypoint.sh" "/ha_entrypoint.sh"

# Entrypoint modifications
ADD "https://raw.githubusercontent.com/alexbelgium/hassio-addons/master/.templates/ha_entrypoint_modif.sh" "/ha_entrypoint_modif.sh"
RUN chmod 777 /ha_entrypoint.sh /ha_entrypoint_modif.sh && /ha_entrypoint_modif.sh && rm /ha_entrypoint_modif.sh

# Standalone bashio command
ADD "https://raw.githubusercontent.com/alexbelgium/hassio-addons/master/.templates/bashio-standalone.sh" "/.bashio-standalone.sh"
RUN chmod 777 /.bashio-standalone.sh


# Avoid spamming logs
# hadolint ignore=SC2016
RUN \
    # Handle Nginx configuration
    if [ -f /etc/nginx/nginx.conf ]; then \
        awk '/http \{/{print; print "map $http_user_agent $dontlog {\n  default 0;\n  \"~*HealthCheck\" 1;\n}\naccess_log /var/log/nginx/access.log combined if=$dontlog;"; next}1' /etc/nginx/nginx.conf > /etc/nginx/nginx.conf.new && \
        mv /etc/nginx/nginx.conf.new /etc/nginx/nginx.conf; \
    fi

# install unrar
#COPY --from=unrar /usr/bin/unrar-alpine /usr/bin/unrar

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
    /etc/s6-overlay/s6-rc.d/svc-monitor/run

VOLUME /config

# ports and volumes
EXPOSE 51820/udp 8080 6881 6881/udp
