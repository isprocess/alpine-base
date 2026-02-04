# syntax=docker/dockerfile:1.7
#
# Node.js 24.13.0 (LTS) on Alpine 3.23 - ultra-slim + scan-friendly + fontconfig
# - Final image base: alpine:3.23
# - Node source: node:24.13.0-alpine3.23 (donor stage)
# - No curl/wget/git/coreutils in final
# - tz/locale fixed, Tencent npm mirror via /etc/npmrc (file-based, no side effects)
# - fontconfig enabled, supports mounted fonts under /fonts or /usr/share/fonts/custom
# - Optional VERIFY=1 (stdout-only), default off
# - Optional INSTALL_BASH=1 (default on), can set to 0 for lower CVE surface
#

# ------------------------------------------------------------
# Node donor
# ------------------------------------------------------------
FROM node:24.13.0-alpine3.23 AS node-src

# 固化 npm 腾讯源（避免 npm config 命令副作用/噪音）
RUN set -eux; \
    printf 'registry=http://mirrors.cloud.tencent.com/npm/\n' > /etc/npmrc; \
    # npm 安全裁剪（不会影响生产功能）
    rm -rf /usr/local/lib/node_modules/npm/docs /usr/local/lib/node_modules/npm/man || true; \
    # best-effort strip（忽略失败；只在 donor 中安装工具）
    apk add --no-cache binutils >/dev/null 2>&1 || true; \
    strip --strip-unneeded /usr/local/bin/node 2>/dev/null || true

# ------------------------------------------------------------
# Final runtime (ultra slim)
# ------------------------------------------------------------
FROM alpine:3.23

LABEL maintainer="devops" \
      description="Node.js 24.13.0 (LTS) on Alpine 3.23 (Shanghai TZ, zh_CN, slim, fontconfig)"

ARG VERIFY=0
ARG INSTALL_BASH=1

ENV TZ=Asia/Shanghai \
    LANG=zh_CN.UTF-8 \
    LANGUAGE=zh_CN:zh \
    LC_ALL=zh_CN.UTF-8 \
    PATH=/usr/local/bin:/usr/local/sbin:/usr/sbin:/usr/bin:/sbin:/bin \
    # 兼容性保留（但以 /etc/npmrc 为准）
    NPM_CONFIG_REGISTRY=http://mirrors.cloud.tencent.com/npm/

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

# ---- Recreate npm/npx/corepack links correctly + optional verify ----
RUN set -eux; \
    # IMPORTANT: recreate npm/npx/corepack links correctly (avoid shim/symlink copy pitfalls)
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
      echo "=== Locale/Timezone ==="; \
      echo "LANG=$LANG LC_ALL=$LC_ALL TZ=$TZ"; \
      date '+%Y-%m-%d %H:%M:%S %z'; \
      echo "=== Mirror (file-based) ==="; \
      echo "/etc/npmrc:"; sed -n '1,80p' /etc/npmrc; \
      echo "npm registry:"; npm config get registry; \
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
