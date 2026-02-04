# syntax=docker/dockerfile:1.7
#
# Python 3.12.12 (pinned) on Alpine 3.23 (final) - ultra-slim + scan-friendly
# - Final image base: alpine:3.23 (optionally pinned by digest via ALPINE_IMAGE)
# - Python source: python:3.12.12-alpine3.23 (donor stage, optionally pin by digest in CI)
# - No curl/wget/git/coreutils in final
# - tz/locale fixed, Tencent pip mirror via /etc/pip.conf (file-based, no side effects)
# - fontconfig enabled, supports mounted fonts under /fonts or /usr/share/fonts/custom
# - Optional VERIFY=1 (stdout-only), default off
# - Optional INSTALL_BASH=1 (default on), can set to 0 for lower CVE surface
#

ARG ALPINE_IMAGE=alpine:3.23

# ============================
# donor stage: pinned python
# ============================
FROM python:3.12.12-alpine3.23 AS py

# Slim in donor stage (doesn't bloat final)
RUN set -eux; \
    apk add --no-cache binutils; \
    \
    # Safety trimming (typically safe for production):
    rm -rf /usr/local/lib/python3.12/test || true; \
    find /usr/local/lib/python3.12 -type d -name "test" -prune -exec rm -rf {} + 2>/dev/null || true; \
    find /usr/local/lib/python3.12 -type d -name "__pycache__" -prune -exec rm -rf {} + 2>/dev/null || true; \
    find /usr/local/lib/python3.12 -type f \( -name "*.pyc" -o -name "*.pyo" \) -delete 2>/dev/null || true; \
    \
    # Best-effort strip (ignore failures)
    strip --strip-unneeded /usr/local/bin/python3.12 2>/dev/null || true; \
    strip --strip-unneeded /usr/local/bin/pip3.12 2>/dev/null || true; \
    strip --strip-unneeded /usr/local/lib/libpython3.12.so* 2>/dev/null || true

# ============================
# final runtime (Alpine 3.23)
# ============================
FROM ${ALPINE_IMAGE}

ARG INSTALL_BASH=1
ARG VERIFY=0

ENV TZ=Asia/Shanghai \
    LANG=zh_CN.UTF-8 \
    LANGUAGE=zh_CN:zh \
    LC_ALL=zh_CN.UTF-8 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1 \
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Runtime deps in ONE layer:
# - Keep allowlist minimal for low CVE noise
# - fontconfig included for mounted fonts
RUN set -eux; \
    apk add --no-cache \
      bash \
      tzdata \
      ca-certificates \
      musl-locales \
      musl-locales-lang \
      libstdc++ \
      libgcc \
      fontconfig; \
    if [ "${INSTALL_BASH}" != "1" ]; then apk del --no-network bash 2>/dev/null || true; fi; \
    update-ca-certificates; \
    \
    # tzdata effective
    cp "/usr/share/zoneinfo/${TZ}" /etc/localtime; \
    echo "${TZ}" > /etc/timezone; \
    \
    # Tencent pip mirror (file-based, no side effects)
    printf '%s\n' \
      '[global]' \
      'index-url = https://mirrors.tencent.com/pypi/simple' \
      'trusted-host = mirrors.tencent.com' \
      > /etc/pip.conf; \
    \
    # now / ll helpers (busybox-compatible)
    printf '%s\n' '#!/bin/sh' 'date "+%Y-%m-%d %H:%M:%S %z"' > /usr/local/bin/now; \
    chmod +x /usr/local/bin/now; \
    printf '%s\n' '#!/bin/sh' 'ls -al "$@"' > /usr/local/bin/ll; \
    chmod +x /usr/local/bin/ll; \
    \
    # fontconfig: enable mounted font dirs (do NOT ship fonts)
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

# ---- Minimal Python runtime copy set (pinned to python:3.12.12-alpine3.23) ----
# Python binaries + stdlib + libpython
COPY --from=py /usr/local/bin/python3.12 /usr/local/bin/python3.12
COPY --from=py /usr/local/bin/pip3.12 /usr/local/bin/pip3.12
COPY --from=py /usr/local/lib/python3.12 /usr/local/lib/python3.12
COPY --from=py /usr/local/lib/libpython3.12.so* /usr/local/lib/

# Shared libs required by CPython & common stdlib modules (ssl/sqlite/readline/zlib/etc).
# Copy from donor to avoid expanding apk surface in final.
COPY --from=py /usr/lib/libssl.so.* /usr/lib/
COPY --from=py /usr/lib/libcrypto.so.* /usr/lib/
COPY --from=py /usr/lib/libz.so.* /usr/lib/
COPY --from=py /usr/lib/libbz2.so.* /usr/lib/
COPY --from=py /usr/lib/liblzma.so.* /usr/lib/
COPY --from=py /usr/lib/libsqlite3.so.* /usr/lib/
COPY --from=py /usr/lib/libffi.so.* /usr/lib/
COPY --from=py /usr/lib/libexpat.so.* /usr/lib/
COPY --from=py /usr/lib/libreadline.so.* /usr/lib/
COPY --from=py /usr/lib/libncursesw.so.* /usr/lib/
COPY --from=py /usr/lib/libpanelw.so.* /usr/lib/
COPY --from=py /usr/lib/libmpdec.so.* /usr/lib/

# Optional libs (present in many builds; ignore if missing)
# Use shell-form COPY? Not allowed. So we keep them out to avoid build failures across variants.
# If your runtime needs gdbm (rare), add explicit COPY lines after verifying they exist in your donor image.

# Create standard symlinks + optional verify (stdout only)
RUN set -eux; \
    ln -sf python3.12 /usr/local/bin/python3; \
    ln -sf python3.12 /usr/local/bin/python; \
    ln -sf pip3.12 /usr/local/bin/pip3; \
    ln -sf pip3.12 /usr/local/bin/pip; \
    \
    if [ "${VERIFY}" = "1" ]; then \
      echo "TZ=${TZ}"; echo "LANG=${LANG}"; echo "LC_ALL=${LC_ALL}"; \
      echo "pip.conf:"; sed -n '1,120p' /etc/pip.conf; \
      date "+%Y-%m-%d %H:%M:%S %z"; \
      python --version; \
      python -c "import sys; assert sys.version_info[:3] == (3,12,12), sys.version"; \
      pip --version; \
      python -c "import ssl, sqlite3, zlib, lzma, bz2, ctypes, readline; print('stdlib-ok')"; \
      fc-list >/dev/null 2>&1 || true; \
      /usr/local/bin/now; \
      /usr/local/bin/ll /; \
    fi

USER app
WORKDIR /app

CMD ["/usr/local/bin/_default_shell"]
