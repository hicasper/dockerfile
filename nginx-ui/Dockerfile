FROM debian:bookworm-slim as builder

ARG TARGETOS
ARG TARGETARCH
ARG TARGETVARIANT
ARG S6_OVERLAY_VERSION=3.2.0.2

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update -y \
    && apt-get install -y --no-install-recommends wget xz-utils logrotate ca-certificates curl \
    && apt-get install -y --no-install-recommends nginx libnginx-mod-http-brotli-* libnginx-mod-http-cache-purge libnginx-mod-http-dav-ext \
       libnginx-mod-http-echo libnginx-mod-http-geoip2 libnginx-mod-http-lua libnginx-mod-http-modsecurity libnginx-mod-http-upstream-fair \
       libnginx-mod-http-xslt-filter libnginx-mod-rtmp libnginx-mod-stream libnginx-mod-stream-geoip2 \
    && apt-get clean &&rm -rf /var/lib/apt/lists/* \
    && mkdir -p /usr/local/etc \
    && mkdir -p /etc/nginx/sites-available \
    && mkdir -p /etc/nginx/sites-enabled \
    && mkdir -p /etc/nginx/streams-available \
    && mkdir -p /etc/nginx/streams-enabled \
    && mkdir -p /etc/nginx/ssl

# copy nginx config
COPY nginx.conf /etc/nginx/nginx.conf
COPY nginx-ui.conf /etc/nginx/conf.d/nginx-ui.conf
COPY default-site /etc/nginx/sites-enabled/default
COPY empty.crt /etc/nginx/ssl/empty.crt
COPY empty.key /etc/nginx/ssl/empty.key

# remove default nginx config
RUN rm -f /etc/nginx/conf.d/default.conf && \
    cp -r /etc/nginx /usr/local/etc/nginx

# copy nginx-ui executable binary
RUN case "${TARGETARCH}/${TARGETVARIANT}" in \
        "amd64/"*) NGXUI_ARCH="linux-64" ;; \
        "arm64/"*) NGXUI_ARCH="linux-arm64-v8a" ;; \
        "arm/v7"*) NGXUI_ARCH="linux-arm32-v7a" ;; \
        "arm/v6"*) NGXUI_ARCH="linux-arm32-v6" ;; \
        "arm/v5"*) NGXUI_ARCH="linux-arm32-v5" ;; \
        *) echo "Unsupported arch: ${TARGETARCH}/${TARGETVARIANT}" && exit 1 ;; \
    esac && \
    curl -fsSL -o /tmp/nginx-ui.tar.gz https://github.com/0xJacky/nginx-ui/releases/latest/download/nginx-ui-${NGXUI_ARCH}.tar.gz && \
    tar -C /tmp -xvzf /tmp/nginx-ui.tar.gz && cp -pf /tmp/nginx-ui /usr/local/bin/nginx-ui

# s6-overlay
RUN case "${TARGETARCH}/${TARGETVARIANT}" in \
        "amd64/"*) S6_ARCH="x86_64" ;; \
        "arm64/"*) S6_ARCH="aarch64" ;; \
        "arm/v7"*) S6_ARCH="arm" ;; \
        "arm/v6"*) S6_ARCH="arm" ;; \
        "arm/v5"*) S6_ARCH="arm" ;; \
        *) echo "Unsupported arch: ${TARGETARCH}/${TARGETVARIANT}" && exit 1 ;; \
    esac && \
    wget -O /tmp/s6-overlay-${S6_ARCH}.tar.xz https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-${S6_ARCH}.tar.xz && \
    tar -C / -Jxpf /tmp/s6-overlay-${S6_ARCH}.tar.xz && \
    wget -O /tmp/s6-overlay-noarch.tar.xz https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz && \
    tar -C / -Jxpf /tmp/s6-overlay-noarch.tar.xz

# register service
COPY nginx /etc/s6-overlay/s6-rc.d/nginx/run
COPY nginx-ui.run /etc/s6-overlay/s6-rc.d/nginx-ui/run
RUN echo 'longrun' > /etc/s6-overlay/s6-rc.d/nginx/type && touch /etc/s6-overlay/s6-rc.d/user/contents.d/nginx && \
    echo 'longrun' > /etc/s6-overlay/s6-rc.d/nginx-ui/type && touch /etc/s6-overlay/s6-rc.d/user/contents.d/nginx-ui

# init config
COPY init-config.up /etc/s6-overlay/s6-rc.d/init-config/up
COPY init-config.sh /etc/s6-overlay/s6-rc.d/init-config/init-config.sh
RUN chmod +x /etc/s6-overlay/s6-rc.d/init-config/init-config.sh && \
    echo 'oneshot' > /etc/s6-overlay/s6-rc.d/init-config/type && \
    touch /etc/s6-overlay/s6-rc.d/user/contents.d/init-config && \
    mkdir -p /etc/s6-overlay/s6-rc.d/nginx/dependencies.d && \
    touch /etc/s6-overlay/s6-rc.d/nginx/dependencies.d/init-config

# recreate access.log and error.log
RUN cat /dev/null > /var/log/nginx/access.log && \
    cat /dev/null > /var/log/nginx/error.log

FROM scratch

COPY --from=builder / /

EXPOSE 80 443

ENTRYPOINT ["/init"]