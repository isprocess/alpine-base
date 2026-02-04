# 📘 Node 22.22.0 + Python 3.12.12（Alpine 3.23）极致瘦身基础镜像使用文档

## 1. 镜像特性总览

✅ **版本锁定（donor 固定版本）**

* Node：**22.22.0**（Alpine 3.23 变体）
* Python：**3.12.12**（Alpine 3.23 变体）

✅ **运行时极简**
final 镜像不包含 curl/wget/git/构建工具，减少体积与 CVE 噪音。

✅ **固定时区与中文 locale**

* `TZ=Asia/Shanghai`
* `LANG=zh_CN.UTF-8`
* `LANGUAGE=zh_CN:zh`
* `LC_ALL=zh_CN.UTF-8`

✅ **腾讯源固化（文件配置，无命令副作用）**

* `/etc/npmrc`：`registry=http://mirrors.cloud.tencent.com/npm/`
* `/etc/pip.conf`：`index-url=https://mirrors.tencent.com/pypi/simple` + `trusted-host=mirrors.tencent.com`

✅ **字体渲染支持（不内置字体）**

* 安装 `fontconfig`
* 默认扫描外挂目录：

  * `/fonts`
  * `/usr/share/fonts/custom`
* 容器启动进入 shell 时自动执行 `fc-cache -f`（若存在字体目录）

✅ **便捷命令**

* `/usr/local/bin/now`：输出当前时间含时区偏移
* `/usr/local/bin/ll`：`ls -al` 列目录

✅ **默认 shell**

* `INSTALL_BASH=1` → 进入 bash
* `INSTALL_BASH=0` → 进入 sh（更低 CVE 面）

---

## 2. 构建镜像

### 2.1 普通构建

```bash
docker build -t node-py:latest .
```

### 2.2 构建期验证（推荐首次）

```bash
docker build --build-arg VERIFY=1 -t node-py:verify .
```

> `VERIFY=1` 仅输出到 stdout，不写入镜像文件系统（避免缓存污染）。

### 2.3 发布镜像建议（降低扫描噪音）

生产环境建议默认不带 bash：

```bash
docker build --build-arg INSTALL_BASH=0 -t node-py:prod .
```

可另做一个 debug tag 带 bash：

```bash
docker build --build-arg INSTALL_BASH=1 -t node-py:debug .
```

---

## 3. 运行与环境验证（含 `docker run --rm` 命令）

> 以下命令均为一次性运行（`--rm`），验证镜像环境是否符合预期。

### 3.1 验证 Node / npm / npx / corepack

```bash
docker run --rm node-py:latest node -v
docker run --rm node-py:latest npm -v
docker run --rm node-py:latest npx -v
docker run --rm node-py:latest corepack --version
```

### 3.2 验证 Python / pip

```bash
docker run --rm node-py:latest python3 -V
docker run --rm node-py:latest pip3 -V
```

### 3.3 验证时区与 locale

```bash
docker run --rm node-py:latest date '+%Y-%m-%d %H:%M:%S %z'
docker run --rm node-py:latest sh -lc 'echo "TZ=$TZ LANG=$LANG LC_ALL=$LC_ALL"'
```

### 3.4 验证腾讯镜像源（配置文件固化）

```bash
docker run --rm node-py:latest sh -lc 'cat /etc/npmrc'
docker run --rm node-py:latest sh -lc 'sed -n "1,120p" /etc/pip.conf'
docker run --rm node-py:latest sh -lc 'npm config get registry'
```

### 3.5 验证 now / ll

```bash
docker run --rm node-py:latest now
docker run --rm node-py:latest ll /
```

### 3.6 验证 Python 常用 stdlib（ssl/sqlite/readline 等）

```bash
docker run --rm node-py:latest python3 -c "import ssl,sqlite3,zlib,lzma,bz2,ctypes,readline; print('stdlib-ok')"
```

---

## 4. 字体外挂与验证（fontconfig 生效）

### 4.1 推荐外挂路径

```bash
docker run --rm \
  -v /host/fonts:/usr/share/fonts/custom:ro \
  node-py:latest \
  fc-list | head
```

### 4.2 备用外挂路径

```bash
docker run --rm \
  -v /host/fonts:/fonts:ro \
  node-py:latest \
  fc-list | head
```

### 4.3 说明：为何要 `fontconfig`

* 仅挂载字体文件（ttf/otf）并不保证所有程序都能自动发现字体
* `fontconfig` 是 Linux 上通用字体发现/缓存机制
* 本镜像通过 `/etc/fonts/conf.d/99-local-fonts.conf` 明确声明外挂字体目录，并在进入 shell 时自动 `fc-cache -f`，保证“挂上就能用”。

---

## 5. 常见运行示例

### 5.1 交互进入容器（默认 shell）

```bash
docker run --rm -it node-py:latest
```

* 若镜像带 bash：进入 bash
* 否则进入 sh

### 5.2 Node 应用（示例）

```bash
docker run --rm \
  -v "$PWD":/app \
  -w /app \
  node-py:latest \
  node server.js
```

### 5.3 Python 应用（示例）

```bash
docker run --rm \
  -v "$PWD":/app \
  -w /app \
  node-py:latest \
  python3 app.py
```

---

## 6. 安全扫描低噪音建议

### 6.1 推荐发布策略

* `INSTALL_BASH=0` 作为生产 tag（最少 CVE 面）
* `INSTALL_BASH=1` 作为 debug tag

### 6.2 Trivy 扫描（仅关注高危且有修复版本）

```bash
trivy image --ignore-unfixed --severity HIGH,CRITICAL node-py:prod
```

---

## 7. 自检清单

* [x] Final 基于 `alpine:3.23`
* [x] Node 22.22.0 可用：`node -v`
* [x] npm/npx/corepack 可用（symlink 重建正确）
* [x] Python 3.12.12 可用：`python3 -V`
* [x] pip 可用：`pip3 -V`
* [x] `TZ=Asia/Shanghai` 生效（date 偏移正确）
* [x] `zh_CN.UTF-8` 环境变量设置完成且 musl-locales 安装
* [x] `/etc/npmrc` 与 `/etc/pip.conf` 固化腾讯源
* [x] now/ll 存在且可执行
* [x] `fontconfig` 已安装
* [x] 外挂字体路径 `/fonts`、`/usr/share/fonts/custom` 生效（`fc-list` 可验证）
* [x] 默认非 root 用户运行

---

## 8. 最佳实践提示

1. **生产镜像默认不装 bash**（减少 CVE 噪音）
2. 依赖安装建议在 CI 构建阶段完成（避免运行时 `npm install` / `pip install` 产生不确定性）
3. 字体请外挂，不内置（体积更小、许可证更干净、扫描更稳定）
4. 若你使用需要渲染的库（如 matplotlib、pillow、playwright 截图等），除字体外可能还需要额外系统库（请按实际依赖增补，但保持最小化原则）

---
