# syntax=docker/dockerfile:1.7
# -----------------------------------------------------------------------------
# Alpine 3.23 + Azul Zulu JRE 8 (musl) - multi-arch (amd64/arm64), reproducible, slim
#
# Avoid pitfalls:
# - Download tools only in builder stage: final image doesn't ship curl/wget/tar -> smaller + less attack surface.
# - Prefer TARGETPLATFORM/TARGETARCH for deterministic multi-arch selection under buildx.
# - Fallback to uname -m only when TARGET* is absent (plain docker build).
# - Keep SHA256 ARGs for reproducibility + supply-chain integrity; empty => downgrade to HTTPS + fail-fast.
# -----------------------------------------------------------------------------

ARG ALPINE_VERSION=3.23

# =========================
# builder stage: fetch + verify + unpack (ONLY here we have curl/tar)
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
    # 1) Prefer BuildKit-provided arch
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
    # Integrity check: prefer sha256; if empty, downgrade (still fail-fast via curl -f)
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
    # Conservative slimming: remove non-runtime extras (keep core libs/security)
    rm -rf /opt/zulu/demo /opt/zulu/sample /opt/zulu/man || true; \
    find /opt/zulu -maxdepth 4 -type f -name "*.zip" -print -delete || true; \
    \
    # Optional build-time verification (stdout only)
    if [ "${VERIFY}" = "1" ]; then \
      /opt/zulu/bin/java -version; \
    fi

# =========================
# final stage: runtime only (strict allowlist)
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

# final stage: single RUN for apk add + tz + scripts + default shell wrapper
RUN set -eux; \
    apk add --no-cache tzdata ca-certificates musl-locales musl-locales-lang libstdc++ libgcc; \
    if [ "${INSTALL_BASH}" = "1" ]; then apk add --no-cache bash; fi; \
    update-ca-certificates; \
    \
    # tzdata effective
    cp "/usr/share/zoneinfo/${TZ}" /etc/localtime; \
    echo "${TZ}" > /etc/timezone; \
    \
    # /usr/local/bin/now
    printf '%s\n' '#!/bin/sh' 'date "+%Y-%m-%d %H:%M:%S %z"' > /usr/local/bin/now; \
    chmod +x /usr/local/bin/now; \
    \
    # /usr/local/bin/ll
    printf '%s\n' '#!/bin/sh' 'ls -al "$@"' > /usr/local/bin/ll; \
    chmod +x /usr/local/bin/ll; \
    \
    # Default shell behavior: if bash installed -> bash, else -> sh
    # (Dockerfile can't conditionally switch CMD JSON array based on ARG reliably, so wrapper is smallest equivalent)
    printf '%s\n' \
      '#!/bin/sh' \
      'if command -v bash >/dev/null 2>&1; then exec bash; fi' \
      'exec sh' \
      > /usr/local/bin/_default_shell; \
    chmod +x /usr/local/bin/_default_shell

# Only allowed JRE copy from builder
COPY --from=builder /opt/zulu /usr/local/zulu

# Optional runtime verification (stdout only; no filesystem writes beyond stdout)
RUN set -eux; \
    if [ "${VERIFY}" = "1" ]; then \
      echo "TZ=${TZ}"; echo "LANG=${LANG}"; echo "LC_ALL=${LC_ALL}"; \
      date "+%Y-%m-%d %H:%M:%S %z"; \
      java -version; \
      /usr/local/bin/now; \
      /usr/local/bin/ll /; \
    fi

CMD ["/usr/local/bin/_default_shell"]
