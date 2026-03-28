# OpenClaw Custom Docker

本仓库提供 OpenClaw 的自定义 Docker 构建与运行配置，包含：
- 多阶段构建（运行时镜像精简，不包含构建工具）
- 可选的浏览器与 Docker CLI 依赖
- 为国内环境准备的插件安装与镜像源切换选项

> 说明：此仓库只包含 Dockerfile 与构建工作流，实际构建需要配合 OpenClaw 源码使用。

## 目录结构

- `Dockerfile`：自定义镜像构建主文件（多阶段构建）
- `.github/workflows/openclaw-release.yml`：镜像发布工作流

## 构建镜像（本地）

本仓库的 Dockerfile 期望构建上下文是 **OpenClaw 源码目录**。可按下面步骤手动构建：

```bash
OPENCLAW_TAG=2026.3.12
mkdir -p _src/openclaw
curl -fsSL "https://github.com/openclaw/openclaw/archive/refs/tags/v${OPENCLAW_TAG}.tar.gz" \
  | tar -xz -C _src/openclaw --strip-components=1

mkdir -p _src/openclaw/openclaw
cp Dockerfile docker-entrypoint.sh _src/openclaw/openclaw/

docker build -f _src/openclaw/openclaw/Dockerfile -t openclaw:${OPENCLAW_TAG} _src/openclaw
```

### 常用构建参数

- `OPENCLAW_VARIANT=slim`：使用 bookworm-slim 基础镜像
- `OPENCLAW_EXTENSIONS="ext1 ext2"`：仅为指定扩展复制依赖元数据
- `OPENCLAW_DOCKER_APT_PACKAGES="..."`：安装额外系统依赖
- `OPENCLAW_INSTALL_BROWSER=1`：预装 Chromium（便于浏览器自动化）
- `OPENCLAW_INSTALL_DOCKER_CLI=1`：安装 Docker CLI（用于 sandbox 相关场景）

示例：
```bash
docker build \
  --build-arg OPENCLAW_VARIANT=slim \
  --build-arg OPENCLAW_INSTALL_BROWSER=1 \
  --build-arg OPENCLAW_DOCKER_APT_PACKAGES="ffmpeg git curl" \
  -f _src/openclaw/openclaw/Dockerfile \
  -t openclaw:${OPENCLAW_TAG} \
  _src/openclaw
```

## 运行镜像

```bash
docker run --rm -p 18789:18789 openclaw:${OPENCLAW_TAG}
```

- 入口：`https://localhost:18789`
- 网关进程在容器内默认绑定 `127.0.0.1:18789`
- 健康检查端点：`/healthz`、`/readyz`（别名 `/health`、`/ready`）

> **重要提示**：镜像默认将网关绑定到 127.0.0.1。若使用 Docker bridge 网络（`-p` 映射端口）对外访问，需：
> - 使用 `--network host`，或
> - 覆盖默认命令并将网关绑定到局域网地址（如 `--bind lan`），并配置认证信息。

### 运行时环境变量

- `CADDY_HTTPS_PORT`：Caddy HTTPS 监听端口（默认 8443）
- `CADDY_SITE_ADDRESS`：站点地址（默认 127.0.0.1）

示例：
```bash
docker run --rm -p 9443:9443 \
  -e CADDY_HTTPS_PORT=9443 \
  -e CADDY_SITE_ADDRESS=localhost \
  openclaw:${OPENCLAW_TAG}
```

## 发布镜像（GitHub Actions）

`.github/workflows/openclaw-release.yml` 会在推送 tag 或手动触发时：
- 下载指定版本的 OpenClaw 源码
- 注入本仓库的 Dockerfile
- 使用 Buildx 构建并推送多架构镜像（默认 `linux/amd64,linux/arm64/v8`）

默认推送到：`ghcr.io/repobor/openclaw`。
