# CoreDNS with view plugin
# 多阶段构建：编译阶段 + 运行阶段

ARG COREDNS_VERSION=1.14.1
ARG GO_VERSION=1.24
ARG BUILD_PLATFORM=linux/amd64

# ============================================================
# 阶段1: 构建 CoreDNS (包含 view 插件)
# ============================================================
FROM --platform=${BUILD_PLATFORM} golang:${GO_VERSION}-alpine AS builder

ARG COREDNS_VERSION

# 安装构建依赖
RUN apk add --no-cache git make

# 设置 Go 代理 (国内加速)
ENV GOPROXY=https://goproxy.cn,direct
ENV CGO_ENABLED=0

WORKDIR /build

# 克隆 CoreDNS 源码
RUN git clone --depth 1 --branch v${COREDNS_VERSION} https://github.com/coredns/coredns.git .

# 复制自定义插件配置
COPY plugin.cfg /build/plugin.cfg

# 生成插件代码并构建
RUN go generate && \
    go build -o coredns -ldflags="-s -w -X github.com/coredns/coredns/coremain.GitCommit=$(git rev-parse --short HEAD)"

# ============================================================
# 阶段2: 运行时镜像
# ============================================================
ARG BUILD_PLATFORM=linux/amd64
FROM --platform=${BUILD_PLATFORM} alpine:3.19

# 安装运行时依赖
RUN apk add --no-cache ca-certificates tzdata bind-tools && \
    adduser -D -H -u 1000 coredns

# 复制构建产物
COPY --from=builder /build/coredns /usr/local/bin/coredns

# 创建配置目录
RUN mkdir -p /etc/coredns /var/log/coredns && \
    chown -R coredns:coredns /etc/coredns /var/log/coredns

# 设置时区
ENV TZ=Asia/Shanghai

# 暴露端口
EXPOSE 53/udp 53/tcp 9153 8080

# 健康检查
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD nslookup health.check 127.0.0.1 || exit 1

USER coredns

ENTRYPOINT ["/usr/local/bin/coredns"]
CMD ["-conf", "/etc/coredns/config/Corefile"]
