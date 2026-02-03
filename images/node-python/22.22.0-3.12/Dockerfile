# syntax=docker/dockerfile:1

# ------------------------------------------------------------
# Sources
# ------------------------------------------------------------
FROM node:22.22.0-alpine3.23 AS node-src
RUN printf "registry=http://mirrors.cloud.tencent.com/npm/\n" > /etc/npmrc

FROM python:3.12.12-alpine3.23 AS py-src
RUN mkdir -p /etc && \
    printf "[global]\nindex-url = https://mirrors.tencent.com/pypi/simple\ntrusted-host = mirrors.tencent.com\n" > /etc/pip.conf


# ------------------------------------------------------------
# Final runtime
# ------------------------------------------------------------
FROM alpine:3.23

LABEL maintainer="devops" \
      description="Node.js 22.22 + Python 3.12 on Alpine 3.23 (ultra slim)"

ARG VERIFY=0

ENV TZ=Asia/Shanghai \
    LANG=zh_CN.UTF-8 \
    LANGUAGE=zh_CN:zh \
    LC_ALL=zh_CN.UTF-8 \
    PATH="/usr/local/bin:${PATH}" \
    NPM_CONFIG_REGISTRY=http://mirrors.cloud.tencent.com/npm/ \
    PIP_INDEX_URL=https://mirrors.tencent.com/pypi/simple \
    PIP_TRUSTED_HOST=mirrors.tencent.com

# ---- minimal deps only ----
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
    cp /usr/share/zoneinfo/${TZ} /etc/localtime; \
    echo ${TZ} > /etc/timezone; \
    \
    # now / ll (busybox based)
    printf '#!/bin/sh\nexec date "+%%Y-%%m-%%d %%H:%%M:%%S %%z"\n' > /usr/local/bin/now; \
    chmod +x /usr/local/bin/now; \
    printf '#!/bin/sh\nexec ls -al "$@"\n' > /usr/local/bin/ll; \
    chmod +x /usr/local/bin/ll

# ---- Node ----
COPY --from=node-src /etc/npmrc /etc/npmrc
COPY --from=node-src /usr/local/bin/node /usr/local/bin/
COPY --from=node-src /usr/local/bin/npm /usr/local/bin/
COPY --from=node-src /usr/local/bin/npx /usr/local/bin/
COPY --from=node-src /usr/local/bin/corepack /usr/local/bin/
COPY --from=node-src /usr/local/lib/node_modules /usr/local/lib/node_modules

# ---- Python ----
COPY --from=py-src /etc/pip.conf /etc/pip.conf
COPY --from=py-src /usr/local/bin/python3.12 /usr/local/bin/
COPY --from=py-src /usr/local/bin/pip3.12 /usr/local/bin/
COPY --from=py-src /usr/local/lib/python3.12 /usr/local/lib/python3.12
COPY --from=py-src /usr/local/lib/libpython3.12.so* /usr/local/lib/

# ---- symlink + aggressive safe prune ----
RUN set -eux; \
    ln -sf python3.12 /usr/local/bin/python; \
    ln -sf python3.12 /usr/local/bin/python3; \
    ln -sf pip3.12 /usr/local/bin/pip; \
    ln -sf pip3.12 /usr/local/bin/pip3; \
    \
    # Python cleanup
    rm -rf /usr/local/lib/python3.12/test /usr/local/lib/python3.12/*/test || true; \
    find /usr/local/lib/python3.12 -type d -name '__pycache__' -exec rm -rf '{}' +; \
    find /usr/local/lib/python3.12 -type f \( -name '*.pyc' -o -name '*.pyo' \) -delete; \
    \
    # npm cleanup
    rm -rf /usr/local/lib/node_modules/npm/docs /usr/local/lib/node_modules/npm/man || true; \
    \
    # strip binaries
    strip --strip-unneeded /usr/local/bin/node 2>/dev/null || true; \
    strip --strip-unneeded /usr/local/lib/libpython3.12.so* 2>/dev/null || true

# ---- optional verify ----
RUN set -eux; \
    if [ "$VERIFY" = "1" ]; then \
      node -v; \
      npm -v; \
      python3 -V; \
      pip3 -V; \
      now; \
      ll /usr/local/bin | head -5; \
      echo "TZ=$TZ LANG=$LANG"; \
      date '+%Y-%m-%d %H:%M:%S %z'; \
    fi

WORKDIR /app
CMD ["bash"]
