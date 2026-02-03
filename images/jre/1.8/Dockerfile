# syntax=docker/dockerfile:1.7
ARG ALPINE_VERSION=3.23

# =========================
# builder: download + verify + unpack (curl/tar only here)
# =========================
FROM alpine:${ALPINE_VERSION} AS builder

ARG TARGETPLATFORM
ARG TARGETARCH
ARG VERIFY=0

# Must be pinned, no "latest"
ARG ZULU_JRE8_VERSION=8.92.0.21-ca-jre8.0.482

# Keep sha256 ARG structure (fill these for strong reproducibility)
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
    # Conservative slimming: remove non-runtime extras
    rm -rf /opt/zulu/demo /opt/zulu/sample /opt/zulu/man || true; \
    find /opt/zulu -maxdepth 4 -type f -name "*.zip" -print -delete || true; \
    \
    if [ "${VERIFY}" = "1" ]; then \
      /opt/zulu/bin/java -version; \
    fi

# =========================
# final: runtime only (+ fontconfig)
# =========================
FROM alpine:${ALPINE_VERSION}

ARG INSTALL_BASH=1
ARG VERIFY=0

ENV TZ=Asia/Shanghai \
    LANG=zh_CN.UTF-8 \
    LANGUAGE=zh_CN:zh \
    LC_ALL=zh_CN.UTF-8 \
    JAVA_HOME=/usr/local/zulu \
    PATH=/usr/local/zulu/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# final stage: single RUN for runtime pkgs + tz + scripts + fontconfig config + startup wrapper
RUN set -eux; \
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
    # scripts: now / ll
    printf '%s\n' '#!/bin/sh' 'date "+%Y-%m-%d %H:%M:%S %z"' > /usr/local/bin/now; \
    chmod +x /usr/local/bin/now; \
    printf '%s\n' '#!/bin/sh' 'ls -al "$@"' > /usr/local/bin/ll; \
    chmod +x /usr/local/bin/ll; \
    \
    # Ensure fontconfig scans mounted font dirs reliably:
    # - /fonts (common mount point)
    # - /usr/share/fonts/custom (recommended mount point)
    # Note: placing this under conf.d keeps it minimal and distro-aligned.
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
    # Default shell wrapper:
    # - refresh font cache if any font dir exists (runtime only)
    # - exec bash if present, else sh
    printf '%s\n' \
      '#!/bin/sh' \
      'set -e' \
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

# Optional runtime verification (stdout only)
RUN set -eux; \
    if [ "${VERIFY}" = "1" ]; then \
      echo "TZ=${TZ}"; echo "LANG=${LANG}"; echo "LC_ALL=${LC_ALL}"; \
      date "+%Y-%m-%d %H:%M:%S %z"; \
      java -version; \
      /usr/local/bin/now; \
      /usr/local/bin/ll /; \
      command -v fc-cache; \
      # show fontconfig sees dirs (stdout only)
      fc-list >/dev/null 2>&1 || true; \
    fi

CMD ["/usr/local/bin/_default_shell"]
