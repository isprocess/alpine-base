# 🐳 Alpine 3.23 + Azul Zulu JRE 1.8（musl）通用 Web 服务基础镜像使用指南

> 适用：Java 8（Zulu 8u482）· Alpine/musl · 多架构（amd64/arm64）· 安全扫描友好 · 字体外挂 · 容器内存感知（Initial/Max/MinRAMPercentage）

---

## 1. 镜像特性总览

✅ 基于 Alpine 3.23（支持 digest 固定，可复现构建）

✅ 内置 **Azul Zulu JRE 1.8（musl）**（示例版本：`8.92.0.21-ca-jre8.0.482` / `1.8.0_482`）

✅ 支持多架构自动适配：

* `linux/amd64`
* `linux/arm64`

✅ 运行时依赖最小化（final stage 不含下载工具/构建工具）

✅ 默认非 root 用户运行（更安全，CVE 噪音更低）

✅ `fontconfig` + 外挂字体（适合报表/PDF/图表/中文渲染）

✅ `tzdata + zh_CN.UTF-8 locale` 开箱即用

✅ 内置便捷脚本：`now` / `ll`

✅ 可选 `VERIFY` 构建期验证

✅ JVM 默认启用容器内存感知参数（按 JDK 8u191+ 的真实逻辑设计）

---

## 2. 目录与内置命令

### 2.1 内置脚本

| 路径                   | 命令    | 功能                               |
| -------------------- | ----- | -------------------------------- |
| `/usr/local/bin/now` | `now` | 输出当前时间：`YYYY-MM-DD HH:mm:ss ±TZ` |
| `/usr/local/bin/ll`  | `ll`  | `ls -al "$@"` 目录列表               |

示例：

```bash
now
ll /
```

### 2.2 Java 环境变量

镜像内固定：

* `JAVA_HOME=/usr/local/zulu`
* `PATH` 已包含 `$JAVA_HOME/bin`
* 默认时区/locale：

  * `TZ=Asia/Shanghai`
  * `LANG=zh_CN.UTF-8`
  * `LANGUAGE=zh_CN:zh`
  * `LC_ALL=zh_CN.UTF-8`

验证：

```bash
java -version
date "+%Y-%m-%d %H:%M:%S %z"
```

---

## 3. 构建镜像

> Dockerfile 采用两阶段构建：builder 负责下载解压 JRE，final 只保留运行必需文件，安全扫描友好。

### 3.1 多架构构建（推荐）

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t your-repo/zulu8-web:latest \
  --build-arg ZULU_JRE8_VERSION=8.92.0.21-ca-jre8.0.482 \
  --build-arg ZULU_SHA256_AMD64="<fill-me>" \
  --build-arg ZULU_SHA256_ARM64="<fill-me>" \
  --build-arg INSTALL_BASH=1 \
  --build-arg VERIFY=0 \
  --push .
```

### 3.2 本机构建（自动识别架构）

```bash
docker build -t zulu8-web:local .
```

### 3.3 固定 Alpine digest（可复现 & 扫描稳定）

```bash
docker build \
  --build-arg ALPINE_IMAGE=alpine:3.23@sha256:<digest> \
  -t zulu8-web:repro .
```

> 推荐在 CI 中固定 digest，可降低扫描波动。

---

## 4. 运行镜像

### 4.1 交互进入容器（shell）

默认行为由 `INSTALL_BASH` 控制：

* `INSTALL_BASH=1`：进入 `bash`
* `INSTALL_BASH=0`：进入 `sh`

运行示例：

```bash
docker run --rm -it zulu8-web:latest
```

### 4.2 运行 Java 应用（Jar）

```bash
docker run --rm \
  -v /path/app.jar:/app/app.jar:ro \
  zulu8-web:latest \
  java -jar /app/app.jar
```

---

## 5. 字体外挂（重要）

镜像 **不内置字体文件**，请通过挂载提供字体，适用于：

* 中文渲染
* PDF/报表服务
* 服务器端图片生成（Charts / Graphics2D 等）

### 5.1 推荐挂载路径

```bash
docker run --rm -it \
  -v /host/fonts:/usr/share/fonts/custom:ro \
  zulu8-web:latest
```

### 5.2 备用挂载路径

```bash
docker run --rm -it \
  -v /host/fonts:/fonts:ro \
  zulu8-web:latest
```

### 5.3 字体生效机制

容器启动时会检测字体目录是否存在，若存在自动执行：

```bash
fc-cache -f
```

确保外挂字体立即可用。

---

## 6. JVM 容器内存参数说明（JRE 8u191+ 真实逻辑）

该镜像针对 **JDK 8u191+ / 9+** 的容器内存感知行为，采用“互斥逻辑”进行默认配置：

### 6.1 默认启用的 JVM 参数

镜像默认 `JAVA_TOOL_OPTIONS` 含：

* `-XX:+UseContainerSupport`
* `-XX:+ExitOnOutOfMemoryError`
* `-XX:InitialRAMPercentage=<...>`
* `-XX:MaxRAMPercentage=<...>`
* `-Dfile.encoding=UTF-8`
* `-Duser.timezone=Asia/Shanghai`
* `-Djava.security.egd=file:/dev/urandom`

### 6.2 参数互斥逻辑（关键）

* **小内存容器**（可用内存 `< 200MB`）：

  * JVM 使用 `MinRAMPercentage` 计算堆
  * 会忽略 `InitialRAMPercentage` / `MaxRAMPercentage`

* **正常内存容器**（可用内存 `≥ 200MB`）：

  * JVM 使用 `InitialRAMPercentage` / `MaxRAMPercentage`
  * 会忽略 `MinRAMPercentage`

> 因此：**不需要**满足 `Min ≤ Initial ≤ Max`
> 只建议：`InitialRAMPercentage ≤ MaxRAMPercentage`（镜像构建时会做此推荐校验）

---

## 7. 默认 Web 服务 JVM 配置（推荐）

镜像默认按 Web 服务场景设置：

* `JAVA_INITIAL_RAM_PERCENT=50.0`
* `JAVA_MAX_RAM_PERCENT=75.0`
* `JAVA_MIN_RAM_PERCENT=""`（默认不设置）

含义（容器内存 ≥200MB）：

* 初始堆：容器内存 × 50%
* 最大堆：容器内存 × 75%

---

## 8. JVM 参数自定义

### 8.1 构建时调整（固化到镜像）

```bash
docker build \
  --build-arg JAVA_INITIAL_RAM_PERCENT=40.0 \
  --build-arg JAVA_MAX_RAM_PERCENT=70.0 \
  --build-arg JAVA_MIN_RAM_PERCENT="" \
  -t zulu8-web:custom .
```

### 8.2 运行时覆盖（推荐，更灵活）

#### 覆盖 Initial/Max（适合 ≥200MB 容器）

```bash
docker run --rm \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:InitialRAMPercentage=40.0 -XX:MaxRAMPercentage=70.0" \
  zulu8-web:latest \
  java -version
```

#### 小内存容器启用 Min（适合 <200MB 容器）

```bash
docker run --rm \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:MinRAMPercentage=60.0" \
  zulu8-web:latest \
  java -version
```

---

## 9. 小内存容器（<200MB）使用建议

如果你的容器内存非常小（例如 128MB），推荐**只设置 MinRAMPercentage**（Initial/Max 会被忽略）：

```bash
docker run --rm \
  -m 128m \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:MinRAMPercentage=60.0" \
  zulu8-web:latest \
  java -version
```

---

## 10. OOM 行为（容器编排友好）

镜像默认启用：

* `-XX:+ExitOnOutOfMemoryError`

当发生 OOM 时 JVM 会直接退出进程，K8s/Swarm 等编排系统能及时重启容器，避免“假活”。

---

## 11. 安全与合规要点

* final stage **不包含** `curl/wget/git` 等下载工具
* final stage 仅 allowlist 运行依赖包
* 默认非 root 用户 `app`
* JRE 移除 demo/sample/man/zip 以减少扫描面
* 支持 Alpine digest 固定减少 CVE 扫描漂移
* 不内置字体文件（减少许可风险与误报）

---

## 12. 构建期验证（可选）

开启构建验证：

```bash
docker build --build-arg VERIFY=1 -t zulu8-web:verify .
```

将输出（stdout-only）：

* Java 版本
* TZ/locale 与 date
* now/ll 可用性
* fontconfig 状态

---

## 13. 常用运行示例（完整）

### 13.1 标准 Web 服务启动（1Gi 容器）

```bash
docker run -d \
  --name my-service \
  -m 1g \
  -v /data/app.jar:/app/app.jar:ro \
  -v /data/fonts:/usr/share/fonts/custom:ro \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:InitialRAMPercentage=50.0 -XX:MaxRAMPercentage=75.0" \
  zulu8-web:latest \
  java -jar /app/app.jar
```

### 13.2 小内存服务（128MB）

```bash
docker run --rm \
  -m 128m \
  -v /data/app.jar:/app/app.jar:ro \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:MinRAMPercentage=60.0" \
  zulu8-web:latest \
  java -jar /app/app.jar
```

---

## 14. 自检清单

* [x] 基于 Alpine 3.23（支持 digest 固定）
* [x] Java 8（Zulu musl）正常：`java -version`
* [x] `TZ=Asia/Shanghai` 生效
* [x] `zh_CN.UTF-8` locale 可用（musl-locales）
* [x] now/ll 存在且可执行
* [x] INSTALL_BASH 可控制 shell（bash/sh）
* [x] VERIFY 默认关闭，开启后输出校验信息
* [x] amd64/arm64 支持（TARGET* + uname 回退）
* [x] final stage 不含下载工具与临时文件
* [x] fontconfig + 外挂字体路径生效
* [x] 默认 Initial/Max（Web 服务），Min 可选（小内存容器）
* [x] 仅校验 Initial ≤ Max（不校验 Min 关系）
