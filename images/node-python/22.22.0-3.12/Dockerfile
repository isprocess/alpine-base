# 多阶段构建：Node.js 22.22 + Python 3.12 (Alpine 3.23)

# ============================================
# Stage 1: Node.js 基础
# ============================================
FROM node:22.22.0-alpine3.23 AS node-builder

ENV NODE_HOME=/usr/local
ENV NPM_CONFIG_REGISTRY=http://mirrors.cloud.tencent.com/npm/

# 配置 npm 腾讯云镜像
RUN npm config set registry http://mirrors.cloud.tencent.com/npm/ && \
    npm config get registry

# 准备 Node.js 打包
RUN mkdir -p /opt/node-export/usr/local/bin && \
    mkdir -p /opt/node-export/usr/local/lib && \
    mkdir -p /opt/node-export/usr/local/include && \
    cp -r /usr/local/bin/node /opt/node-export/usr/local/bin/ && \
    cp -r /usr/local/lib/node_modules /opt/node-export/usr/local/lib/ && \
    ln -sf ../lib/node_modules/npm/bin/npm-cli.js /opt/node-export/usr/local/bin/npm && \
    ln -sf ../lib/node_modules/npm/bin/npx-cli.js /opt/node-export/usr/local/bin/npx

# ============================================
# Stage 2: Python 基础
# ============================================
FROM python:3.12.12-alpine3.23 AS python-builder

ENV PYTHON_HOME=/usr/local
ENV INSTALLDIR=/usr/local
ENV PIP_INDEX_URL=https://mirrors.tencent.com/pypi/simple
ENV PIP_TRUSTED_HOST=mirrors.tencent.com

# 准备 Python 打包
RUN mkdir -p /opt/python-export/usr/local/bin && \
    mkdir -p /opt/python-export/usr/local/lib && \
    mkdir -p /opt/python-export/usr/local/include && \
    cp /usr/local/bin/python3.12 /opt/python-export/usr/local/bin/ && \
    cp /usr/local/bin/pip3.12 /opt/python-export/usr/local/bin/ 2>/dev/null || cp /usr/local/bin/pip3 /opt/python-export/usr/local/bin/pip3.12 && \
    cp -r /usr/local/lib/python3.12 /opt/python-export/usr/local/lib/ && \
    cp /usr/local/lib/libpython3.12.so* /opt/python-export/usr/local/lib/ 2>/dev/null || true && \
    cp -r /usr/local/include/python3.12 /opt/python-export/usr/local/include/ 2>/dev/null || true && \
    ln -sf python3.12 /opt/python-export/usr/local/bin/python && \
    ln -sf python3.12 /opt/python-export/usr/local/bin/python3 && \
    ln -sf pip3.12 /opt/python-export/usr/local/bin/pip && \
    ln -sf pip3.12 /opt/python-export/usr/local/bin/pip3

# ============================================
# Stage 3: 最终最小化镜像
# ============================================
FROM alpine:3.23

LABEL maintainer="devops" \
      description="Node.js 22.22 + Python 3.12 on Alpine 3.23 (Shanghai TZ, zh_CN)"

# 时区和语言环境（必须保留 tzdata 才能正确识别夏令时等规则）
ENV TZ=Asia/Shanghai \
    LANG=zh_CN.UTF-8 \
    LANGUAGE=zh_CN:zh \
    LC_ALL=zh_CN.UTF-8 \
    NODE_HOME=/usr/local \
    PYTHON_HOME=/usr/local \
    PATH="/usr/local/bin:${PATH}" \
    PIP_INDEX_URL=https://mirrors.tencent.com/pypi/simple \
    PIP_TRUSTED_HOST=mirrors.tencent.com \
    NPM_CONFIG_REGISTRY=http://mirrors.cloud.tencent.com/npm/

# 安装运行时依赖（保留 tzdata 确保时区数据库完整）
RUN apk add --no-cache \
        tzdata \
        musl-locales \
        musl-locales-lang \
        ca-certificates \
        libstdc++ \
        libgcc \
        bash \
        curl \
        coreutils \
    && cp /usr/share/zoneinfo/${TZ} /etc/localtime \
    && echo ${TZ} > /etc/timezone

# 复制 Node.js 和 Python
COPY --from=node-builder /opt/node-export/usr/local /usr/local
COPY --from=python-builder /opt/python-export/usr/local /usr/local

# 添加实用工具脚本（now 和 ll）
RUN echo -e '#!/bin/sh\ndate "+%Y-%m-%d %H:%M:%S %z"' > /usr/local/bin/now && \
    chmod +x /usr/local/bin/now && \
    echo -e '#!/bin/sh\nls --color=auto -alF' > /usr/local/bin/ll && \
    chmod +x /usr/local/bin/ll && \
    # 验证脚本
    echo "=== Testing custom commands ===" && \
    echo "Testing 'now':" && now && \
    echo "Testing 'll':" && ll /usr/local/bin/ | head -5

# 验证安装
RUN echo "=== Environment Check ===" && \
    echo "Node.js: $(node --version)" && \
    echo "NPM: $(npm --version)" && \
    echo "Python: $(python3 --version)" && \
    echo "Pip: $(pip3 --version)" && \
    echo "" && \
    echo "=== Locale Check ===" && \
    echo "LANG: $LANG" && \
    echo "LC_ALL: $LC_ALL" && \
    echo "" && \
    echo "=== Timezone Check ===" && \
    echo "TZ: $TZ" && \
    echo "Date: $(date '+%Y-%m-%d %H:%M:%S %z')" && \
    echo "" && \
    echo "=== Mirror Config ===" && \
    echo "NPM: $(npm config get registry)" && \
    echo "PIP: $PIP_INDEX_URL"

WORKDIR /app

CMD ["bash"]
