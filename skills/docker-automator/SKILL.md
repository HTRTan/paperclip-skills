---
name: docker-automator
description: 提供 Docker CLI 的自动化操作知识、镜像构建与优化、Registry 推送/拉取、容器生命周期管理（启动、停止、日志、清理）以及常见故障排查模板。
---

# Docker Automator 技能

本技能提供日常 Docker 操作的自动化模板、最佳实践及故障排查指南，帮助您高效管理容器化工作流，包括镜像构建、标签管理、容器运行、资源清理和 CI/CD 集成。

## 何时使用

- 需要编写自动化脚本（Bash/Python）管理 Docker 镜像和容器
- 优化 Dockerfile 构建效率与镜像体积
- 批量操作镜像：拉取、标记、推送至多个仓库
- 容器生命周期管理（启动、停止、重启、日志收集）
- 资源清理：删除未使用的镜像、容器、卷、网络
- 将 Docker 操作集成到 CI/CD 流水线（Jenkins/GitHub Actions）
- 调试 Docker 守护进程、网络或卷问题

## 核心概念速查

| 概念 | 说明 | 常用命令/参数 |
|------|------|--------------|
| 镜像层 | Dockerfile 每条指令生成一层 | `docker history` |
| 多阶段构建 | 最终镜像只包含必要产物 | `FROM ... AS builder` |
| 构建缓存 | 利用缓存加速重建 | `--no-cache` 禁用 |
| 镜像标签 | 标识版本和环境 | `tag`, `:` 分隔 |
| Registry | 存储和分发镜像 | Docker Hub, Harbor, ECR, GCR |
| 容器状态 | created, running, exited, paused, dead | `docker ps -a` |
| 卷 | 持久化数据 | `docker volume`, `-v` / `--mount` |
| 网络 | 容器间通信 | `bridge`, `host`, `overlay` |
| 资源限制 | CPU/内存约束 | `--cpus`, `--memory` |
| 健康检查 | 探测容器健康状态 | `HEALTHCHECK` 指令 |

## Docker 自动化脚本模板

### 镜像构建与优化模板 (Bash)

```bash
#!/bin/bash
# 构建并优化镜像，支持多架构、标签、缓存管理

set -e

# 配置变量
IMAGE_NAME="myapp"
VERSION=$(git describe --tags --always --dirty)
REGISTRY="docker.io/username"
PLATFORMS="linux/amd64,linux/arm64"   # 多架构构建

# 构建前清理旧镜像（可选）
docker rmi ${IMAGE_NAME}:${VERSION} 2>/dev/null || true

# 构建（使用 BuildKit 加速）
DOCKER_BUILDKIT=1 docker build \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  --cache-from ${REGISTRY}/${IMAGE_NAME}:latest \
  --tag ${IMAGE_NAME}:${VERSION} \
  --tag ${IMAGE_NAME}:latest \
  --file Dockerfile \
  .

# 多架构构建并推送（需启用 buildx）
if [[ "$1" == "--multi-arch" ]]; then
  docker buildx build \
    --platform ${PLATFORMS} \
    --push \
    --tag ${REGISTRY}/${IMAGE_NAME}:${VERSION} \
    --tag ${REGISTRY}/${IMAGE_NAME}:latest \
    .
fi

# 本地安全扫描（使用 trivy 或 docker scan）
docker scan ${IMAGE_NAME}:${VERSION} || echo "扫描完成"

# 显示镜像大小
docker images ${IMAGE_NAME}:${VERSION}
```

### 容器生命周期管理函数库 (Bash)

```bash
#!/bin/bash
# 容器管理常用函数

# 启动容器（带资源限制和环境变量）
start_container() {
  local name=$1
  local image=$2
  local port_mapping=$3
  local mem_limit="512m"
  local cpu_limit="0.5"

  docker run -d \
    --name ${name} \
    --restart unless-stopped \
    --memory ${mem_limit} \
    --cpus ${cpu_limit} \
    -p ${port_mapping} \
    -e ENV=production \
    -v ${name}-data:/app/data \
    ${image}
}

# 优雅停止并移除容器
stop_and_remove() {
  local name=$1
  docker stop ${name} 2>/dev/null && docker rm ${name} 2>/dev/null
  echo "容器 ${name} 已停止并移除"
}

# 获取容器日志（最后100行并持续输出）
tail_logs() {
  local name=$1
  docker logs --tail 100 -f ${name}
}

# 批量操作：按条件停止容器（如含 "web" 名称）
stop_containers_by_name() {
  local pattern=$1
  docker ps --filter "name=${pattern}" -q | xargs -r docker stop
}

# 进入容器 shell
enter_container() {
  local name=$1
  docker exec -it ${name} /bin/sh   # 若 bash 不存在则用 sh
}
```

### 镜像仓库操作（推送、拉取、同步）

```bash
#!/bin/bash
# 多仓库镜像同步脚本

SOURCE_REGISTRY="registry.example.com"
TARGET_REGISTRIES=("docker.io/myteam" "gcr.io/myproject")

# 拉取、重新标记并推送
sync_image() {
  local image_name=$1
  local source_tag=$2

  docker pull ${SOURCE_REGISTRY}/${image_name}:${source_tag}
  
  for target in "${TARGET_REGISTRIES[@]}"; do
    docker tag ${SOURCE_REGISTRY}/${image_name}:${source_tag} ${target}/${image_name}:${source_tag}
    docker push ${target}/${image_name}:${source_tag}
  done
  
  echo "同步完成"
}

# 批量同步所有版本
sync_all_tags() {
  local image_name=$1
  # 获取远程仓库所有标签（需 skopeo 或 registry API）
  tags=$(skopeo list-tags docker://${SOURCE_REGISTRY}/${image_name} | jq -r '.Tags[]')
  for tag in ${tags}; do
    sync_image ${image_name} ${tag}
  done
}
```

### 资源清理自动化（Cron 任务）

```bash
#!/bin/bash
# 清理未使用的 Docker 资源

# 删除停止的容器
docker container prune -f

# 删除未被任何容器引用的镜像
docker image prune -a -f --filter "until=24h"

# 删除未使用的卷（危险，谨慎使用）
docker volume prune -f

# 删除未使用的网络
docker network prune -f

# 显示释放的磁盘空间
docker system df

# 全系统清理（包括构建缓存）
docker system prune -a -f --volumes
```

## Dockerfile 优化模板

### 多阶段构建（Go 应用）

```dockerfile
# 构建阶段
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp .

# 运行阶段
FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
WORKDIR /root/
COPY --from=builder /app/myapp .
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD ["./myapp", "health"]
CMD ["./myapp"]
```

### Python 应用优化（利用缓存）

```dockerfile
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .

ENV PATH=/root/.local/bin:$PATH
EXPOSE 8000
CMD ["gunicorn", "app:app", "-b", "0.0.0.0:8000"]
```

### Node.js 应用（pnpm + 构建时缓存）

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install --frozen-lockfile

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

## CI/CD 集成模板

### Jenkins Pipeline 中使用 Docker

```groovy
pipeline {
    agent any
    environment {
        DOCKER_REGISTRY = 'docker.io/myteam'
        IMAGE_NAME = 'myapp'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }
    stages {
        stage('Build Image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }
        stage('Test Container') {
            steps {
                sh '''
                    docker run --rm -d --name test-${BUILD_NUMBER} -p 8080:80 ${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 5
                    curl --fail http://localhost:8080/health || exit 1
                    docker stop test-${BUILD_NUMBER}
                '''
            }
        }
        stage('Push to Registry') {
            when { branch 'main' }
            steps {
                withCredentials([string(credentialsId: 'docker-hub-password', variable: 'DOCKER_PASSWORD')]) {
                    sh '''
                        echo ${DOCKER_PASSWORD} | docker login -u myuser --password-stdin
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    '''
                }
            }
        }
    }
    post {
        always {
            sh 'docker logout'
        }
    }
}
```

### GitHub Actions 工作流（构建并推送到多个仓库）

```yaml
name: Docker Build and Push

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

env:
  REGISTRY: docker.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
            ghcr.io/${{ env.IMAGE_NAME }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## 最佳实践与参考

### Dockerfile 最佳实践

| 实践 | 说明 | 示例 |
|------|------|------|
| 选择合适的基础镜像 | 优先 alpine/slim，减小体积 | `FROM alpine:3.19` |
| 合并 RUN 指令 | 减少层数，清理临时文件 | `RUN apt update && apt install -y pkg && rm -rf /var/lib/apt/lists/*` |
| 利用构建缓存 | 按变更频率排序 COPY/ADD | 先 `COPY requirements.txt`，后 `COPY .` |
| 使用多阶段构建 | 最终镜像不含编译工具 | 见上文 Go 示例 |
| 设置非 root 用户 | 提高安全性 | `RUN addgroup -g 1001 -S appgroup && adduser -u 1001 -S appuser -G appgroup` |
| 明确 EXPOSE 端口 | 文档化意图 | `EXPOSE 8080` |
| 添加 HEALTHCHECK | 便于编排系统监控 | `HEALTHCHECK CMD curl --fail http://localhost/ || exit 1` |
| 使用 `.dockerignore` | 避免发送无关文件 | 类似 `.gitignore` |

### 容器运行最佳实践

- 每个容器只运行一个进程（使用 `init` 或 `tini` 处理信号）。
- 始终设置资源限制（`--memory`, `--cpus`），避免资源耗尽。
- 使用命名卷存储持久数据，避免匿名卷。
- 不要在生产环境使用 `--privileged` 或挂载 `/var/run/docker.sock` 除非必要。
- 为容器打标签（`--label`）便于分类和过滤。
- 定期更新基础镜像以修复安全漏洞。
- 使用 `--restart` 策略（`on-failure` 或 `unless-stopped`）提高弹性。

### 安全加固建议

| 操作 | 方法 |
|------|------|
| 镜像扫描 | 使用 `docker scan`、Trivy、Clair 或 Snyk |
| 签名与信任 | Docker Content Trust (`DOCKER_CONTENT_TRUST=1`) |
| 禁止 root 用户 | `USER 1000:1000` |
| 只读根文件系统 | `--read-only`，挂载临时卷用于写入 |
| 限制内核能力 | `--cap-drop=ALL --cap-add=NET_ADMIN` |
| 使用 secrets 管理 | Docker BuildKit secrets (`--secret id=mysecret,src=file`) |

### 故障排查速查表

| 问题 | 诊断命令 | 解决方案 |
|------|----------|----------|
| 镜像构建失败 | `docker build --no-cache --progress=plain` | 查看详细错误，检查网络或依赖 |
| 容器无法启动 | `docker logs <container>` | 检查入口脚本、权限、端口冲突 |
| 磁盘空间不足 | `docker system df` | 运行 `docker system prune -a` |
| 网络不通 | `docker exec <c> ping <target>` | 检查网络模式、防火墙、DNS |
| 卷权限错误 | `ls -l` 在容器内查看 | 调整宿主机目录权限或使用命名卷 |
| 容器退出代码 137 | `docker inspect <c> -f '{{.State.ExitCode}}'` | 通常是 OOM，增加 `--memory` |
| Registry 认证失败 | `docker logout` 后重新登录 | 检查凭据、访问令牌、Registry URL |

### 常用自动化命令片段

```bash
# 批量删除所有 exited 容器
docker rm $(docker ps -aq -f status=exited)

# 强制删除所有 <none> 镜像
docker rmi $(docker images -f "dangling=true" -q) 2>/dev/null

# 给镜像添加新标签
docker tag old:tag new:tag

# 查看容器资源使用情况
docker stats --no-stream

# 复制容器内文件到宿主机
docker cp <container>:/path/file ./host-file

# 基于当前容器状态创建新镜像
docker commit <container> new-image:tag

# 查看镜像分层历史
docker history --no-trunc <image>
```

## 与 CI/CD 流水线集成要点

- **缓存策略**：使用 `--cache-from` 引用远程镜像层，加速构建。
- **标签策略**：建议使用 git commit SHA 作为唯一标签，`latest` 仅用于开发环境。
- **多架构构建**：使用 `docker buildx` 创建 manifest list，一次构建支持 amd64/arm64。
- **推送前测试**：在临时容器中运行健康检查或冒烟测试。
- **回滚机制**：保留至少 5 个旧版本镜像，通过更改标签快速回滚。

## 扩展资源

| 主题 | 链接 |
|------|------|
| Dockerfile 官方参考 | [https://docs.docker.com/engine/reference/builder/](https://docs.docker.com/engine/reference/builder/) |
| Docker CLI 命令行 | [https://docs.docker.com/engine/reference/commandline/cli/](https://docs.docker.com/engine/reference/commandline/cli/) |
| Buildx 多架构构建 | [https://docs.docker.com/build/building/multi-platform/](https://docs.docker.com/build/building/multi-platform/) |
| 最佳实践指南 | [https://docs.docker.com/develop/develop-images/dockerfile_best-practices/](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) |
| 安全扫描 | [https://docs.docker.com/engine/scan/](https://docs.docker.com/engine/scan/) |
| Docker Compose 自动化 | 可结合 `docker compose up -d` 进行多容器编排 |