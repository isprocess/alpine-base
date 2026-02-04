# 📘 Alpine 3.23 + Azul Zulu JRE 21 LTS（musl）基础镜像使用指南

适用于：

* ✅ Java 21 LTS Web 服务
* ✅ 多架构 amd64 / arm64
* ✅ Alpine/musl 极致瘦身
* ✅ 容器内存自适应堆大小
* ✅ 字体外挂（PDF / 报表 / 图表 / 图片渲染）
* ✅ 安全扫描友好生产环境

---

## 📦 一、镜像特性

* 基于 **Alpine Linux 3.23**（支持 digest 固定，可复现构建）
* 内置 **Azul Zulu JRE 21 LTS（musl 版）**
* 自动适配平台：

  * `linux/amd64`
  * `linux/arm64`
* 两阶段构建（final 镜像无下载工具）
* 默认非 root 用户运行（`app` 用户）
* 内置组件：

| 组件           | 说明        |
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
  -t your-repo/zulu21-web:latest \
  --build-arg ZULU_JRE21_VERSION=21.46.19-ca-jre21.0.9 \
  --build-arg ZULU_SHA256_AMD64="<fill-me>" \
  --build-arg ZULU_SHA256_ARM64="<fill-me>" \
  --build-arg INSTALL_BASH=1 \
  --build-arg VERIFY=0 \
  --push .
```

---

### ✅ 本机构建（自动识别架构）

```bash
docker build -t zulu21-web:local .
```

---

### ✅ 固定 Alpine digest（强可复现 + 扫描稳定）

```bash
docker build \
  --build-arg ALPINE_IMAGE=alpine:3.23@sha256:<digest> \
  -t zulu21-web:repro .
```

---

## ▶️ 三、运行镜像

### 📌 进入交互 Shell

```bash
docker run --rm -it zulu21-web:latest
```

行为说明：

* `INSTALL_BASH=1` → 进入 `bash`
* `INSTALL_BASH=0` → 进入 `sh`

---

### 📌 运行 Java 应用

```bash
docker run --rm \
  -v /path/app.jar:/app/app.jar:ro \
  zulu21-web:latest \
  java -jar /app/app.jar
```

---

## 🔤 四、字体外挂（强烈推荐）

镜像不内置任何字体文件，需通过挂载方式提供。

### ✅ 推荐路径

```bash
docker run --rm -it \
  -v /host/fonts:/usr/share/fonts/custom:ro \
  zulu21-web:latest
```

### ✅ 备用路径

```bash
docker run --rm -it \
  -v /host/fonts:/fonts:ro \
  zulu21-web:latest
```

### ⚙️ 生效机制

容器启动时自动执行：

```bash
fc-cache -f
```

无需手动刷新字体缓存。

---

## ☕ 五、JVM 默认调优（JRE 21）

镜像默认通过 `JAVA_TOOL_OPTIONS` 启用：

```text
-Dfile.encoding=UTF-8
-Duser.timezone=Asia/Shanghai
-XX:+ExitOnOutOfMemoryError
-XX:InitialRAMPercentage=50.0
-XX:MaxRAMPercentage=75.0
```

### 🎯 设计目标

* 适配容器内存限制
* 给 native 内存（线程栈、direct buffer、metaspace）预留空间
* OOM 直接退出，交给编排系统重启

### 示例（1Gi 内存容器）

| 项目  | 结果     |
| --- | ------ |
| 初始堆 | ~512MB |
| 最大堆 | ~768MB |

---

## 📉 六、小内存容器支持（可选）

当容器内存很小（<200MB），JVM 会优先使用 `MinRAMPercentage`。

### 启用方式（推荐运行时覆盖）

```bash
docker run --rm \
  -m 128m \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:MinRAMPercentage=60.0" \
  zulu21-web:latest \
  java -version
```

效果：

* 堆大小 ≈ 128MB × 60% ≈ 77MB

---

## ⚙️ 七、JVM 参数覆盖示例

### 覆盖 Initial / Max（生产常用）

```bash
docker run --rm \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:InitialRAMPercentage=40.0 -XX:MaxRAMPercentage=70.0" \
  zulu21-web:latest \
  java -version
```

### 仅启用 Min（极小内存）

```bash
docker run --rm \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:MinRAMPercentage=65.0" \
  zulu21-web:latest \
  java -version
```

---

## 🧪 八、构建期验证（可选）

```bash
docker build --build-arg VERIFY=1 -t zulu21-web:verify .
```

输出内容：

* Java 版本
* TZ / locale
* now / ll
* fontconfig 状态

👉 仅 stdout，不影响最终镜像层。

---

## 🔐 九、安全与合规设计

✔ final stage 无 curl / wget / git
✔ 默认非 root 用户运行
✔ 精简 JRE（删除 demo/sample/man/zip）
✔ 支持 Alpine digest 固定
✔ 字体外挂避免许可证与扫描误报

兼容常见扫描器：

* Trivy
* Grype
* Anchore

---

## 📁 十、内置命令速查

| 命令     | 功能     |
| ------ | ------ |
| `now`  | 当前时间   |
| `ll`   | 目录列表   |
| `java` | JRE 21 |

---

## 🚀 十一、完整 Web 服务启动示例

### 1Gi 内存 + 字体支持

```bash
docker run -d \
  --name my-java21-app \
  -m 1g \
  -v /data/app.jar:/app/app.jar:ro \
  -v /data/fonts:/usr/share/fonts/custom:ro \
  -e JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:InitialRAMPercentage=50.0 -XX:MaxRAMPercentage=75.0" \
  zulu21-web:latest \
  java -jar /app/app.jar
```

---

## ✅ 十二、自检清单

* [x] Alpine 3.23
* [x] Zulu JRE 21 musl
* [x] amd64 / arm64 支持
* [x] Asia/Shanghai 时区生效
* [x] zh_CN.UTF-8 locale 生效
* [x] fontconfig + 字体外挂
* [x] now / ll 可用
* [x] 非 root 用户运行
* [x] 容器内存自适应堆大小
* [x] OOM 快速退出
* [x] final 镜像无构建工具

---

## 🎯 十三、设计理念总结

该镜像遵循：

✅ 极致瘦身
✅ 多架构统一构建
✅ JVM 行为可预测
✅ 字体渲染无坑
✅ 安全扫描友好

非常适合作为：

👉 Java 21 微服务统一基础镜像
👉 生产 Web 服务基线镜像
