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

#### 配置Nginx代理，实现HTTPS访问

```nginx
server {
    listen 4012;
    listen 4002 ssl;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4:!DH:!DHE;
    ssl_prefer_server_ciphers on;
    ssl_certificate ssl/demo_com.crt;
    ssl_certificate_key ssl/demo_com.key;
    server_name your-docker-repo.demo.com;
    # disable any limits to avoid HTTP 413 for large image uploads
    client_max_body_size 0;
    # required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)
    chunked_transfer_encoding on;
    # Docker /v2 and /v1 (for search) requests
    location /v2 {
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto "https";
        proxy_pass http://nexus3/repository/docker-group/$request_uri;
    }
    location /v1 {
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto "https";
        proxy_pass http://nexus3/repository/docker-group/$request_uri;
    }

    location / {
        proxy_pass https://docker-groups_ssl_upstream;
        proxy_set_header X-Forwarded-Proto "https";
        include include/proxy.conf; # 代理配置略
    }
}

server {
    listen 4013;
    listen 4003 ssl;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4:!DH:!DHE;
    ssl_prefer_server_ciphers on;
    ssl_certificate ssl/demo_com.crt;
    ssl_certificate_key ssl/demo_com.key;
    server_name your-docker-repo.demo.com;
    # disable any limits to avoid HTTP 413 for large image uploads
    client_max_body_size 0;
    # required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)
    chunked_transfer_encoding on;
    # Docker /v2 and /v1 (for search) requests
    location /v2 {
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto "https";
        proxy_pass http://nexus3/repository/docker-hosted-noauth/$request_uri;
    }
    location /v1 {
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto "https";
        proxy_pass http://nexus3/repository/docker-hosted-noauth/$request_uri;
    }

    location / {
        proxy_pass https://docker_hosts_noauth_ssl_upstream;
        proxy_set_header X-Forwarded-Proto "https";
        include include/proxy.conf; # 代理配置略
    }
}
```



#### 配置 Docker 仓库

在 Nexus 中创建以下 Docker 仓库:

1. **Docker (proxy)** - 端口 4002
   - 名称: `docker-proxy`
   - 远程地址: `https://registry-1.docker.io`
   - 用途: 缓存 Docker Hub 镜像

2. **Docker (hosted)** - 端口 4003
   - 名称: `docker-hosted-noauth`
   - HTTP 端口: 4003
   - 用途: 存储自定义构建的镜像 (如 `coredns:1.14.1-view`)

3. **Docker (group)** - 端口 4002
   - 名称: `docker-group`
   - 成员仓库: `docker-proxy`, `docker-hosted-noauth`
   - HTTP 端口: 4002
   - 用途: 统一访问入口

#### 配置 Docker 客户端

在构建节点上配置 Docker 信任 Nexus 仓库:

```bash
# /etc/docker/daemon.json
{
  "insecure-registries": [
    "your-docker-repo.demo.com:4012"
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

本 Role 提供了完整的 Makefile 模板用于镜像构建。

#### 1. 使用 Ansible 部署构建文件

**第一步：部署 Makefile 和构建文件**

在构建节点上执行 playbook,使用 `--tags build` 仅部署构建相关文件:

```bash
# 部署 Dockerfile, Makefile, plugin.cfg 到构建节点
ansible-playbook -i inventory playbook-coredns.yaml --tags build

# 如果只想部署到特定主机
ansible-playbook -i inventory playbook-coredns.yaml --tags build --limit coredns-builder
```

执行后会在目标节点生成:
- `/root/coredns/Dockerfile` - 镜像构建文件
- `/root/coredns/Makefile` - 构建和运维工具(根据变量自动生成)
- `/root/coredns/plugin.cfg` - 插件配置

**第二步：配置构建参数**

在 playbook 或 inventory 中配置镜像仓库和版本:

```yaml
# playbook-coredns.yaml 或 group_vars
vars:
  # 镜像仓库配置
  coredns_registry: "your-docker-repo.demo.com:4003"
  coredns_image_name: "coredns"

  # CoreDNS 和 Go 版本
  coredns_build_version: "1.14.1"
  coredns_go_version: "1.24"

  # Docker 网络配置
  coredns_network_subnet: "192.168.100.0/24"
  coredns_network_gateway: "192.168.100.1"

  # 系统 DNS 服务器
  coredns_system_dns_servers:
    - "192.168.0.100"
    - "192.168.0.101"
```

部署后 Makefile 会自动使用这些变量,无需手动修改。

#### 2. 构建镜像

SSH 登录到构建节点:

```bash
cd /root/coredns

# 查看所有可用命令和当前配置
make help
make info
```

输出示例:
```
============================================================
CoreDNS 构建配置信息
============================================================
Registry:         your-docker-repo.demo.com:4003
Image Name:       coredns
CoreDNS Version:  1.14.1
Go Version:       1.24
Docker Networks:  192.168.100.0/24
Image Tag:        your-docker-repo.demo.com:4003/coredns:1.14.1-view
Image Tag Latest: your-docker-repo.demo.com:4003/coredns:latest
============================================================
```

**开始构建:**

```bash
# 构建镜像
make build

# 无缓存构建 (当需要强制重新编译时)
make build-no-cache
```

#### 3. 推送到 Nexus

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

#### 4. 验证镜像

```bash
# 查看镜像信息
make info

# 从 Nexus 拉取验证
docker pull your-docker-repo.demo.com:4003/coredns:1.14.1-view

# 查看镜像详情
docker inspect your-docker-repo.demo.com:4003/coredns:1.14.1-view
```

#### 5. 更新构建配置

如果需要修改构建参数(如版本号、仓库地址):

```bash
# 方法 1: 修改 playbook 变量后重新部署 (推荐)
ansible-playbook -i inventory playbook-coredns.yaml --tags build

# 方法 2: 直接在目标节点编辑 Makefile (不推荐)
vi /root/coredns/Makefile
```

推荐使用方法 1,通过 Ansible 统一管理配置。

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

### AD 域控 DNS 配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `coredns_ad_dns_enabled` | `false` | 是否启用 AD 域控 DNS 集成 |
| `coredns_ad_dns_servers` | `[]` | AD 域控 DNS 服务器列表 (主/备) |
| `coredns_ad_domains` | `[]` | AD 域名列表 (需转发给 AD DNS) |
| `coredns_ad_reverse_zones` | `[]` | AD 反向解析区域 (PTR 记录) |
| `coredns_ad_dns_policy` | `sequential` | DNS 转发策略 (`sequential` 或 `random`) |

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

### 示例 5: AD 域控 DNS 集成 (企业最佳实践)

#### 场景说明

企业内网通常部署有 Windows Active Directory 域控服务器,CoreDNS 需要与 AD DNS 配合工作:
- **AD 域名** (如 `company.local`) 转发给 AD 域控 DNS
- **反向解析** (PTR 记录) 转发给 AD 域控
- **其他域名** 由 CoreDNS 正常解析
- **智能路由**: 内网客户端查询 AD 域名时使用域控 DNS,外网域名使用 CoreDNS

#### 基础配置

```yaml
# 启用 AD DNS 集成
coredns_ad_dns_enabled: true

# AD 域控 DNS 服务器 (主/备)
coredns_ad_dns_servers:
  - "192.168.1.10"   # 主域控 DNS
  - "192.168.1.11"   # 备域控 DNS

# AD 域名列表
coredns_ad_domains:
  - "company.local"              # AD 主域
  - "corp.company.local"         # AD 子域
  - "_msdcs.company.local"       # AD 服务定位
  - "_sites.company.local"       # AD 站点
  - "_tcp.company.local"         # AD SRV 记录
  - "_udp.company.local"         # AD SRV 记录

# 反向解析区域 (内网网段)
coredns_ad_reverse_zones:
  - "1.168.192.in-addr.arpa"     # 192.168.1.0/24
  - "2.168.192.in-addr.arpa"     # 192.168.2.0/24
  - "10.in-addr.arpa"            # 10.0.0.0/8 (可选)

# DNS 转发策略
coredns_ad_dns_policy: "sequential"  # 顺序尝试,主域控优先
```

#### 完整配置示例

```yaml
---
- hosts: coredns
  vars:
    # === CoreDNS 基础配置 ===
    coredns_upstream_servers:
      - 223.5.5.5      # 阿里 DNS
      - 114.114.114.114

    # === AD 域控 DNS 集成 ===
    coredns_ad_dns_enabled: true
    coredns_ad_dns_servers:
      - "192.168.1.10"
      - "192.168.1.11"

    coredns_ad_domains:
      - "company.local"
      - "corp.company.local"
      - "_msdcs.company.local"
      - "_sites.company.local"
      - "_tcp.company.local"
      - "_udp.company.local"

    coredns_ad_reverse_zones:
      - "1.168.192.in-addr.arpa"
      - "2.168.192.in-addr.arpa"

    # === 内网 Hosts 解析 ===
    coredns_hosts_entries:
      - 192.168.1.10 dc01.company.local dc01
      - 192.168.1.11 dc02.company.local dc02
      - 192.168.1.20 fileserver.company.local

    # === 特定域名转发 (GitHub, Google 等) ===
    coredns_forward_zones:
      - domains:
          - github.com
          - githubusercontent.com
        upstream: 223.5.5.5

  roles:
    - coredns
```

#### 验证 AD DNS 集成

部署后验证 AD DNS 功能:

```bash
# 1. 测试 AD 域名解析
nslookup dc01.company.local <coredns-ip>
nslookup company.local <coredns-ip>

# 2. 测试 AD SRV 记录
nslookup -type=SRV _ldap._tcp.company.local <coredns-ip>
nslookup -type=SRV _kerberos._tcp.company.local <coredns-ip>

# 3. 测试反向解析 (PTR)
nslookup 192.168.1.10 <coredns-ip>

# 4. 测试外网域名 (应该走上游 DNS)
nslookup www.baidu.com <coredns-ip>

# 5. 查看 CoreDNS 日志
docker logs -f coredns | grep "company.local"
```

#### 故障排查

**问题 1: AD 域名无法解析**

检查 AD 域控 DNS 是否可达:
```bash
# 在 CoreDNS 容器内测试
docker exec coredns nslookup company.local 192.168.1.10

# 检查网络连通性
docker exec coredns ping 192.168.1.10
```

**问题 2: SRV 记录查询失败**

确保配置了所有必要的 AD 子域:
```yaml
coredns_ad_domains:
  - "company.local"
  - "_msdcs.company.local"  # 必需
  - "_sites.company.local"  # 必需
  - "_tcp.company.local"    # 必需
  - "_udp.company.local"    # 必需
```

**问题 3: 反向解析不工作**

确认反向区域配置正确:
```bash
# 192.168.1.10 的反向区域是 1.168.192.in-addr.arpa
# 10.20.30.40 的反向区域是 30.20.10.in-addr.arpa
```

#### 高级配置: 结合视图实现智能解析

针对不同客户端返回不同的 DNS 服务器:

```yaml
# 内网客户端使用 AD DNS,外网客户端使用公网 DNS
coredns_view_enabled: true
coredns_view_rules:
  - domain: "company.local"
    views:
      # 内网客户端
      - name: "internal"
        cidr: "192.168.0.0/16"
        ttl: 60
        # 直接返回域控 IP
        ips:
          - "192.168.1.10"

      # VPN 客户端
      - name: "vpn"
        cidr: "10.8.0.0/24"
        ttl: 60
        ips:
          - "192.168.1.10"

    # 外网客户端不返回结果
    default:
      ttl: 60
      ips: []
```

#### 性能优化建议

1. **缓存时间**: AD 域名使用较短缓存 (30秒),避免域控更新延迟
2. **健康检查**: 启用域控健康检查,自动切换到备域控
3. **策略选择**:
   - `sequential`: 主域控优先,适合主备场景
   - `random`: 随机选择,适合负载均衡场景

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

