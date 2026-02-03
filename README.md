## 📦 Alpine Multi-Language Base Images

一个专注于构建**高性能、多架构、最小体积** Alpine Linux 基础 Docker 镜像的项目，覆盖主流开发语言运行环境，包括：

* Java
* Python
* Node.js
* PHP
* 以及更多语言版本持续扩展中

所有镜像均基于 Alpine Linux 构建，并针对多架构平台进行统一优化。

---

### ✨ 核心特性

✅ 极简 Alpine 基础系统，体积小、启动快
✅ 支持多语言、多版本并行维护
✅ 原生多架构支持：

* linux/amd64
* linux/arm64/v8
* 可扩展支持 arm、riscv64、ppc64le、s390x 等平台

✅ 工程级 CI/CD 自动化构建与发布
✅ 智能缓存加速构建（GitHub Actions Cache）
✅ 供应链安全增强（SBOM + Provenance）

---

### 📁 项目结构

```
images/
  <language>/
    <version>/
      Dockerfile
```

示例：

```
images/python/3.12/Dockerfile
images/java/17/Dockerfile
images/node/20/Dockerfile
```

每个目录即对应一个可独立构建与发布的基础镜像。

---

### 🚀 构建与发布机制

本项目采用 GitHub Actions + Docker Buildx 实现：

* 手动点名构建目标镜像
* 多架构一次性生成 manifest
* 默认构建 amd64 + arm64/v8
* 支持按需扩展至全平台
* 自动推送至 DockerHub 公共仓库

并提供：

* dry-run 构建验证模式
* 发布级安全证明（SBOM + Provenance）

---

### 🏷 镜像命名规范

```
<dockerhub-namespace>/<project-name>-<language>:<version>
```

示例：

```
acme/alpine-base-python:3.12
acme/alpine-base-java:17
acme/alpine-base-node:20
```

---

### 🎯 设计目标

* 提供稳定、统一规范的 Alpine 基础运行环境
* 降低多语言项目 Docker 构建复杂度
* 原生支持云原生与 ARM 生态
* 面向生产环境的安全与性能优化

---

### 🔐 安全与最佳实践

* 不在仓库中存放任何敏感信息
* 所有发布凭据通过 GitHub Secrets 管理
* 使用官方推荐 Buildx 多架构构建流程
* 提供供应链安全元数据支持

---

### 🎯 提示词

```
你是一名具备 DevOps 架构设计、Docker 多架构构建、CI/CD 性能优化经验的专家。

请为一个专门维护 Alpine 基础 Docker 镜像的项目设计或优化 GitHub Actions workflow，要求：

项目结构固定为：
images/<language>/<version>/Dockerfile

核心目标：
- 仅手动触发构建（workflow_dispatch）
- 必须支持点名构建一个或多个镜像目录
- 同时支持多架构构建（可选配置 platforms，默认 linux/amd64,linux/arm64/v8）
- 使用 Docker Buildx + QEMU
- 启用智能缓存（GHA cache，按 language-version scope）
- 镜像仓库为 public，不包含任何敏感信息
- DockerHub 凭据使用 GitHub Secrets 管理
- 自动生成 Docker tag（默认只使用版本号作为 tag）
- 可选 latest 与额外 tag
- dry-run 时不 push，并优化构建速度
- 发布时启用 provenance 与 SBOM attestation
- 避免 deprecated 用法（如 docker buildx install）

请输出：
1. 完整可运行的 workflow YAML
2. 关键设计说明与性能优化点
3. 推荐的镜像命名与发布策略

要求方案工程化、稳定、可扩展，并符合 Docker 官方最佳实践。
```
