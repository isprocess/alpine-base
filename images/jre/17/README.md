# 📘 Alpine 3.23 + Azul Zulu JRE 17 LTS（musl）基础镜像使用指南

适用于：

* ✅ Java 17 LTS Web 服务
* ✅ 多架构 amd64 / arm64
* ✅ Alpine/musl 极致瘦身
* ✅ 容器内存自适应堆大小
* ✅ 字体外挂（PDF/报表/图表/图片渲染）
* ✅ 安全扫描友好生产环境

---

## 📦 一、镜像特性

* 基于 **Alpine Linux 3.23**（支持 digest 固定）
* 内置 **Azul Zulu JRE 17 LTS（musl 版）**
* 自动适配平台：

  * `linux/amd64`
  * `linux/arm64`
* 两阶段构建（final 镜像无下载工具）
* 默认非 root 用户运行
* 内置工具：

| 工具           | 说明        |
| ------------ | --------- |
| `now`        | 当前时间（含时区） |
| `ll`         | 目录详细列表    |
| `fontconfig` | 字体管理      |
| `bash`       | 可选安装      |

* 构建期验证开关 `VERIFY`
* JVM 默认针对容器 Web 服务调优

---

## 🏗 二、构建镜像

### ✅ 多架构构建（推荐）

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t your-repo/zulu17-web:latest \
  --build-arg ZULU_JRE17_VERSION=17.64.15-ca-jre17.0.18 \
  --build-arg ZULU_SHA256_AMD64="<fill-me>" \
  --build-arg ZULU_SHA256_ARM64="<fill-me>" \
  --build-arg INSTALL_BASH=1 \
  --build-arg VERIFY=0 \
  --push .
```

---

### ✅ 本机构建（自动识别架构）

```bash
docker build -t zulu17-web:local .
```

---

### ✅ 固定 Alpine digest（强可复现）

```bash
docker build \
  --build-arg ALPINE_IMAGE=alpine:3.23@sha256:<digest> \
  -t zulu17-web:repro .
```

---

## ▶️ 三、运行镜像

### 📌 进入 Shell

```bash
docker run --rm -it zulu17-web:latest
```

行为：

* `INSTALL_BASH=1` → bash
* `INSTALL_BASH=0` → sh

---

### 📌 运行 Java 应用

```bash
docker run --rm \
  -v /path/app.jar:/app/app.jar:ro \
  zulu17-web:latest \
  java -jar /app/app.jar
```

---

## 🔤 四、字体外挂（强烈推荐）

镜像不内置字体文件，需外挂。

### ✅ 推荐路径

```bash
docker run --rm -it \
  -v /host/fonts:/usr/share/fonts/custom:ro \
  zulu17-web:latest
```

### ✅ 备用路径

```bash
docker run --rm -it \
  -v /host/fonts:/fonts:ro \
  zulu17-web:latest
```

### ⚙️ 自动生效

容器启动时自动执行：

```bash
fc-cache -f
```

---

## ☕ 五、JVM 默认调优（JRE 17）

默认 `JAVA_TOOL_OPTIONS`：

```text
-Dfile.encoding=UTF-8
-Duser.timezone=Asia/Shanghai
-XX:+ExitOnOutOfMemoryError
-XX:InitialRAMPercentage=50.0
-XX:MaxRAMPercentage=75.0
```

### 🎯 适用场景

* 单 JVM Web 服务容器
* 内存 ≥ 200MB

效果示例（1Gi 容器）：

| 参数  | 实际值    |
| --- | ------ |
| 初始堆 | ~512MB |
| 最大堆 | ~768MB |

---

## 📉 六、小内存容器支持（可选）

JRE 17 与 8/11 一样支持 `MinRAMPercentage`（小于 ~200MB 时生效）。

### 启用方式（运行时推荐）

```bash
docker run --rm \
  -m 128m \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:MinRAMPercentage=60.0" \
  zulu17-web:latest \
  java -version
```

---

## ⚙️ 七、JVM 参数覆盖示例

### 覆盖 Initial / Max（生产常用）

```bash
docker run --rm \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:InitialRAMPercentage=40.0 -XX:MaxRAMPercentage=70.0" \
  zulu17-web:latest \
  java -version
```

### 仅启用 Min（小内存）

```bash
docker run --rm \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:MinRAMPercentage=65.0" \
  zulu17-web:latest \
  java -version
```

---

## 🧪 八、构建期验证（可选）

```bash
docker build --build-arg VERIFY=1 -t zulu17-web:verify .
```

输出内容：

* Java 版本
* TZ / locale
* now / ll
* fontconfig 状态

👉 仅 stdout，不影响最终镜像。

---

## 🔐 九、安全与合规设计

✔ final stage 无 curl/wget 等工具
✔ 默认非 root 用户运行
✔ 精简 JRE（无 demo/sample/man/zip）
✔ 支持 Alpine digest 固定
✔ 字体外挂避免许可证风险

适配主流扫描器：

* Trivy
* Grype
* Anchore

---

## 📁 十、内置命令速查

| 命令     | 功能     |
| ------ | ------ |
| `now`  | 当前时间   |
| `ll`   | 列目录    |
| `java` | JRE 17 |

---

## 🚀 十一、完整 Web 服务示例

### 1Gi 内存 + 字体支持

```bash
docker run -d \
  --name my-java17-app \
  -m 1g \
  -v /data/app.jar:/app/app.jar:ro \
  -v /data/fonts:/usr/share/fonts/custom:ro \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:InitialRAMPercentage=50.0 -XX:MaxRAMPercentage=75.0" \
  zulu17-web:latest \
  java -jar /app/app.jar
```

---

## ✅ 十二、自检清单

* [x] Alpine 3.23
* [x] Zulu JRE 17 musl
* [x] amd64 / arm64 支持
* [x] Asia/Shanghai 时区生效
* [x] zh_CN.UTF-8 locale 生效
* [x] fontconfig + 字体外挂
* [x] now / ll 可用
* [x] 非 root 用户运行
* [x] 容器内存自适应堆
* [x] OOM 快速退出
* [x] final 镜像无构建工具

---

## 🎯 十三、设计理念总结

该镜像强调：

✅ 极致瘦身
✅ 跨架构统一构建
✅ JVM 行为可预测
✅ 字体通用无坑
✅ 安全扫描友好

非常适合：

👉 Java 17 微服务统一基础镜像
👉 生产 Web 服务基线
