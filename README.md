# CoreDNS Ansible Role

这是一个用于部署 CoreDNS DNS 服务器的 Ansible Role,支持 Docker Compose 方式部署,并包含 View 插件用于智能 DNS 解析。

## 目录

- [功能特性](#功能特性)
- [先决条件](#先决条件)
- [镜像构建](#镜像构建)
- [快速开始](#快速开始)
- [变量说明](#变量说明)
- [使用示例](#使用示例)
- [Docker 安装](#docker-安装)
- [高级配置](#高级配置)
- [运维操作](#运维操作)
- [常见问题](#常见问题)

## 功能特性

- **智能解析 (View-based DNS)**: 基于客户端 IP 地址返回不同的解析结果
- **正则匹配**: 支持泛域名解析 (如 `*-dev.demo.cn`)
- **域名转发**: 特定域名使用指定的上游 DNS 服务器
- **Hosts 文件**: 静态域名解析
- **配置热加载**: 支持 `SIGUSR1` 信号重载配置,无需重启服务
- **日志轮转**: 自动管理 DNS 查询日志
- **健康检查**: 内置健康检查端点
- **Prometheus 监控**: 导出 DNS 查询指标
- **Docker 自动安装**: 可选的 Docker 自动安装功能

## 先决条件

### 1. 基础环境

- **操作系统**: Ubuntu 22.04 (推荐) 或其他支持 Docker 的 Linux 发行版
- **Ansible**: 2.9+
- **Python**: 3.6+

### 2. Docker 环境

#### 方式一: 使用 Role 自动安装 Docker

本 Role 支持自动安装 Docker,只需在变量中启用:

```yaml
docker_install_enabled: true
```

详见 [Docker 安装](#docker-安装) 章节。

#### 方式二: 手动安装 Docker

```bash
# Ubuntu 22.04
curl -fsSL https://get.docker.com | bash
systemctl enable docker
systemctl start docker
```

### 3. Nexus 私有镜像仓库 (推荐)

为了高效管理 CoreDNS 镜像,**强烈推荐**部署 Nexus Repository Manager 并配置 Docker 仓库。

#### Nexus 部署

```bash
# 使用 Docker 快速部署 Nexus
docker run -d -p 8081:8081 -p 4002:4002 -p 4003:4003 \
  --name nexus \
  -v nexus-data:/nexus-data \
  sonatype/nexus3:latest
```

访问 `http://<your-server>:8081`,默认用户名 `admin`,密码在 `/nexus-data/admin.password`。

#### 配置 Docker 仓库

在 Nexus 中创建以下 Docker 仓库:

1. **Docker (proxy)** - 端口 4002
   - 名称: `docker-proxy`
   - 远程地址: `https://registry-1.docker.io`
   - 用途: 缓存 Docker Hub 镜像

2. **Docker (hosted)** - 端口 4003
   - 名称: `docker-hosted`
   - HTTP 端口: 4003
   - 用途: 存储自定义构建的镜像 (如 `coredns:1.14.1-view`)

3. **Docker (group)** - 端口 4002
   - 名称: `docker-group`
   - 成员仓库: `docker-proxy`, `docker-hosted`
   - HTTP 端口: 4002
   - 用途: 统一访问入口

#### 配置 Docker 客户端

在构建节点上配置 Docker 信任 Nexus 仓库:

```bash
# /etc/docker/daemon.json
{
  "insecure-registries": [
    "your-docker-repo.demo.com:4002",
    "your-docker-repo.demo.com:4003"
  ],
  "registry-mirrors": [
    "https://your-docker-repo.demo.com:4002"
  ]
}
```

重启 Docker:

```bash
systemctl restart docker
```

登录 Nexus Docker 仓库:

```bash
docker login your-docker-repo.demo.com:4003
```

## 镜像构建

CoreDNS 镜像包含 View 插件,需要从源码构建。

### 方法一: 使用 Makefile (推荐)

本 Role 提供了完整的 Makefile 用于镜像构建。

#### 1. 准备构建环境

```bash
# 在任意节点上,确保已部署 coredns role 或复制以下文件:
# - Dockerfile
# - Makefile
# - plugin.cfg

cd /root/coredns  # 或其他部署目录
```

#### 2. 修改配置 (可选)

编辑 `Makefile`,调整以下变量:

```makefile
REGISTRY := your-docker-repo.demo.com:4003  # Nexus Docker 仓库地址
IMAGE_NAME := coredns
COREDNS_VERSION := 1.14.1
GO_VERSION := 1.24
```

#### 3. 构建镜像

```bash
# 查看所有可用命令
make help

# 构建镜像
make build

# 无缓存构建
make build-no-cache
```

#### 4. 推送到 Nexus

```bash
# 登录 Nexus (首次)
make login

# 推送镜像
make push

# 推送并打 latest 标签
make push-latest

# 一键构建并推送
make release
```

#### 5. 验证镜像

```bash
# 查看镜像信息
make info

# 从 Nexus 拉取验证
docker pull your-docker-repo.demo.com:4003/coredns:1.14.1-view
```

### 方法二: 手动构建

```bash
# 构建镜像
docker build \
  --build-arg COREDNS_VERSION=1.14.1 \
  --build-arg GO_VERSION=1.24 \
  -t your-docker-repo.demo.com:4003/coredns:1.14.1-view \
  .

# 推送到 Nexus
docker push your-docker-repo.demo.com:4003/coredns:1.14.1-view
```

### 构建说明

- **构建时间**: 首次构建约 5-10 分钟 (取决于网络速度)
- **镜像大小**: 约 50MB (多阶段构建,运行时镜像仅包含必要组件)
- **插件配置**: 通过 `plugin.cfg` 控制包含哪些插件
- **国内加速**: 使用 `GOPROXY=https://goproxy.cn` 加速 Go 模块下载

## 快速开始

### 1. 准备 Playbook

创建 `playbook-coredns.yaml`:

```yaml
- hosts: coredns
  serial: 1  # 逐台执行,确保 DNS 服务连续性
  vars:
    coredns_version: "1.14.1-view"
    coredns_image: "your-docker-repo.demo.com:4002/coredns"

    # 基础配置 示例
    coredns_upstream_servers:
      - 192.168.0.100
      - 192.168.0.101

    # Hosts 静态解析 示例
    coredns_hosts_entries:
      - 10.21.70.81 git-dev.demo.cn

  roles:
    - coredns
```

### 2. 执行部署

```bash
# 部署到所有 coredns 主机
ansible-playbook -i inventory playbook-coredns.yaml

# 指定标签,仅更新配置
ansible-playbook -i inventory playbook-coredns.yaml --tags config

# 仅构建文件部署 (用于镜像构建)
ansible-playbook -i inventory playbook-coredns.yaml --tags build
```

### 3. 验证部署

```bash
# 检查服务状态
docker-compose -f /root/coredns/docker-compose.yml ps

# 测试 DNS 解析
nslookup git-dev.demo.cn <coredns-server-ip>

# 查看日志
docker logs coredns
```

## 变量说明

### 核心配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `coredns_deploy_dir` | `/root/coredns` | 部署目录 |
| `coredns_logs_dir` | `/data/docker/coredns/logs` | 日志目录 |
| `coredns_image` | `your-docker-repo.demo.com:4002/coredns` | 镜像地址 |
| `coredns_version` | `1.14.1-view` | 镜像版本 |
| `coredns_upstream_servers` | `["192.168.0.100", "192.168.0.101"]` | 上游 DNS 服务器 |
| `coredns_cache_ttl` | `300` | 缓存时间 (秒) |

### 端口配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `coredns_dns_port` | `53` | DNS 服务端口 |
| `coredns_health_port` | `8080` | 健康检查端口 |
| `coredns_metrics_port` | `9153` | Prometheus 指标端口 |

### 功能配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `coredns_hosts_entries` | `[]` | Hosts 静态解析列表 |
| `coredns_forward_zones` | `[]` | 域名转发规则 |
| `coredns_view_enabled` | `false` | 是否启用视图解析 |
| `coredns_view_rules` | `[]` | 视图解析规则 |
| `coredns_regex_rules` | `[]` | 正则匹配规则 |
| `coredns_validate_config` | `true` | 是否验证配置 |
| `coredns_reload_method` | `reload` | 重载方式 (`reload` 或 `restart`) |
| `coredns_sync_build_files` | `false` | 是否同步构建文件 |

### Docker 配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `docker_daemon_config` | `true` | 是否配置 daemon.json |
| `docker_registry_mirrors` | 见 defaults | Docker 镜像加速器 |
| `docker_insecure_registries` | 见 defaults | 非 HTTPS 仓库列表 |

### Docker 安装配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `docker_install_enabled` | `false` | 是否自动安装 Docker |
| `docker_install_ce_version` | `5:24.0.9-1~ubuntu.22.04~jammy` | Docker CE 版本 |
| `docker_install_containerd_version` | `1.6.33-1` | containerd 版本 |
| `docker_install_user` | `ubuntu` | 添加到 docker 组的用户 |
| `docker_install_proxy_enabled` | `false` | 是否使用代理下载 docker-compose |
| `docker_install_proxy_url` | `http://<your-proxy-server>:8001` | 代理地址 |

## 使用示例

### 示例 1: 基础 Hosts 解析

```yaml
coredns_hosts_entries:
  - 10.21.70.81 git-dev.demo.cn
```

### 示例 2: 特定域名转发

```yaml
coredns_forward_zones:
  # GitHub 使用阿里 DNS
  - domains:
      - github.com
      - githubusercontent.com
    upstream: 223.5.5.5

  # Google 使用阿里 DNS
  - domains:
      - google.com
      - googleapis.com
    upstream: 223.5.5.5
```

### 示例 3: 视图智能解析

```yaml
coredns_view_enabled: true
coredns_view_rules:
  - domain: "test2026.demo.cn"
    views:
      # 内网访问
      - name: "internal"
        cidr: "10.21.0.0/16"
        ttl: 60
        ips:
          - "10.21.70.81"

      # 阿里云访问
      - name: "aliyun"
        cidr: "10.100.1.0/24"
        ttl: 60
        ips:
          - "192.168.70.81"

    # 默认解析
    default:
      ttl: 60
      ips:
        - "10.21.70.61"
```

### 示例 4: 正则泛域名解析

```yaml
coredns_regex_rules:
  - zone: "demo.cn"
    rules:
      # 匹配 *-dev.demo.cn
      - pattern: "^[a-zA-Z0-9]+-dev[.]demo[.]cn[.]$"
        comment: "泛域名 *-dev.demo.cn 解析到开发环境"
        ttl: 60
        ips:
          - "10.21.70.81"

      # 匹配 *-ops.demo.cn
      - pattern: "^[a-zA-Z0-9]+-ops[.]demo[.]cn[.]$"
        comment: "泛域名 *-ops.demo.cn 解析到生产环境"
        ttl: 60
        ips:
          - "10.100.1.212"
```

## Docker 安装

本 Role 支持自动安装 Docker,使用优化过的安装脚本。

### 启用 Docker 自动安装

在 Playbook 或 Inventory 中设置:

```yaml
# 启用 Docker 安装
docker_install_enabled: true

# 可选: 自定义配置
docker_install_ce_version: "5:24.0.9-1~ubuntu.22.04~jammy"
docker_install_containerd_version: "1.6.33-1"
docker_install_user: "ubuntu"
```

### 工作原理

1. **检测 Docker**: 首先检查 Docker 是否已安装
2. **部署脚本**: 从 template 生成 `docker_install.sh` 脚本
3. **执行安装**: 运行脚本安装 Docker CE 和相关组件
4. **清理脚本**: 安装完成后删除临时脚本

### 安装内容

- Docker CE (指定版本,防止意外升级)
- Docker CLI
- containerd
- docker-buildx-plugin
- docker-compose-plugin
- docker-compose (独立二进制)

### 代理配置

如果下载 docker-compose 需要代理:

```yaml
docker_install_proxy_enabled: true
docker_install_proxy_url: "http://<your-proxy-server>:8001"
```

### 仅安装 Docker (不部署 CoreDNS)

```bash
ansible-playbook -i inventory playbook-coredns.yaml --tags docker-install
```

## 高级配置

### 配置热加载

CoreDNS 支持 `SIGUSR1` 信号重载配置,无需重启容器:

```bash
# 修改配置后重载
ansible-playbook -i inventory playbook-coredns.yaml --tags config

# 配置会自动触发 reload handler
```

手动重载:

```bash
docker kill --signal=SIGUSR1 coredns
```

### 日志轮转

默认配置 30 天日志轮转,可调整:

```yaml
coredns_logrotate_days: 14  # 保留 14 天
```

### Prometheus 监控

CoreDNS 在 `:9153/metrics` 端点导出监控指标:

```yaml
# Prometheus 配置
scrape_configs:
  - job_name: 'coredns'
    static_configs:
      - targets: ['coredns-server:9153']
```

### 健康检查

健康检查端点: `http://<coredns-server>:8080/health`

```bash
curl http://localhost:8080/health
```

## 运维操作

### 使用 Makefile (推荐)

在部署目录 (`/root/coredns`) 中:

```bash
# 启动服务
make up

# 停止服务
make down

# 重启服务
make restart

# 重载配置 (不重启)
make reload

# 查看日志
make logs

# 查看状态
make status

# 测试 DNS 解析
make test

# 测试视图解析
make test-view
```

### 使用 Ansible

```bash
# 更新配置
ansible-playbook -i inventory playbook-coredns.yaml --tags config

# 重启服务
ansible-playbook -i inventory playbook-coredns.yaml --tags docker

# 验证配置
ansible-playbook -i inventory playbook-coredns.yaml --tags validate
```

### 手动操作

```bash
# 查看容器状态
docker-compose -f /root/coredns/docker-compose.yml ps

# 重载配置
docker kill --signal=SIGUSR1 coredns

# 重启服务
docker-compose -f /root/coredns/docker-compose.yml restart

# 查看日志
docker logs -f coredns
```

## 常见问题

### 1. 镜像构建失败

**问题**: 构建时无法下载 Go 模块

**解决**:
```bash
# 确保设置了 Go 代理，或者使用 http://<your-proxy-server>:8001
export GOPROXY=https://goproxy.cn,direct

# 或在 Dockerfile 中已配置 (已内置)
```

### 2. DNS 解析不生效

**问题**: 配置更新后解析结果未变化

**解决**:
```bash
# 重载配置
docker kill --signal=SIGUSR1 coredns

# 或使用 Makefile
make reload

# 检查配置语法
docker exec coredns /usr/local/bin/coredns -plugins
```

### 3. 视图解析不工作

**问题**: 所有客户端返回相同结果

**解决**:
```yaml
# 检查 CIDR 配置是否正确
coredns_view_rules:
  - domain: "example.com"
    views:
      - name: "internal"
        cidr: "192.168.1.0/24"  # 确保 CIDR 格式正确
        ips: ["192.168.1.10"]

# 确保启用了视图功能
coredns_view_enabled: true
```

### 4. Docker 安装失败

**问题**: 自动安装 Docker 时出错

**解决**:
```bash
# 检查网络连接
curl -I https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu/

# 检查系统版本
lsb_release -cs  # 应返回 jammy (Ubuntu 22.04)

# 手动执行安装脚本调试
bash -x /tmp/docker_install.sh
```

### 5. Nexus 镜像推送失败

**问题**: `docker push` 失败

**解决**:
```bash
# 确保已登录
docker login your-docker-repo.demo.com:4003

# 检查 daemon.json 配置
cat /etc/docker/daemon.json

# 确保仓库在 insecure-registries 列表中
{
  "insecure-registries": ["your-docker-repo.demo.com:4003"]
}

# 重启 Docker
systemctl restart docker
```

### 6. 配置验证失败

**问题**: `CONFIG_INVALID` 错误

**解决**:
```bash
# 进入容器检查配置
docker exec -it coredns sh
/usr/local/bin/coredns -plugins
/usr/local/bin/coredns -conf /etc/coredns/config/Corefile -dry-run

# 检查配置文件语法
cat /root/coredns/config/Corefile
```

### 7. 日志权限问题

**问题**: 日志目录无写入权限

**解决**:
```bash
# 确保目录权限正确
chown -R 1000:1000 /data/docker/coredns/logs
chmod 755 /data/docker/coredns/logs
```

## 文件清单

```
roles/coredns/
├── README.md                           # 本文档
├── defaults/
│   └── main.yml                        # 默认变量配置
├── tasks/
│   └── main.yml                        # 主要任务
├── templates/
│   ├── docker-compose.yml.j2           # Docker Compose 配置
│   ├── Corefile.j2                     # CoreDNS 主配置文件
│   ├── hosts.j2                        # Hosts 文件
│   ├── daemon.json.j2                  # Docker daemon 配置
│   ├── Makefile.j2                     # 镜像构建和运维工具
│   ├── logrotate.coredns.j2            # 日志轮转配置
│   ├── docker_install.sh.j2            # Docker 安装脚本 (NEW)
│   └── zones/
│       ├── views.conf.j2               # 视图解析配置
│       ├── forward-zones.conf.j2       # 域名转发配置
│       └── regex.conf.j2               # 正则匹配配置
├── files/
│   ├── Dockerfile                      # CoreDNS 镜像构建文件
│   └── plugin.cfg                      # 插件配置
├── handlers/
│   └── main.yml                        # 事件处理器
└── playbook-coredns.yaml.sample        # Playbook 示例

```

## 版本历史

- **v1.2.0** (2026-01-31)
  - 新增 Docker 自动安装功能
  - 将 docker_install.sh 转换为 template
  - 添加 Docker 安装配置变量
  - 优化文档,添加 Nexus 配置说明

- **v1.1.0**
  - 支持 CoreDNS 1.14.1 with View plugin
  - 添加正则匹配泛域名解析
  - 支持配置热加载 (SIGUSR1)

- **v1.0.0**
  - 初始版本
  - 基础 DNS 解析功能

## 许可



## 联系方式

