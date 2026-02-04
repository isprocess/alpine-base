# 🐳 Alpine 3.23 + Azul Zulu JRE 8 通用 Web 服务基础镜像使用指南

> 极致瘦身 · 多架构自动适配 · 安全扫描友好 · 字体外挂支持 · Alpine/musl 优化

---

## 📦 镜像特性总览

✅ 基于 Alpine 3.23（支持 digest 固定，可复现构建）

✅ 内置 Azul Zulu JRE 8（musl 版本，Java 1.8.0_482）

✅ 支持多架构自动适配：

* linux/amd64
* linux/arm64

✅ 构建与运行统一命令（无需指定平台参数）

✅ 通用 Web 服务 JVM 内存策略（RAM% 动态适配容器限制）

✅ OOM 自动快速退出（容器编排友好）

✅ fontconfig + 外挂字体支持（中文 PDF/报表/图表无乱码）

✅ 默认非 root 运行（更安全、CVE 噪音更低）

---

## 🏗️ 构建镜像

### 🔹 多架构构建（推荐）

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t your-repo/zulu8-web:latest \
  --build-arg ZULU_JRE8_VERSION=8.92.0.21-ca-jre8.0.482 \
  --build-arg ZULU_SHA256_AMD64=<fill-me> \
  --build-arg ZULU_SHA256_ARM64=<fill-me> \
  --push .
```

---

### 🔹 单机本地构建（自动识别架构）

```bash
docker build -t zulu8-web:local .
```

---

### 🔹 固定基础镜像 digest（可复现 & 扫描稳定）

```bash
docker build \
  --build-arg ALPINE_IMAGE=alpine:3.23@sha256:<digest> \
  -t zulu8-web:repro .
```

---

## ▶️ 运行镜像

### 🔹 基本运行

```bash
docker run --rm -it zulu8-web:latest
```

进入：

* bash（若构建时 INSTALL_BASH=1）
* 否则进入 sh

---

### 🔹 运行 Java 应用

```bash
docker run --rm \
  -v /path/app.jar:/app/app.jar \
  zulu8-web:latest \
  java -jar /app/app.jar
```

---

## 🔤 字体外挂（强烈推荐）

该镜像**不内置字体文件**，请通过挂载方式提供字体。

### ✅ 推荐方式

```bash
docker run --rm -it \
  -v /host/fonts:/usr/share/fonts/custom:ro \
  zulu8-web:latest
```

### ✅ 备用方式

```bash
docker run --rm -it \
  -v /host/fonts:/fonts:ro \
  zulu8-web:latest
```

容器启动时会自动执行：

```bash
fc-cache -f
```

确保字体立即生效。

---

## ☕ JVM 默认调优策略（Web 服务推荐）

镜像内默认设置：

```text
-XX:+UseContainerSupport
-XX:+ExitOnOutOfMemoryError

-XX:MaxRAMPercentage=75.0
-XX:InitialRAMPercentage=25.0
-XX:MinRAMPercentage=10.0
```

### 🎯 含义说明

| 参数                     | 作用         |
| ---------------------- | ---------- |
| UseContainerSupport    | 感知容器内存限制   |
| ExitOnOutOfMemoryError | OOM 立即退出   |
| MaxRAMPercentage       | 最大堆占容器内存比例 |
| InitialRAMPercentage   | 初始堆比例      |
| MinRAMPercentage       | 最小堆比例      |

👉 默认适合 **单 JVM Web 服务容器**

---

## ⚙️ JVM 内存策略自定义

### 🔹 构建时调整

```bash
docker build \
  --build-arg JAVA_MAX_RAM_PERCENT=60.0 \
  --build-arg JAVA_INITIAL_RAM_PERCENT=20.0 \
  --build-arg JAVA_MIN_RAM_PERCENT=10.0 \
  -t zulu8-web:custom .
```

---

### 🔹 运行时覆盖（更灵活，推荐）

```bash
docker run --rm \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:MaxRAMPercentage=60.0" \
  zulu8-web:latest
```

---

### 📌 常见场景推荐

| 场景                   | 推荐配置         |
| -------------------- | ------------ |
| 通用 Web 服务            | 75 / 25 / 10 |
| off-heap 多（Netty/图像） | 60 / 20 / 10 |
| 批处理吞吐优先              | 80 / 30 / 10 |

（格式：Max / Initial / Min）

---

## 🔐 安全特性

* ✅ final stage 无下载工具/构建工具
* ✅ 最小依赖集（严格 allowlist）
* ✅ 默认非 root 用户运行
* ✅ JRE 清理 demo/sample/man/zip 降低扫描面
* ✅ 支持基础镜像 digest 固定

👉 非常适合 Trivy / Grype / Anchore 等扫描环境。

---

## 🧪 构建期验证（可选）

开启：

```bash
docker build --build-arg VERIFY=1 .
```

将输出：

* Java 版本
* TZ / locale
* now / ll 可用性
* fontconfig 状态

（仅 stdout，不污染镜像层）

---

## 📁 内置工具

| 命令  | 功能        |
| --- | --------- |
| now | 当前时间（含时区） |
| ll  | 目录详细列表    |

---

## ✅ 自检清单

* [x] Alpine 3.23
* [x] Java 8 (Zulu 8.92.0.21)
* [x] amd64 / arm64 自动适配
* [x] TZ Asia/Shanghai 生效
* [x] zh_CN.UTF-8 locale 生效
* [x] fontconfig + 字体外挂支持
* [x] 非 root 运行
* [x] JVM 容器内存调优
* [x] OOM 快速退出
* [x] final 镜像无构建工具

---

## 📎 示例：完整 Web 服务启动

```bash
docker run -d \
  --name my-service \
  -m 1g \
  -v /data/fonts:/usr/share/fonts/custom:ro \
  -v /data/app.jar:/app/app.jar \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:MaxRAMPercentage=70.0" \
  zulu8-web:latest \
  java -jar /app/app.jar
```

---

## 🎯 设计理念总结

该镜像遵循：

✔ 极致精简（只保留运行必需）
✔ 多架构可复现
✔ 生产安全基线
✔ JVM 行为可预测
✔ 字体渲染通用无坑

👉 适合作为 **Java 8 微服务 / Web 服务统一基础镜像**
