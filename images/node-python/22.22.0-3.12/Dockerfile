# syntax=docker/dockerfile:1.7

# ------------------------------------------------------------
# Sources (runtime donors)
# ------------------------------------------------------------
FROM node:22.22.0-alpine3.23 AS node-src
# 固化 npm 腾讯源（避免 npm config 命令副作用/噪音）
RUN set -eux; \
    printf 'registry=http://mirrors.cloud.tencent.com/npm/\n' > /etc/npmrc; \
    # npm 安全裁剪（不会影响生产功能）
    rm -rf /usr/local/lib/node_modules/npm/docs /usr/local/lib/node_modules/npm/man || true; \
    # best-effort strip（忽略失败）
    apk add --no-cache binutils >/dev/null 2>&1 || true; \
    strip --strip-unneeded /usr/local/bin/node 2>/dev/null || true

FROM python:3.12.12-alpine3.23 AS py-src
# 固化 pip 腾讯源
RUN set -eux; \
    mkdir -p /etc; \
    printf '%s\n' \
      '[global]' \
      'index-url = https://mirrors.tencent.com/pypi/simple' \
      'trusted-host = mirrors.tencent.com' \
      > /etc/pip.conf; \
    \
    # Python 安全裁剪：tests/__pycache__/pyc
    rm -rf /usr/local/lib/python3.12/test || true; \
    find /usr/local/lib/python3.12 -type d -name "test" -prune -exec rm -rf {} + 2>/dev/null || true; \
    find /usr/local/lib/python3.12 -type d -name "__pycache__" -prune -exec rm -rf {} + 2>/dev/null || true; \
    find /usr/local/lib/python3.12 -type f \( -name "*.pyc" -o -name "*.pyo" \) -delete 2>/dev/null || true; \
    \
    # best-effort strip（忽略失败）
    apk add --no-cache binutils >/dev/null 2>&1 || true; \
    strip --strip-unneeded /usr/local/bin/python3.12 2>/dev/null || true; \
    strip --strip-unneeded /usr/local/bin/pip3.12 2>/dev/null || true; \
    strip --strip-unneeded /usr/local/lib/libpython3.12.so* 2>/dev/null || true

# ------------------------------------------------------------
# Final runtime (ultra slim)
# ------------------------------------------------------------
FROM alpine:3.23

LABEL maintainer="devops" \
      description="Node.js 22.22.0 + Python 3.12.12 on Alpine 3.23 (Shanghai TZ, zh_CN, slim, fontconfig)"

ARG VERIFY=0
ARG INSTALL_BASH=1

ENV TZ=Asia/Shanghai \
    LANG=zh_CN.UTF-8 \
    LANGUAGE=zh_CN:zh \
    LC_ALL=zh_CN.UTF-8 \
    PATH=/usr/local/bin:/usr/local/sbin:/usr/sbin:/usr/bin:/sbin:/bin \
    # 兼容性保留（但以 /etc/npmrc 与 /etc/pip.conf 为准）
    NPM_CONFIG_REGISTRY=http://mirrors.cloud.tencent.com/npm/ \
    PIP_INDEX_URL=https://mirrors.tencent.com/pypi/simple \
    PIP_TRUSTED_HOST=mirrors.tencent.com \
    # Python 运行习惯（减少容器内无意义写入/缓冲）
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1

# ---- minimal deps + tz/locale + helpers + fontconfig + non-root + wrapper (尽量单层) ----
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
    # timezone
    cp "/usr/share/zoneinfo/${TZ}" /etc/localtime; \
    echo "${TZ}" > /etc/timezone; \
    \
    # helper scripts (busybox based)
    printf '%s\n' '#!/bin/sh' 'exec date "+%Y-%m-%d %H:%M:%S %z"' > /usr/local/bin/now; \
    chmod +x /usr/local/bin/now; \
    printf '%s\n' '#!/bin/sh' 'exec ls -al "$@"' > /usr/local/bin/ll; \
    chmod +x /usr/local/bin/ll; \
    \
    # fontconfig 外挂字体路径声明（不内置字体文件）
    mkdir -p /etc/fonts/conf.d /var/cache/fontconfig; \
    printf '%s\n' \
      '<?xml version="1.0"?>' \
      '<!DOCTYPE fontconfig SYSTEM "fonts.dtd">' \
      '<fontconfig>' \
      '  <dir>/fonts</dir>' \
      '  <dir>/usr/share/fonts/custom</dir>' \
      '</fontconfig>' \
      > /etc/fonts/conf.d/99-local-fonts.conf; \
    \
    # Non-root runtime（更安全/更少扫描噪音）
    addgroup -S app; \
    adduser  -S -G app -h /app -s /sbin/nologin app; \
    mkdir -p /app /tmp; \
    chown -R app:app /app /tmp /var/cache/fontconfig; \
    chmod 1777 /tmp; \
    \
    # Default shell wrapper: mounted fonts -> refresh cache, then bash/sh
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

# ---- Node runtime: copy essentials only (avoid copying /usr/local/bin/npm shim) ----
COPY --from=node-src /etc/npmrc /etc/npmrc
COPY --from=node-src /usr/local/bin/node /usr/local/bin/node
COPY --from=node-src /usr/local/lib/node_modules /usr/local/lib/node_modules

# ---- Python runtime: copy essentials only + required shared libs (avoid apk bloat) ----
COPY --from=py-src /etc/pip.conf /etc/pip.conf
COPY --from=py-src /usr/local/bin/python3.12 /usr/local/bin/python3.12
COPY --from=py-src /usr/local/bin/pip3.12 /usr/local/bin/pip3.12
COPY --from=py-src /usr/local/lib/python3.12 /usr/local/lib/python3.12
COPY --from=py-src /usr/local/lib/libpython3.12.so* /usr/local/lib/

# Shared libs commonly required by CPython stdlib modules (ssl/sqlite/readline/zlib/etc)
COPY --from=py-src /usr/lib/libssl.so.* /usr/lib/
COPY --from=py-src /usr/lib/libcrypto.so.* /usr/lib/
COPY --from=py-src /usr/lib/libz.so.* /usr/lib/
COPY --from=py-src /usr/lib/libbz2.so.* /usr/lib/
COPY --from=py-src /usr/lib/liblzma.so.* /usr/lib/
COPY --from=py-src /usr/lib/libsqlite3.so.* /usr/lib/
COPY --from=py-src /usr/lib/libffi.so.* /usr/lib/
COPY --from=py-src /usr/lib/libexpat.so.* /usr/lib/
COPY --from=py-src /usr/lib/libreadline.so.* /usr/lib/
COPY --from=py-src /usr/lib/libncursesw.so.* /usr/lib/
COPY --from=py-src /usr/lib/libpanelw.so.* /usr/lib/
COPY --from=py-src /usr/lib/libmpdec.so.* /usr/lib/

# ---- Symlinks (python/pip + npm/npx/corepack) + optional verify ----
RUN set -eux; \
    # Standard python links
    ln -sf python3.12 /usr/local/bin/python; \
    ln -sf python3.12 /usr/local/bin/python3; \
    ln -sf pip3.12 /usr/local/bin/pip; \
    ln -sf pip3.12 /usr/local/bin/pip3; \
    \
    # Recreate npm/npx/corepack links correctly (avoid shim/symlink copy pitfalls)
    ln -sf ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm; \
    ln -sf ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx; \
    ln -sf ../lib/node_modules/corepack/dist/corepack.js /usr/local/bin/corepack; \
    \
    if [ "${VERIFY}" = "1" ]; then \
      echo "=== Versions ==="; \
      node -v; \
      npm -v; \
      npx -v; \
      corepack --version || true; \
      python3 -V; \
      pip3 -V; \
      echo "=== Locale/Timezone ==="; \
      echo "LANG=$LANG LC_ALL=$LC_ALL TZ=$TZ"; \
      date '+%Y-%m-%d %H:%M:%S %z'; \
      echo "=== Mirror (file-based) ==="; \
      echo "/etc/npmrc:"; sed -n '1,80p' /etc/npmrc; \
      echo "/etc/pip.conf:"; sed -n '1,120p' /etc/pip.conf; \
      echo "npm registry:"; npm config get registry; \
      echo "=== Python stdlib sanity ==="; \
      python3 -c "import ssl,sqlite3,zlib,lzma,bz2,ctypes,readline; print('stdlib-ok')"; \
      echo "=== Fontconfig ==="; \
      fc-cache -f >/dev/null 2>&1 || true; \
      fc-list >/dev/null 2>&1 || true; \
      echo "=== Custom Commands ==="; \
      now; \
      ll /usr/local/bin | head -20; \
    fi

USER app
WORKDIR /app

# 默认进入 shell：若 INSTALL_BASH=1 -> bash，否则 sh
CMD ["/usr/local/bin/_default_shell"]
