# syntax=docker/dockerfile:1

# ------------------------------------------------------------
# Sources (use official images as runtime "donors")
# ------------------------------------------------------------
FROM node:22.22.0-alpine3.23 AS node-src
# 固化 npm 腾讯源（避免 npm config 命令产生副作用/噪音）
RUN printf "registry=http://mirrors.cloud.tencent.com/npm/\n" > /etc/npmrc

FROM python:3.12.12-alpine3.23 AS py-src
# 固化 pip 腾讯源
RUN mkdir -p /etc && \
    printf "[global]\nindex-url = https://mirrors.tencent.com/pypi/simple\ntrusted-host = mirrors.tencent.com\n" > /etc/pip.conf


# ------------------------------------------------------------
# Final runtime (ultra slim)
# ------------------------------------------------------------
FROM alpine:3.23

LABEL maintainer="devops" \
      description="Node.js 22.22 + Python 3.12 on Alpine 3.23 (Shanghai TZ, zh_CN, slim)"

# 构建期可选验证：默认关闭，避免影响缓存/构建速度
ARG VERIFY=0

ENV TZ=Asia/Shanghai \
    LANG=zh_CN.UTF-8 \
    LANGUAGE=zh_CN:zh \
    LC_ALL=zh_CN.UTF-8 \
    PATH="/usr/local/bin:${PATH}" \
    NPM_CONFIG_REGISTRY=http://mirrors.cloud.tencent.com/npm/ \
    PIP_INDEX_URL=https://mirrors.tencent.com/pypi/simple \
    PIP_TRUSTED_HOST=mirrors.tencent.com

# ---- minimal deps + timezone/locale + helper scripts (single layer) ----
RUN set -eux; \
    apk add --no-cache \
        bash \
        tzdata \
        ca-certificates \
        musl-locales \
        musl-locales-lang \
        libstdc++ \
        libgcc; \
    \
    # timezone
    cp /usr/share/zoneinfo/${TZ} /etc/localtime; \
    echo ${TZ} > /etc/timezone; \
    \
    # helper scripts (busybox based)
    printf '#!/bin/sh\nexec date "+%%Y-%%m-%%d %%H:%%M:%%S %%z"\n' > /usr/local/bin/now; \
    chmod +x /usr/local/bin/now; \
    printf '#!/bin/sh\nexec ls -al "$@"\n' > /usr/local/bin/ll; \
    chmod +x /usr/local/bin/ll

# ---- Node runtime: copy essentials only (avoid copying /usr/local/bin/npm shim) ----
COPY --from=node-src /etc/npmrc /etc/npmrc
COPY --from=node-src /usr/local/bin/node /usr/local/bin/node
COPY --from=node-src /usr/local/lib/node_modules /usr/local/lib/node_modules

# ---- Python runtime: copy essentials only ----
COPY --from=py-src /etc/pip.conf /etc/pip.conf
COPY --from=py-src /usr/local/bin/python3.12 /usr/local/bin/python3.12
COPY --from=py-src /usr/local/bin/pip3.12 /usr/local/bin/pip3.12
COPY --from=py-src /usr/local/lib/python3.12 /usr/local/lib/python3.12
COPY --from=py-src /usr/local/lib/libpython3.12.so* /usr/local/lib/

# ---- Symlinks + safe pruning + (optional) strip ----
RUN set -eux; \
    # Standard python links
    ln -sf python3.12 /usr/local/bin/python; \
    ln -sf python3.12 /usr/local/bin/python3; \
    ln -sf pip3.12 /usr/local/bin/pip; \
    ln -sf pip3.12 /usr/local/bin/pip3; \
    \
    # IMPORTANT: recreate npm/npx/corepack links correctly
    ln -sf ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm; \
    ln -sf ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx; \
    ln -sf ../lib/node_modules/corepack/dist/corepack.js /usr/local/bin/corepack; \
    \
    # ----- Safe pruning: remove caches/tests/docs (typical prod-safe) -----
    # Python: drop stdlib tests + bytecode caches
    rm -rf /usr/local/lib/python3.12/test /usr/local/lib/python3.12/*/test || true; \
    find /usr/local/lib/python3.12 -type d -name '__pycache__' -exec rm -rf '{}' +; \
    find /usr/local/lib/python3.12 -type f \( -name '*.pyc' -o -name '*.pyo' \) -delete; \
    \
    # npm: drop docs/man
    rm -rf /usr/local/lib/node_modules/npm/docs /usr/local/lib/node_modules/npm/man || true; \
    \
    # strip ELF where possible (ignore failures)
    strip --strip-unneeded /usr/local/bin/node 2>/dev/null || true; \
    strip --strip-unneeded /usr/local/lib/libpython3.12.so* 2>/dev/null || true

# ---- Optional build-time verification (disabled by default) ----
RUN set -eux; \
    if [ "${VERIFY}" = "1" ]; then \
      echo "=== Environment Check ==="; \
      node -v; \
      npm -v; \
      npx -v; \
      corepack --version || true; \
      python3 -V; \
      pip3 -V; \
      echo "=== Locale/Timezone Check ==="; \
      echo "LANG=$LANG LC_ALL=$LC_ALL TZ=$TZ"; \
      date '+%Y-%m-%d %H:%M:%S %z'; \
      echo "=== Mirror Check ==="; \
      npm config get registry; \
      python3 -c "import os; print('PIP_INDEX_URL=', os.getenv('PIP_INDEX_URL'))"; \
      echo "=== Custom Commands Check ==="; \
      now; \
      ll /usr/local/bin | head -5; \
    fi

WORKDIR /app
CMD ["bash"]
