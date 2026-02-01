# ============================================
# 基础构建环境（保持完整版，需要构建工具链）
# ============================================
FROM node:24.10 AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable && corepack prepare pnpm@9.15.2 --activate

WORKDIR /app
COPY . .
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --frozen-lockfile --ignore-scripts

# ============================================
# 前端构建阶段
# ============================================
FROM base AS frontend-build
ARG VITE_DEFAULT_SERVER=http://127.0.0.1:18010
ARG VITE_FULLSTACK=false
ENV VITE_DEFAULT_SERVER=${VITE_DEFAULT_SERVER}
ENV VITE_FULLSTACK=${VITE_FULLSTACK}
RUN pnpm run build:webui

# ============================================
# 后端构建阶段
# ============================================
FROM base AS backend-build
RUN pnpm run build:base && pnpm run --filter bililive-cli build && cd packages/CLI && node ./scripts/removeDev.js && npm install --force

# 下载二进制依赖（优化：合并 apt 操作，减少层）
RUN apt-get update && apt-get install -y --no-install-recommends curl ca-certificates unzip && \
    # 下载字体
    curl -L -o /usr/local/share/fonts/NotoSansSC-Regular.ttf https://github.com/renmu123/biliLive-tools/releases/download/0.2.1/NotoSansSC-Regular.ttf && \
    # 下载 BaiduPCS-Go
    mkdir -p /app/baidu-temp && \
    ARCH=$(uname -m) && \
    if [ "$ARCH" = "x86_64" ]; then \
        curl -L -o /app/baidu-temp/BaiduPCS-Go.zip https://github.com/qjfoidnh/BaiduPCS-Go/releases/download/v4.0.0/BaiduPCS-Go-v4.0.0-linux-amd64.zip; \
    elif [ "$ARCH" = "aarch64" ]; then \
        curl -L -o /app/baidu-temp/BaiduPCS-Go.zip https://github.com/qjfoidnh/BaiduPCS-Go/releases/download/v4.0.0/BaiduPCS-Go-v4.0.0-linux-arm64.zip; \
    fi && \
    unzip /app/baidu-temp/BaiduPCS-Go.zip -d /app/baidu-temp && \
    mv /app/baidu-temp/BaiduPCS-Go-*/BaiduPCS-Go /app/packages/app/resources/bin/BaiduPCS-Go && \
    chmod +x /app/packages/app/resources/bin/BaiduPCS-Go && \
    # 下载 aliyunpan
    mkdir -p /app/aliyun-temp && \
    if [ "$ARCH" = "x86_64" ]; then \
        curl -L -o /app/aliyun-temp/aliyunpan.zip https://github.com/tickstep/aliyunpan/releases/download/v0.3.7/aliyunpan-v0.3.7-linux-amd64.zip; \
    elif [ "$ARCH" = "aarch64" ]; then \
        curl -L -o /app/aliyun-temp/aliyunpan.zip https://github.com/tickstep/aliyunpan/releases/download/v0.3.7/aliyunpan-v0.3.7-linux-arm64.zip; \
    fi && \
    unzip /app/aliyun-temp/aliyunpan.zip -d /app/aliyun-temp && \
    mv /app/aliyun-temp/aliyunpan-*/aliyunpan /app/packages/app/resources/bin/aliyunpan && \
    chmod +x /app/packages/app/resources/bin/aliyunpan && \
    # 清理临时文件和 apt 缓存（关键优化）
    rm -rf /app/baidu-temp /app/aliyun-temp && \
    apt-get purge -y unzip && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN pnpm run install:bin

# ============================================
# 目标1: 纯前端镜像（Caddy 已基于 Alpine，无需优化）
# ============================================
FROM caddy:latest AS frontend

WORKDIR /app
RUN mkdir -p public data bin

COPY --from=frontend-build /app/packages/app/out/renderer /app/public
COPY --from=base /app/docker/Caddyfile ./

ENV NODE_ENV=production

EXPOSE 3000

CMD ["caddy", "run", "--config", "/app/Caddyfile"]

# ============================================
# 目标2: 纯后端镜像（核心优化点：node:24.10 → node:24.10-slim）
# ============================================
FROM node:24.10-slim AS backend  # ✅ 关键优化：使用 slim 镜像

WORKDIR /app

RUN mkdir -p data bin video

# 复制构建产物
COPY --from=backend-build /app/packages/CLI/lib /app
COPY --from=backend-build /app/packages/CLI/node_modules /app/node_modules
COPY --from=backend-build /app/packages/app/resources/bin /app/bin

# 安装最小运行时依赖（关键：--no-install-recommends + 清理缓存）
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libfontconfig1 \
        ffmpeg \          # 显式安装 ffmpeg（项目必需）
        fontconfig && \
    apt-get install -y /app/bin/audiowaveform.deb && \
    rm -rf /var/lib/apt/lists/* && \
    # 清理 deb 包（已安装）
    rm -f /app/bin/audiowaveform.deb

# 复制字体（优化路径：使用标准字体目录）
COPY --from=backend-build /usr/local/share/fonts/NotoSansSC-Regular.ttf /usr/share/fonts/truetype/noto/NotoSansSC-Regular.ttf
RUN fc-cache -f -v  # 刷新字体缓存

COPY --from=backend-build /app/docker/config.json ./

# 创建非 root 用户（安全最佳实践）
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

USER appuser

ENV NODE_ENV=production
ENV IS_DOCKER=true
ENV BAIDUPCS_GO_CONFIG_DIR=/app/data
ENV ALIYUNPAN_CONFIG_DIR=/app/data
ENV TZ=Asia/Shanghai

EXPOSE 18010

CMD ["node", "index.cjs", "server", "--config", "config.json"]

# ============================================
# 目标3: 全栈镜像 (默认)（核心优化点：node:24.10 → node:24.10-slim）
# ============================================
FROM node:24.10-slim AS fullstack  # ✅ 关键优化：使用 slim 镜像

WORKDIR /app

# 创建必要的目录
RUN mkdir -p /app/data /app/video /app/bin /app/backend /app/public

# 复制后端
COPY --from=backend-build /app/packages/CLI/lib /app/backend
COPY --from=backend-build /app/packages/CLI/node_modules /app/backend/node_modules
COPY --from=backend-build /app/packages/app/resources/bin /app/bin

# 安装最小运行时依赖（合并所有 apt 操作，减少层）
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libfontconfig1 \
        ffmpeg \
        fontconfig \
        debian-keyring \
        debian-archive-keyring \
        apt-transport-https \
        curl && \
    apt-get install -y /app/bin/audiowaveform.deb && \
    # 安装 Caddy（使用官方脚本简化）
    curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg && \
    curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list && \
    apt-get update && \
    apt-get install -y caddy && \
    # 清理所有缓存和临时文件（关键优化）
    rm -f /app/bin/audiowaveform.deb && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 复制字体
COPY --from=backend-build /usr/local/share/fonts/NotoSansSC-Regular.ttf /usr/share/fonts/truetype/noto/NotoSansSC-Regular.ttf
RUN fc-cache -f -v

# 复制前端
COPY --from=frontend-build /app/packages/app/out/renderer /app/public

# 复制配置
COPY --from=base /app/docker/config.json /app/backend/config.json
COPY --from=base /app/docker/fullstack-Caddyfile /etc/caddy/Caddyfile
COPY --from=base /app/docker/start.sh /app/start.sh

# 创建非 root 用户
RUN chmod +x /app/start.sh && \
    useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app && \
    chown -R appuser:appuser /etc/caddy

USER appuser

ENV NODE_ENV=production
ENV IS_DOCKER=true
ENV BAIDUPCS_GO_CONFIG_DIR=/app/data
ENV ALIYUNPAN_CONFIG_DIR=/app/data
ENV TZ=Asia/Shanghai

EXPOSE 3000

CMD ["/app/start.sh"]
