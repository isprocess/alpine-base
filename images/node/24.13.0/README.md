# 📘 Node.js 24.13.0 (LTS) + Alpine 3.23 极致瘦身基础镜像使用指南

> 本镜像基于 Alpine Linux 3.23，使用官方 `node:24.13.0-alpine3.23` 作为 donor，多阶段 COPY 最小运行集合，并集成字体外挂自动生效机制，适用于生产级 Node.js 服务与需要字体渲染的应用。

---

## 📦 一、镜像特性

### ✔ 基础环境

| 项目      | 说明                |
| ------- | ----------------- |
| OS      | Alpine Linux 3.23 |
| Node.js | 24.13.0 (LTS)     |
| Shell   | bash（可选）          |
| 用户      | 非 root（app）       |

---

### ✔ 本地化与时区

```
TZ=Asia/Shanghai
LANG=zh_CN.UTF-8
LANGUAGE=zh_CN:zh
LC_ALL=zh_CN.UTF-8
```

---

### ✔ npm 镜像源（固化配置）

配置文件：

```
/etc/npmrc
registry=http://mirrors.cloud.tencent.com/npm/
```

---

### ✔ 字体支持（不内置字体）

* 已安装 `fontconfig`
* 自动扫描外挂路径：

```
/fonts
/usr/share/fonts/custom
```

* 启动进入 shell 时自动执行：

```
fc-cache -f
```

---

### ✔ 内置辅助命令

| 命令  | 功能          |
| --- | ----------- |
| now | 当前时间（含时区偏移） |
| ll  | ls -al      |

---

## 🏗 二、构建镜像

### 🔹 普通构建

```bash
docker build -t node24-slim:latest .
```

---

### 🔹 构建期验证（推荐首次）

```bash
docker build \
  --build-arg VERIFY=1 \
  -t node24-slim:verify .
```

---

### 🔹 生产瘦身构建（无 bash）

```bash
docker build \
  --build-arg INSTALL_BASH=0 \
  -t node24-slim:prod .
```

---

## ▶️ 三、环境快速验证（--rm）

### ✅ Node / npm / npx / corepack

```bash
docker run --rm node24-slim:latest node -v
docker run --rm node24-slim:latest npm -v
docker run --rm node24-slim:latest npx -v
docker run --rm node24-slim:latest corepack --version
```

---

### ✅ 时区与 locale

```bash
docker run --rm node24-slim:latest date '+%Y-%m-%d %H:%M:%S %z'
```

```bash
docker run --rm node24-slim:latest sh -lc 'echo "TZ=$TZ LANG=$LANG LC_ALL=$LC_ALL"'
```

---

### ✅ npm 镜像源

```bash
docker run --rm node24-slim:latest cat /etc/npmrc
docker run --rm node24-slim:latest npm config get registry
```

---

### ✅ 内置工具

```bash
docker run --rm node24-slim:latest now
docker run --rm node24-slim:latest ll /
```

---

## 🔤 四、字体外挂与验证

### 📌 推荐方式

```bash
docker run --rm \
  -v /host/fonts:/usr/share/fonts/custom:ro \
  node24-slim:latest \
  fc-list | head
```

---

### 📌 备用方式

```bash
docker run --rm \
  -v /host/fonts:/fonts:ro \
  node24-slim:latest \
  fc-list | head
```

---

## 🚀 五、运行 Node 应用示例

### ▶ 单文件

```bash
docker run --rm \
  -v "$PWD":/app \
  -w /app \
  node24-slim:latest \
  node server.js
```

---

### ▶ Web 服务

```bash
docker run -d \
  --name node24-app \
  -p 3000:3000 \
  -v "$PWD":/app \
  node24-slim:latest \
  node app.js
```

---

## 🔐 六、安全扫描低噪音建议

### ✔ 推荐 Trivy 参数

```bash
trivy image \
  --ignore-unfixed \
  --severity HIGH,CRITICAL \
  node24-slim:prod
```

### ✔ 最佳实践

* 生产镜像默认 `INSTALL_BASH=0`
* donor 镜像可进一步锁定 digest
* 避免在 final 镜像中引入构建工具或下载工具

---

## ✅ 七、自检清单

* [x] Alpine 3.23
* [x] Node.js 24.13.0 (LTS) 可执行
* [x] npm / npx / corepack 正常
* [x] Asia/Shanghai 时区生效
* [x] zh_CN.UTF-8 locale 设置完成
* [x] `/etc/npmrc` 固化腾讯源
* [x] fontconfig 已安装
* [x] 外挂字体路径生效（`fc-list` 可验证）
* [x] now / ll 可用
* [x] 非 root 用户运行
* [x] bash 可选

---

## 🧠 八、设计理念总结

本镜像遵循：

* ✅ 最小运行时集合（更小体积 & 更少 CVE）
* ✅ donor COPY 提取运行所需最小文件
* ✅ 配置文件固化避免隐式状态
* ✅ 字体外挂化（不内置）
* ✅ 构建期验证与运行时解耦

非常适合作为：

👉 企业 Node.js LTS 基础镜像
👉 Web/API 服务统一运行环境
👉 需要 PDF/图片渲染的 Node 应用
