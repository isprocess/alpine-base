# syntax=docker/dockerfile:1.7

# -----------------------------------------------------------------------------
# Reproducible base image:
# - Default: alpine:3.23
# - Pin digest by passing: --build-arg ALPINE_IMAGE=alpine:3.23@sha256:...
# -----------------------------------------------------------------------------
ARG ALPINE_IMAGE=alpine:3.23

# =========================
# builder: download + verify + unpack (curl/tar only here)
# =========================
FROM ${ALPINE_IMAGE} AS builder

ARG TARGETPLATFORM
ARG TARGETARCH
ARG VERIFY=0

# Must be pinned, no "latest"
ARG ZULU_JRE8_VERSION=8.92.0.21-ca-jre8.0.482

# Keep sha256 ARG structure (fill for strong reproducibility)
ARG ZULU_SHA256_AMD64=""
ARG ZULU_SHA256_ARM64=""

RUN set -eux; \
    apk add --no-cache ca-certificates curl tar; \
    update-ca-certificates; \
    base="https://cdn.azul.com/zulu/bin"; \
    \
    # 1) Prefer BuildKit-provided arch (buildx)
    arch="${TARGETARCH:-}"; \
    if [ -z "${arch}" ]; then \
      case "${TARGETPLATFORM:-}" in \
        "linux/amd64") arch="amd64" ;; \
        "linux/arm64") arch="arm64" ;; \
      esac; \
    fi; \
    \
    # 2) Fallback for plain `docker build` when TARGET* not present
    if [ -z "${arch}" ]; then \
      m="$(uname -m)"; \
      case "${m}" in \
        x86_64) arch="amd64" ;; \
        aarch64|arm64) arch="arm64" ;; \
        *) echo >&2 "ERROR: unsupported arch from uname -m: ${m}"; exit 22 ;; \
      esac; \
    fi; \
    \
    case "${arch}" in \
      amd64) file="zulu${ZULU_JRE8_VERSION}-linux_musl_x64.tar.gz"; sha="${ZULU_SHA256_AMD64}" ;; \
      arm64) file="zulu${ZULU_JRE8_VERSION}-linux_musl_aarch64.tar.gz"; sha="${ZULU_SHA256_ARM64}" ;; \
      *) echo >&2 "ERROR: unsupported arch=${arch}"; exit 22 ;; \
    esac; \
    \
    url="${base}/${file}"; \
    echo "Downloading: ${url}"; \
    curl -fsSL --retry 3 --retry-delay 1 --connect-timeout 10 --max-time 300 -o /tmp/zulu.tgz "${url}"; \
    \
    if [ -n "${sha}" ]; then \
      echo "${sha}  /tmp/zulu.tgz" | sha256sum -c -; \
    else \
      echo >&2 "WARN: sha256 not provided for ${arch}; integrity check downgraded (HTTPS + fail-fast)."; \
      echo "${file}" | grep -F "${ZULU_JRE8_VERSION}" >/dev/null; \
    fi; \
    \
    mkdir -p /opt/zulu; \
    tar -xzf /tmp/zulu.tgz -C /opt/zulu --strip-components=1; \
    rm -f /tmp/zulu.tgz; \
    \
    # Conservative slimming: remove non-runtime extras (reduce scan surface)
    rm -rf /opt/zulu/demo /opt/zulu/sample /opt/zulu/man || true; \
    find /opt/zulu -maxdepth 6 -type f -name "*.zip" -print -delete || true; \
    \
    if [ "${VERIFY}" = "1" ]; then \
      /opt/zulu/bin/java -version; \
    fi

# =========================
# final: runtime only (+ fontconfig, non-root, scan-friendly, web-default JVM tuning)
# =========================
FROM ${ALPINE_IMAGE}

ARG INSTALL_BASH=1
ARG VERIFY=0

# Web-service defaults (normal memory containers >=200MB):
# - Initial/Max are used; MinRAMPercentage is only relevant for small memory containers (<200MB).
ARG JAVA_INITIAL_RAM_PERCENT=50.0
ARG JAVA_MAX_RAM_PERCENT=75.0

# Optional for small-memory containers (<200MB). Empty means "do not set".
ARG JAVA_MIN_RAM_PERCENT=""

ENV TZ=Asia/Shanghai \
    LANG=zh_CN.UTF-8 \
    LANGUAGE=zh_CN:zh \
    LC_ALL=zh_CN.UTF-8 \
    JAVA_HOME=/usr/local/zulu \
    PATH=/usr/local/zulu/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Base JVM options:
# - Container aware (8u191+): UseContainerSupport
# - OOM fast-exit: ExitOnOutOfMemoryError
# - Web-service heap sizing: Initial/Max RAM%
# - MinRAMPercentage is intentionally NOT set by default (only for <200MB containers).
ENV JAVA_TOOL_OPTIONS="-Dfile.encoding=UTF-8 -Duser.timezone=Asia/Shanghai -Djava.security.egd=file:/dev/urandom \
-XX:+UseContainerSupport -XX:+ExitOnOutOfMemoryError \
-XX:InitialRAMPercentage=${JAVA_INITIAL_RAM_PERCENT} -XX:MaxRAMPercentage=${JAVA_MAX_RAM_PERCENT}"

# final stage: single RUN for runtime pkgs + tz + scripts + fontconfig config + non-root + wrapper + arg sanity checks (+ optional MinRAMPercentage)
RUN set -eux; \
    \
    # Recommended sanity: Initial should not exceed Max (JVM will clamp, but keep config predictable)
    awk 'BEGIN{ \
      init='${JAVA_INITIAL_RAM_PERCENT}'; max='${JAVA_MAX_RAM_PERCENT}'; \
      if (init > max) exit 22; \
    }' >/dev/null; \
    \
    apk add --no-cache \
      tzdata \
      ca-certificates \
      musl-locales \
      musl-locales-lang \
      libstdc++ \
      libgcc \
      fontconfig; \
    if [ "${INSTALL_BASH}" = "1" ]; then apk add --no-cache bash; fi; \
    update-ca-certificates; \
    \
    # tzdata effective
    cp "/usr/share/zoneinfo/${TZ}" /etc/localtime; \
    echo "${TZ}" > /etc/timezone; \
    \
    # scripts: now / ll (busybox-compatible)
    printf '%s\n' '#!/bin/sh' 'date "+%Y-%m-%d %H:%M:%S %z"' > /usr/local/bin/now; \
    chmod +x /usr/local/bin/now; \
    printf '%s\n' '#!/bin/sh' 'ls -al "$@"' > /usr/local/bin/ll; \
    chmod +x /usr/local/bin/ll; \
    \
    # Ensure fontconfig scans mounted font dirs reliably:
    mkdir -p /etc/fonts/conf.d; \
    printf '%s\n' \
      '<?xml version="1.0"?>' \
      '<!DOCTYPE fontconfig SYSTEM "fonts.dtd">' \
      '<fontconfig>' \
      '  <dir>/fonts</dir>' \
      '  <dir>/usr/share/fonts/custom</dir>' \
      '</fontconfig>' \
      > /etc/fonts/conf.d/99-local-fonts.conf; \
    \
    # Non-root runtime (scan-friendly + safer default):
    addgroup -S app; \
    adduser  -S -G app -h /app -s /sbin/nologin app; \
    mkdir -p /app /tmp /var/cache/fontconfig; \
    chown -R app:app /app /tmp /var/cache/fontconfig; \
    chmod 1777 /tmp; \
    \
    # Persist JAVA_TOOL_OPTIONS (+ optional MinRAMPercentage) for interactive shells.
    # Note: MinRAMPercentage is only meaningful for very small memory containers (<200MB).
    mkdir -p /etc/profile.d; \
    if [ -n "${JAVA_MIN_RAM_PERCENT}" ]; then \
      printf '%s\n' "export JAVA_TOOL_OPTIONS='${JAVA_TOOL_OPTIONS} -XX:MinRAMPercentage=${JAVA_MIN_RAM_PERCENT}'" > /etc/profile.d/java_tool_options.sh; \
    else \
      printf '%s\n' "export JAVA_TOOL_OPTIONS='${JAVA_TOOL_OPTIONS}'" > /etc/profile.d/java_tool_options.sh; \
    fi; \
    \
    # Default shell wrapper:
    # - refresh font cache if any font dir exists (runtime only)
    # - exec bash if present, else sh
    printf '%s\n' \
      '#!/bin/sh' \
      'set -e' \
      '' \
      '# Source profile to apply persisted JAVA_TOOL_OPTIONS for interactive entry.' \
      '[ -f /etc/profile.d/java_tool_options.sh ] && . /etc/profile.d/java_tool_options.sh || true' \
      '' \
      '# If fonts are mounted, refresh cache for consistent rendering (runtime only).' \
      'if [ -d /fonts ] || [ -d /usr/share/fonts/custom ] || [ -d /usr/share/fonts ] || [ -d /usr/local/share/fonts ]; then' \
      '  fc-cache -f >/dev/null 2>&1 || true' \
      'fi' \
      '' \
      'if command -v bash >/dev/null 2>&1; then exec bash; fi' \
      'exec sh' \
      > /usr/local/bin/_default_shell; \
    chmod +x /usr/local/bin/_default_shell

# Only allowed JRE copy from builder
COPY --from=builder /opt/zulu /usr/local/zulu

# Drop privileges by default
USER app
WORKDIR /app

# Optional runtime verification (stdout only)
RUN set -eux; \
    if [ "${VERIFY}" = "1" ]; then \
      echo "TZ=${TZ}"; echo "LANG=${LANG}"; echo "LC_ALL=${LC_ALL}"; \
      echo "JAVA_TOOL_OPTIONS=${JAVA_TOOL_OPTIONS}"; \
      if [ -n "${JAVA_MIN_RAM_PERCENT}" ]; then echo "JAVA_MIN_RAM_PERCENT=${JAVA_MIN_RAM_PERCENT}"; fi; \
      date "+%Y-%m-%d %H:%M:%S %z"; \
      java -version; \
      /usr/local/bin/now; \
      /usr/local/bin/ll /; \
      command -v fc-cache; \
      fc-list >/dev/null 2>&1 || true; \
    fi

CMD ["/usr/local/bin/_default_shell"]
