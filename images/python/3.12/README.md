# 📘 Python 3.12.12 + Alpine 3.23 极致瘦身基础镜像使用指南

适用于：

* ✅ Python 3.12 Web 服务 / 数据处理
* ✅ Alpine/musl 极简运行环境
* ✅ 安全扫描低噪音生产场景
* ✅ 字体渲染（PDF/图片/报表）
* ✅ 多架构 amd64 / arm64

---

## 📦 一、镜像特性

* 基于 **Alpine Linux 3.23**
* Python 固定版本：**3.12.12**
* 多阶段裁剪，final 镜像仅保留最小运行集合
* 默认非 root 用户运行（`app`）
* 内置：

| 组件           | 说明            |
| ------------ | ------------- |
| bash         | 可选（默认启用）      |
| tzdata       | Asia/Shanghai |
| musl-locales | zh_CN.UTF-8   |
| fontconfig   | 字体管理          |
| now          | 当前时间          |
| ll           | 目录列表          |

* pip 默认使用 **腾讯镜像源**
* 支持外挂字体目录自动生效
* 构建期验证开关 `VERIFY`

---

## 🏗 二、构建镜像

### ✅ 普通构建

```bash
docker build -t py312:latest .
```

---

### ✅ 构建并验证环境（推荐首次）

```bash
docker build \
  --build-arg VERIFY=1 \
  -t py312:verify .
```

---

### ✅ 固定 Alpine digest（强可复现）

```bash
docker build \
  --build-arg ALPINE_IMAGE=alpine:3.23@sha256:<digest> \
  -t py312:repro .
```

---

## ▶️ 三、基础环境验证（你要求的 --rm 示例）

### 🔍 验证 Python 版本 + pip 源

```bash
docker run --rm py312:latest python -V
```

```bash
docker run --rm py312:latest pip -V
```

```bash
docker run --rm py312:latest pip config list
```

---

### 🔍 验证时区与 locale

```bash
docker run --rm py312:latest date
```

```bash
docker run --rm py312:latest python - <<'EOF'
import locale, time
print(locale.getpreferredencoding())
print(time.tzname)
EOF
```

---

### 🔍 验证内置工具

```bash
docker run --rm py312:latest now
```

```bash
docker run --rm py312:latest ll /
```

---

## 🔤 四、字体外挂与验证

### ✅ 推荐外挂路径

```bash
docker run --rm \
  -v /host/fonts:/usr/share/fonts/custom:ro \
  py312:latest fc-list | head
```

### ✅ 备用路径

```bash
docker run --rm \
  -v /host/fonts:/fonts:ro \
  py312:latest fc-list | head
```

若能看到字体列表，说明 fontconfig 已生效。

---

## ☕ 五、运行 Python 应用示例

### 📌 单文件脚本

```bash
docker run --rm \
  -v $PWD/app.py:/app/app.py:ro \
  py312:latest \
  python /app/app.py
```

---

### 📌 Web 服务（示例）

```bash
docker run -d \
  --name py-app \
  -p 8000:8000 \
  -v $PWD:/app \
  py312:latest \
  python app.py
```

---

## 🔐 六、安全扫描友好设计说明

本镜像已做到：

✔ 固定基础版本
✔ 最小 runtime 依赖
✔ 无构建工具残留
✔ 无字体内置
✔ 删除 Python 测试/缓存文件

推荐扫描参数：

### Trivy

```bash
trivy image \
  --ignore-unfixed \
  --severity HIGH,CRITICAL \
  py312:latest
```

### Grype

```bash
grype py312:latest --only-fixed --fail-on high
```

---

## 📁 七、镜像内目录结构速览

```
/usr/local/bin/
 ├─ python
 ├─ python3
 ├─ pip
 ├─ pip3
 ├─ now
 ├─ ll
 └─ _default_shell

/usr/local/lib/python3.12/
/etc/pip.conf
/etc/fonts/conf.d/99-local-fonts.conf
```

---

## ✅ 八、自检清单

* [x] Alpine 3.23
* [x] Python 3.12.12 固定版本
* [x] zh_CN.UTF-8 locale 生效
* [x] Asia/Shanghai 时区
* [x] Tencent pip 镜像源
* [x] fontconfig + 外挂字体
* [x] now / ll 可用
* [x] 非 root 用户运行
* [x] runtime 依赖极小
* [x] 构建期 VERIFY 可选

---

## 🎯 九、设计理念总结

该镜像遵循：

* ✅ 极致瘦身
* ✅ 构建可复现
* ✅ CVE 噪音极低
* ✅ 运行行为稳定可预测
* ✅ 字体渲染无坑

非常适合：

👉 Python 微服务统一基础镜像
👉 数据处理容器
👉 企业生产环境
