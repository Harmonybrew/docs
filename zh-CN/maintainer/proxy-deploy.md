# 代理部署指南

## 1 前言

Harmonybrew 使用华为云香港 Region 的服务器作为 CI 服务器。然而大量源码网站将华为云数据中心的 IP（尤其是非中国大陆的 IP）屏蔽掉了，用华为云的服务器去访问这些开源网站会得到 403 或 429 等返回值。

目前已发现的网站有这些：

```txt
archive.apache.org
codeberg.org
code.videolan.org
gitlab.archlinux.org
gitlab.freedesktop.org
gitlab.inria.fr
gitlab.torproject.org
git.madhouse-project.org
git.sr.ht
gitweb.gentoo.org
gkrellm.srcbox.net
linuxnetworks.de
math.lbl.gov
mbsd.evolvis.org
salsa.debian.org
siag.nu
src.fedoraproject.org
www.apache.org
www.cs.unm.edu
www.freedesktop.org
www.gaia-gis.it
```

为了解决这类问题，我们需要搭建一套代理系统，把通往这些网站的请求全部用 WARP（Cloudflare 免费 SOCKS5 代理）做一遍代理。

本文档用于记录技术方案和部署流程。文档末尾提供了一键部署脚本。

注意，本文档中的技术方案仅适用于境外服务器。中国大陆境内的服务器无法直连 WARP，本方案不适用。

## 2 方案简介

架构：Xray（SNI 嗅探 + 本地 DNS 解析）--> WARP（Cloudflare 免费 SOCKS5 代理）--> 海外源码站

```
宿主机 / 容器 / 内网其他设备
     │
     │ 请求 codeberg.org:443
     │ （客户端 /etc/hosts 指向 Xray 服务器 IP）
     ▼
┌─────────────────────┐
│   Xray dokodemo-door │  <-- 监听 0.0.0.0:80/443（或指定网卡 IP）
│     SNI 嗅探         │      提取 TLS ClientHello 中的域名
│     DNS 解析域名→IP  │      绕过系统 DNS，直接查 1.1.1.1
└─────────┬───────────┘
          │ 真实 IP: 217.197.84.140:443
          ▼
┌─────────────────────┐
│  WARP SOCKS5         │  <-- 127.0.0.1:40000
│  Cloudflare 出口     │      出口 IP: 104.28.215.x (HK)
└─────────┬───────────┘
          │
          ▼
     目标源码站
   codeberg.org / gitlab.freedesktop.org / code.videolan.org
```

原则：
- 仅 `codeberg.org` / `gitlab.freedesktop.org` / `code.videolan.org` 等特定域名走 WARP 代理，其余直连。
- 通过 `/etc/hosts` 注入 DNS 劫持，无需设置环境变量。

## 3 安装部署

从本文档附录获取 deploy-xray-warp.sh 脚本后，使用它进行部署

```bash
# 默认监听 0.0.0.0（所有网卡，内网设备共享同一实例）
chmod +x deploy-xray-warp.sh
./deploy-xray-warp.sh

# 仅监听指定内网 IP
./deploy-xray-warp.sh --listen 10.0.0.5

# 仅监听 Docker 桥接，仅供本机容器使用
./deploy-xray-warp.sh --listen 172.17.0.1

# 查看帮助
./deploy-xray-warp.sh --help
```

脚本支持幂等重入，可在已部署或部署中断的机器上重复执行。它会自动完成：检测系统 -> 部署 WARP -> 注册并连接 WARP -> 下载安装 Xray -> 写入 systemd 服务 -> 生成 `config.json`（按 `--listen` 注入监听地址）-> 启动 -> Docker 容器验证下载。

## 4 验证功能

通过启动临时容器验证代理是否工作

```bash
docker run --rm -i \
  --add-host codeberg.org:<HOST_IP> \
  --add-host gitlab.freedesktop.org:<HOST_IP> \
  --add-host code.videolan.org:<HOST_IP> \
  --add-host salsa.debian.org:<HOST_IP> \
  alpine:3.20 sh -c '
    apk add --no-cache curl >/dev/null 2>&1
    for url in \
      "https://codeberg.org/gpanders/ijq/archive/v1.3.0.tar.gz" \
      "https://gitlab.freedesktop.org/xorg/lib/libxp/-/archive/libXp-1.0.4/libxp-libXp-1.0.4.tar.bz2" \
      "https://code.videolan.org/rist/librist/-/archive/v0.2.17/librist-v0.2.17.tar.gz"; do
      f=$(echo "$url" | sed "s|.*/||" | cut -d? -f1)
      echo -n "$f ... "
      curl -sL --max-time 60 -o "/tmp/$f" "$url" && echo "OK ($(stat -c%s /tmp/$f) bytes)" || echo "FAIL"
    done
  '
# 期望: 全部 OK，size > 1000
```

Xray 日志确认

```bash
# 查看详细调试日志（先设为 debug 级别）
sed -i 's/"loglevel": "warning"/"loglevel": "debug"/' /usr/local/etc/xray/config.json
systemctl restart xray

# 请求一次目标站点后查看日志
curl -s --max-time 15 https://codeberg.org > /dev/null 2>&1
tail -20 /var/log/xray/error.log
# 应出现关键信息:
#   app/dns: returning 1 IP(s) for domain codeberg.org -> [217.197.84.140]
#   proxy/freedom: dialing to tcp:217.197.84.140:443
#   app/proxyman/outbound: proxying to socks-proxy for dest tcp:217.197.84.140:443

# 恢复 warning 级别
sed -i 's/"loglevel": "debug"/"loglevel": "warning"/' /usr/local/etc/xray/config.json
systemctl restart xray
```


## 5 代理使用方法

在 CI 环境中（服务器或 Docker 容器，取决于你的 CI 跑在哪里）设置 `/etc/hosts`，将需要被代理的站点解析到 Xray 服务器

```bash
cat >> /etc/hosts << 'EOF'
<HOST_IP> archive.apache.org
<HOST_IP> codeberg.org
<HOST_IP> code.videolan.org
<HOST_IP> gitlab.archlinux.org
<HOST_IP> gitlab.freedesktop.org
<HOST_IP> gitlab.inria.fr
<HOST_IP> gitlab.torproject.org
<HOST_IP> git.madhouse-project.org
<HOST_IP> git.sr.ht
<HOST_IP> gitweb.gentoo.org
<HOST_IP> gkrellm.srcbox.net
<HOST_IP> linuxnetworks.de
<HOST_IP> math.lbl.gov
<HOST_IP> mbsd.evolvis.org
<HOST_IP> salsa.debian.org
<HOST_IP> siag.nu
<HOST_IP> src.fedoraproject.org
<HOST_IP> www.apache.org
<HOST_IP> www.cs.unm.edu
<HOST_IP> www.freedesktop.org
<HOST_IP> www.gaia-gis.it
EOF
```


## 6 故障排查

### 6.1 常见问题

| 症状 | 原因 | 解决 |
|------|------|------|
| `SSL_ERROR_SYSCALL` | Xray 没有正确解析域名为 IP，传给 WARP 时还是域名 | 检查 Xray 日志是否有 `domainStrategy: "UseIPv4"` 和 `dns.hosts` 配置 |
| WARP SOCKS5 拒绝连接 (错误码 5) | WARP 内部 DNS 解析失败，拒绝域名连接 | 确保 Xray 的 `proxySettings` 链式代理配置正确，freedom 先解析再转发 |
| 容器内连接超时 | `/etc/hosts` 指向的 IP 不是容器能访问的宿主机 IP | 确认 `ip addr show docker0` 与宿主机网卡 IP，进行修正 |
| 内网其他设备连接超时 | Xray 未监听对应网卡 IP，或防火墙拦截 | 检查 `ss -tlnp \| grep xray` 监听地址；确认防火墙放行可信网段到 80/443 |
| 监听 `0.0.0.0` 但外网仍能访问 | 防火墙未配置 | 仅放行可信内网网段到 80/443，拒绝公网来源 |
| 宿主机连不上目标站点 | `/etc/hosts` 未配置或 Xray 未运行 | 检查 `ss -tlnp \| grep xray` 和 `/etc/hosts` 条目 |
| 非目标域名也走代理 | DNS 被错误劫持 | 检查 Xray 路由规则，确认只匹配 `inboundTag` |
| 下载到 HTML 而非压缩包 | 目标站点返回验证页（如 Anubis challenge） | 通过代理访问即可解决，WARP 出口 IP 可绕过反爬 |
| 下载文件大小为 0 | 连接被拒绝或超时 | 增加 `--max-time 60`，检查网络连通性 |

### 6.2 查看 Xray 日志

```bash
# 错误日志
tail -50 /var/log/xray/error.log

# 访问日志（显示连接来源和出站）
tail -20 /var/log/xray/access.log

# 实时跟踪
journalctl -u xray -f --no-pager
```

### 6.3 验证 DNS 解析

```bash
# 检查目标域名在 Xray 内部的解析结果
# 在日志中搜索: app/dns: returning XX IP(s) for domain XXXX -> [x.x.x.x]

# 手动验证域名真实 IP（绕开 /etc/hosts）
dig @1.1.1.1 +short codeberg.org
# 期望: 217.197.84.140（不是 127.1.2.3）
```

### 6.4 WARP 连通性验证

```bash
# 验证 WARP 自身工作正常
curl -s --max-time 15 -x socks5h://127.0.0.1:40000 https://cloudflare.com/cdn-cgi/trace
# warp=on 表示 WARP 出口

# 验证 WARP 能连目标（用 IP，不要用域名）
curl -s --max-time 15 -x socks5h://127.0.0.1:40000 https://217.197.84.140 -o /dev/null -w "%{http_code}"
```

## 7 卸载清理

本文档附录提供 `uninstall-xray-warp.sh` 一键卸载脚本，可用于清理环境

```bash
chmod +x uninstall-xray-warp.sh

# 默认卸载：停止并移除 Xray 和 WARP，保留 /var/log/xray 日志便于事后排查
./uninstall-xray-warp.sh

# 彻底清理：连日志、Xray 配置、WARP 用户数据一起删除
./uninstall-xray-warp.sh --purge

# 仅卸载 Xray，保留 WARP（WARP 还可被其他用途继续使用）
./uninstall-xray-warp.sh --keep-warp

# 仅卸载 WARP，保留 Xray（Xray 配置仍在，但出站会失效，需重新配置出站代理）
./uninstall-xray-warp.sh --keep-xray

# 查看帮助
./uninstall-xray-warp.sh --help
```

卸载范围

| 资源 | 默认卸载 | `--purge` | `--keep-warp` | `--keep-xray` |
|------|:--------:|:---------:|:------------:|:-------------:|
| `xray.service` / `xray@.service` | ✓ | ✓ | ✓ | ✗ |
| `/usr/local/bin/xray` 二进制 | ✓ | ✓ | ✓ | ✗ |
| `/usr/local/etc/xray` 配置 | ✓ | ✓ | ✓ | ✗ |
| `/usr/local/share/xray` geoip/geosite | ✓ | ✓ | ✓ | ✗ |
| `/var/log/xray` 日志 | 保留 | ✓ | 保留 | 保留 |
| `warp-svc.service` | ✓ | ✓ | ✗ | ✓ |
| `warp-cli` 注册注销 | ✓ | ✓ | ✗ | ✓ |
| `cloudflare-warp` 软件包 | ✓ | ✓ | ✗ | ✓ |
| Cloudflare apt 源与密钥 | ✓ | ✓ | ✗ | ✓ |
| `/var/lib/cloudflare-warp` 等用户数据 | 保留 | ✓ | 保留 | 保留 |


## 8 附录：脚本内容

deploy-xray-warp.sh

```bash
#!/usr/bin/env bash
# ============================================================================
# deploy-xray-warp.sh  —  Xray + WARP 本地部署脚本
# 用法：在目标服务器上直接执行
#       chmod +x deploy-xray-warp.sh
#       ./deploy-xray-warp.sh                      # 默认监听 0.0.0.0（所有网卡）
#       ./deploy-xray-warp.sh --listen 0.0.0.0     # 显式指定监听所有网卡
#       ./deploy-xray-warp.sh --listen 10.0.0.5    # 仅监听指定内网 IP
#       ./deploy-xray-warp.sh --listen 172.17.0.1  # 仅监听 Docker 桥接
#
#       本脚本支持幂等重入，可在已部署或部署中断的机器上重复执行，无副作用。
#
# 监听地址说明：
#       * 默认 0.0.0.0：Xray 监听所有网卡，内网任意设备都可通过宿主机 IP
#         访问同一个 Xray 实例，无需每台机器单独部署。
#       * 指定具体 IP：仅监听该网卡，便于做最小化暴露面（配合防火墙更安全）。
#       * 监听 0.0.0.0 时请确认宿主机防火墙只放行可信网段，避免暴露到公网。
# ============================================================================
set -euo pipefail

# ---- 配置（如需升级 Xray 版本，改这里）----
XRAY_VERSION="v26.3.27"

# ---- 解析命令行参数 ----
LISTEN_ADDR="0.0.0.0"
while [ $# -gt 0 ]; do
  case "$1" in
    --listen)
      [ $# -ge 2 ] || { echo "[ERROR] --listen 需要一个参数" >&2; exit 1; }
      LISTEN_ADDR="$2"
      shift 2
      ;;
    --listen=*)
      LISTEN_ADDR="${1#--listen=}"
      shift
      ;;
    -h|--help)
      sed -n '2,20p' "$0"
      exit 0
      ;;
    *)
      echo "[ERROR] 未知参数: $1" >&2
      exit 1
      ;;
  esac
done

# ---- 辅助函数 ----
die() { echo "[ERROR] $*" >&2; exit 1; }
info() { echo "[INFO] $*"; }
ok()   { echo "[OK]   $*"; }

# ---- 0. 检查 root ----
[ "$EUID" -eq 0 ] || die "请以 root 身份运行"

# ---- 1. 检查系统 ----
info "========== Xray + WARP 本地部署 =========="
info "Xray 版本: $XRAY_VERSION"
info "监听地址: $LISTEN_ADDR"
echo ""

info "检查系统..."
OS=$(grep -oP '^ID=\K.*' /etc/os-release 2>/dev/null || echo "unknown")
ARCH=$(uname -m)
DOCKER_VER=$(docker --version 2>/dev/null || echo "not-installed")
echo "  OS: $OS"
echo "  ARCH: $ARCH"
echo "  DOCKER: $DOCKER_VER"

case "$ARCH" in
  x86_64)  ZIP_ARCH="64" ;;
  aarch64) ZIP_ARCH="arm64-v8a" ;;
  armv7l)  ZIP_ARCH="arm32-v7a" ;;
  *) die "不支持的架构: $ARCH" ;;
esac

echo "$DOCKER_VER" | grep -q "not-installed" && die "请先安装 Docker"
ok "系统检查通过"
echo ""

# ---- 2. 检测 Docker 桥接 IP（容器访问宿主机代理用）----
info "检测 Docker 桥接 IP..."
DOCKER_BRIDGE_IP=$(ip -4 addr show docker0 2>/dev/null | grep -oP 'inet \K[\d.]+' | head -1)
[ -z "$DOCKER_BRIDGE_IP" ] && DOCKER_BRIDGE_IP="172.17.0.1"
info "Docker 桥接 IP: $DOCKER_BRIDGE_IP"
info "Xray 监听地址: $LISTEN_ADDR"
# 容器通过 --add-host domain:<宿主机可达IP> 把流量导向 Xray。
# 当 Xray 监听 0.0.0.0 或具体网卡 IP 时，--add-host 应指向该网卡 IP（推荐用内网 IP）；
# 兼容场景下也可继续指向 docker0 网关 IP。
HOST_HINT="$LISTEN_ADDR"
[ "$HOST_HINT" = "0.0.0.0" ] && HOST_HINT="$DOCKER_BRIDGE_IP"
info "容器 --add-host 建议指向: $HOST_HINT"
echo ""

# ---- 3. 部署 WARP 客户端 ----
info "===== 1/4: 部署 WARP 客户端 ====="
if systemctl is-active warp-svc >/dev/null 2>&1; then
  ok "WARP 已安装且运行中，跳过安装"
else
  if command -v warp-cli >/dev/null 2>&1; then
    echo "WARP 已安装，启动服务..."
    systemctl start warp-svc
  else
    echo "安装 WARP 客户端..."
    curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | gpg --yes --dearmor -o /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg 2>/dev/null
    CODENAME=$(grep -oP 'VERSION_CODENAME=\K.*' /etc/os-release)
    echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ $CODENAME main" > /etc/apt/sources.list.d/cloudflare-client.list
    apt-get update -qq
    apt-get install -y -qq cloudflare-warp
  fi
  ok "WARP 安装完成"
fi
echo ""

# ---- 4. 注册并配置 WARP ----
info "===== 2/4: 配置 WARP 客户端 ====="
for i in $(seq 1 10); do
  if warp-cli --accept-tos status >/dev/null 2>&1; then break; fi
  sleep 1
done

if warp-cli --accept-tos status 2>&1 | grep -q "Connected"; then
  ok "WARP 已连接，跳过注册"
else
  echo "注册 WARP..."
  warp-cli --accept-tos registration new 2>/dev/null || echo "  (可能已注册，忽略)"
fi

warp-cli --accept-tos mode proxy 2>/dev/null || true
warp-cli --accept-tos proxy port 40000 2>/dev/null || true
warp-cli --accept-tos connect 2>/dev/null || true

for i in $(seq 1 15); do
  if warp-cli --accept-tos status 2>&1 | grep -q "Connected"; then
    ok "WARP 已连接"
    break
  fi
  sleep 2
done
echo ""

# ---- 5. 部署 Xray ----
info "===== 3/4: 部署 Xray ${XRAY_VERSION} ====="
XRAY_BIN="/usr/local/bin/xray"
XRAY_CURRENT=""
[ -f "$XRAY_BIN" ] && XRAY_CURRENT=$("$XRAY_BIN" version 2>/dev/null | awk 'NR==1{print $2}') || true

if [ "$XRAY_CURRENT" = "$XRAY_VERSION" ]; then
  ok "Xray $XRAY_VERSION 已安装，跳过"
else
  echo "下载 Xray ${XRAY_VERSION}..."
  ZIP_FILE="/tmp/Xray-linux-${ZIP_ARCH}.zip"
  curl -fL --progress-bar -o "$ZIP_FILE" "https://github.com/XTLS/Xray-core/releases/download/${XRAY_VERSION}/Xray-linux-${ZIP_ARCH}.zip"
  echo ""
  TMP_DIR="/tmp/xray-install"
  mkdir -p "$TMP_DIR"
  unzip -o -q "$ZIP_FILE" -d "$TMP_DIR"
  install -m 755 "${TMP_DIR}/xray" "$XRAY_BIN"
  mkdir -p /usr/local/etc/xray /usr/local/share/xray /var/log/xray
  install -m 644 "${TMP_DIR}"/geoip.dat /usr/local/share/xray/ 2>/dev/null || true
  install -m 644 "${TMP_DIR}"/geosite.dat /usr/local/share/xray/ 2>/dev/null || true
  rm -rf "$TMP_DIR" "$ZIP_FILE"

  if [ ! -f /etc/systemd/system/xray.service ]; then
    echo "创建 systemd 服务..."
    cat > /etc/systemd/system/xray.service << 'EOF'
[Unit]
Description=Xray Service
Documentation=https://github.com/xtls
After=network.target nss-lookup.target

[Service]
User=root
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ExecStart=/usr/local/bin/xray run -config /usr/local/etc/xray/config.json
Restart=on-failure
RestartPreventExitStatus=23
LimitNPROC=10000
LimitNOFILE=1000000

[Install]
WantedBy=multi-user.target
EOF
    systemctl daemon-reload
  fi
  ok "Xray 部署完成"
fi
echo ""

# ---- 6. 写入 Xray 配置 ----
info "===== 4/4: 写入 Xray 配置 ====="

cat > /usr/local/etc/xray/config.json << EOF
{
  "log": {
    "loglevel": "warning",
    "access": "/var/log/xray/access.log",
    "error": "/var/log/xray/error.log"
  },
  "dns": {
    "hosts": {
      "archive.apache.org": ["65.108.204.189"],
      "codeberg.org": ["217.197.84.140"],
      "code.videolan.org": ["213.36.253.9"],
      "gitlab.archlinux.org": ["213.133.111.15"],
      "gitlab.freedesktop.org": ["151.101.195.52"],
      "gitlab.inria.fr": ["128.93.193.18"],
      "gitlab.torproject.org": ["204.8.99.149"],
      "git.madhouse-project.org": ["95.216.196.166"],
      "git.sr.ht": ["46.23.81.155"],
      "gitweb.gentoo.org": ["151.101.1.91"],
      "gkrellm.srcbox.net": ["152.53.246.159"],
      "linuxnetworks.de": ["85.13.131.27"],
      "math.lbl.gov": ["104.18.38.38"],
      "mbsd.evolvis.org": ["217.144.135.10"],
      "salsa.debian.org": ["209.87.16.82"],
      "siag.nu": ["93.188.2.52"],
      "src.fedoraproject.org": ["38.145.32.20"],
      "www.apache.org": ["151.101.2.132"],
      "www.cs.unm.edu": ["64.106.20.76"],
      "www.freedesktop.org": ["131.252.210.176"],
      "www.gaia-gis.it": ["178.104.132.174"]
    },
    "servers": [
      { "address": "1.1.1.1", "port": 53, "skipFallback": true },
      { "address": "8.8.8.8", "port": 53, "skipFallback": true }
    ],
    "queryStrategy": "UseIPv4"
  },
  "inbounds": [
    {
      "tag": "tls-in",
      "listen": "${LISTEN_ADDR}",
      "port": 443,
      "protocol": "dokodemo-door",
      "settings": { "network": "tcp", "followRedirect": false },
      "sniffing": { "enabled": true, "destOverride": ["tls"], "metadataOnly": false }
    },
    {
      "tag": "http-in",
      "listen": "${LISTEN_ADDR}",
      "port": 80,
      "protocol": "dokodemo-door",
      "settings": { "network": "tcp", "followRedirect": false },
      "sniffing": { "enabled": true, "destOverride": ["http"], "metadataOnly": false }
    }
  ],
  "outbounds": [
    {
      "tag": "warp-out",
      "protocol": "freedom",
      "settings": { "domainStrategy": "UseIPv4" },
      "proxySettings": { "tag": "socks-proxy" }
    },
    {
      "tag": "socks-proxy",
      "protocol": "socks",
      "settings": {
        "servers": [{ "address": "127.0.0.1", "port": 40000 }]
      }
    }
  ],
  "routing": {
    "rules": [
      {
        "type": "field",
        "inboundTag": ["tls-in", "http-in"],
        "outboundTag": "warp-out"
      }
    ]
  }
}
EOF

xray -test -config /usr/local/etc/xray/config.json 2>&1 | tail -3
ok "Xray 配置写入并验证"
echo ""

# ---- 7. 启动 Xray ----
info "启动 Xray 服务..."
systemctl enable xray 2>/dev/null || true
systemctl restart xray
sleep 2

if systemctl is-active xray >/dev/null 2>&1; then
  ok "Xray 运行中"
  ss -tlnp | grep xray
else
  die "Xray 启动失败，日志: journalctl -u xray --no-pager -n 20"
fi
echo ""

# ---- 8. 验证 ----
info "========== 最终验证 =========="

PASS=0
FAIL=0

echo "--- 代理域名下载测试（通过 Docker 容器，--add-host 指向 $HOST_HINT）---"
docker run --rm -i \
  --add-host archive.apache.org:"${HOST_HINT}" \
  --add-host codeberg.org:"${HOST_HINT}" \
  --add-host code.videolan.org:"${HOST_HINT}" \
  --add-host gitlab.archlinux.org:"${HOST_HINT}" \
  --add-host gitlab.freedesktop.org:"${HOST_HINT}" \
  --add-host salsa.debian.org:"${HOST_HINT}" \
  alpine:3.20 sh -c '
    apk add --no-cache curl >/dev/null 2>&1
    ok=0; fail=0
    check() {
      local url="$1" name="$2"
      local size
      size=$(curl -sL --max-time 60 -o /tmp/f "$url" -w "%{size_download}" 2>/dev/null) || { echo "  ✗ $name: 下载失败"; fail=$((fail+1)); return; }
      if [ "$size" -gt 1000 ]; then
        echo "  ✓ $name: ${size} bytes"
        ok=$((ok+1))
      else
        echo "  ✗ $name: 文件太小 (${size})"
        fail=$((fail+1))
      fi
    }
    check "https://codeberg.org/gpanders/ijq/archive/v1.3.0.tar.gz" "codeberg.org"
    check "https://gitlab.freedesktop.org/xorg/lib/libxp/-/archive/libXp-1.0.4/libxp-libXp-1.0.4.tar.bz2" "gitlab.freedesktop.org"
    check "https://code.videolan.org/rist/librist/-/archive/v0.2.17/librist-v0.2.17.tar.gz" "code.videolan.org"
    check "https://salsa.debian.org/autotools-files/automake/repository/archive.tar.gz?ref=master" "salsa.debian.org"
    echo ""
    echo "结果: ${ok} 通过, ${fail} 失败"
    [ "$fail" -gt 0 ] && exit 1 || exit 0
  ' 2>&1 | grep -v "Unable to find image\|Pulling\|Digest\|Status\|Download" || { echo "  容器测试失败"; FAIL=$((FAIL+1)); }

echo "--- 直连域名测试 ---"
if curl -sI --max-time 15 https://github.com 2>&1 | grep -q "HTTP/2 200\|HTTP/2 301\|HTTP/1.1 200"; then
  echo "  ✓ github.com (直连)"
  PASS=$((PASS+1))
else
  echo "  ✗ github.com (直连)"
  FAIL=$((FAIL+1))
fi

echo ""
echo "结果: ${PASS} 通过, ${FAIL} 失败"
[ "$FAIL" -gt 0 ] && die "部分验证失败" || ok "部署成功！"
```

uninstall-xray-warp.sh

```bash
#!/usr/bin/env bash
# ============================================================================
# uninstall-xray-warp.sh  —  Xray + WARP 一键卸载脚本
# 用法：在目标服务器上直接执行
#       chmod +x uninstall-xray-warp.sh
#       ./uninstall-xray-warp.sh                # 卸载 Xray 和 WARP（保留日志）
#       ./uninstall-xray-warp.sh --purge        # 同时删除日志和配置（彻底清理）
#       ./uninstall-xray-warp.sh --keep-warp    # 仅卸载 Xray，保留 WARP
#       ./uninstall-xray-warp.sh --keep-xray    # 仅卸载 WARP，保留 Xray
#
#       本脚本幂等：任何资源不存在时跳过，不报错。
# ============================================================================
set -euo pipefail

# ---- 解析命令行参数 ----
PURGE=0
KEEP_WARP=0
KEEP_XRAY=0
while [ $# -gt 0 ]; do
  case "$1" in
    --purge)       PURGE=1; shift ;;
    --keep-warp)   KEEP_WARP=1; shift ;;
    --keep-xray)   KEEP_XRAY=1; shift ;;
    -h|--help)
      sed -n '2,14p' "$0"
      exit 0
      ;;
    *)
      echo "[ERROR] 未知参数: $1" >&2
      exit 1
      ;;
  esac
done

# ---- 辅助函数 ----
die()  { echo "[ERROR] $*" >&2; exit 1; }
info() { echo "[INFO] $*"; }
ok()   { echo "[OK]   $*"; }
skip() { echo "[SKIP] $*"; }

# ---- 0. 检查 root ----
[ "$EUID" -eq 0 ] || die "请以 root 身份运行"

info "========== Xray + WARP 卸载 =========="
info "purge=$PURGE keep_warp=$KEEP_WARP keep_xray=$KEEP_XRAY"
echo ""

# ============================================================================
# 卸载 Xray
# ============================================================================
if [ "$KEEP_XRAY" -eq 0 ]; then
  info "===== 1/2: 卸载 Xray ====="

  # 停止并禁用 systemd 服务。
  # 无条件尝试 stop/disable：对已加载的服务会真正停止，对不存在的服务
  # systemd 安静失败（被 || true 吞掉）。这样无需精确判断 unit 是否注册，
  # 避免 list-unit-files / list-units 在 unit 文件被删/状态为 not-found 时漏判。
  if systemctl is-active --quiet xray 2>/dev/null || systemctl list-units --full --all 2>/dev/null | grep -q 'xray\.service'; then
    info "停止 xray.service..."
  fi
  systemctl stop xray 2>/dev/null || true
  systemctl disable xray 2>/dev/null || true
  if systemctl is-active --quiet xray 2>/dev/null || systemctl list-units --full --all 2>/dev/null | grep -q 'xray\.service'; then
    ok "xray.service 已停止并禁用"
  else
    skip "xray.service 未加载"
  fi
  # 兜底：万一 unit 已从磁盘删除但进程仍在跑（孤儿），直接杀掉
  if pgrep -x xray >/dev/null 2>&1; then
    info "发现残留 xray 进程，发送 SIGTERM..."
    pkill -x xray 2>/dev/null || true
    sleep 1
    pkill -9 -x xray 2>/dev/null || true
    ok "残留 xray 进程已清理"
  fi

  # 删除 systemd 单元文件（本仓库部署脚本只写 xray.service；
  # 也兼容官方脚本遗留的 xray@.service）
  for unit in xray.service xray@.service; do
    if [ -f "/etc/systemd/system/${unit}" ]; then
      rm -f "/etc/systemd/system/${unit}"
      ok "已删除 /etc/systemd/system/${unit}"
    else
      skip "/etc/systemd/system/${unit} 不存在"
    fi
  done
  systemctl daemon-reload 2>/dev/null || true

  # 删除二进制
  if [ -f /usr/local/bin/xray ]; then
    rm -f /usr/local/bin/xray
    ok "已删除 /usr/local/bin/xray"
  else
    skip "/usr/local/bin/xray 不存在"
  fi

  # 删除配置 / 数据 / 日志
  # --purge：连配置和日志一起删；否则保留日志便于事后排查
  if [ "$PURGE" -eq 1 ]; then
    for path in /usr/local/etc/xray /usr/local/share/xray /var/log/xray; do
      if [ -e "$path" ]; then
        rm -rf "$path"
        ok "已删除 $path"
      else
        skip "$path 不存在"
      fi
    done
  else
    if [ -e /usr/local/etc/xray ]; then
      rm -rf /usr/local/etc/xray
      ok "已删除 /usr/local/etc/xray（配置）"
    else
      skip "/usr/local/etc/xray 不存在"
    fi
    if [ -e /usr/local/share/xray ]; then
      rm -rf /usr/local/share/xray
      ok "已删除 /usr/local/share/xray（geoip/geosite 数据）"
    else
      skip "/usr/local/share/xray 不存在"
    fi
    # 日志默认保留（除非 --purge）
    if [ -d /var/log/xray ]; then
      info "保留 /var/log/xray 日志目录（如需删除请加 --purge）"
    fi
  fi

  # 清理下载残留
  rm -rf /tmp/xray-install /tmp/Xray-linux-*.zip 2>/dev/null || true

  ok "Xray 卸载完成"
else
  info "===== 1/2: 跳过 Xray（--keep-xray）====="
fi
echo ""

# ============================================================================
# 卸载 WARP
# ============================================================================
if [ "$KEEP_WARP" -eq 0 ]; then
  info "===== 2/2: 卸载 WARP 客户端 ====="

  # 断开并注销
  if command -v warp-cli >/dev/null 2>&1; then
    info "断开 WARP..."
    warp-cli disconnect 2>/dev/null || true
    info "注销 WARP 注册..."
    warp-cli registration delete 2>/dev/null || true
    ok "WARP 已断开并注销"
  else
    skip "warp-cli 未安装"
  fi

  # 停止并禁用 systemd 服务（无条件尝试 stop/disable，避免漏判）
  if systemctl is-active --quiet warp-svc 2>/dev/null || systemctl list-units --full --all 2>/dev/null | grep -q 'warp-svc\.service'; then
    info "停止 warp-svc.service..."
  fi
  systemctl stop warp-svc 2>/dev/null || true
  systemctl disable warp-svc 2>/dev/null || true
  if systemctl is-active --quiet warp-svc 2>/dev/null || systemctl list-units --full --all 2>/dev/null | grep -q 'warp-svc\.service'; then
    ok "warp-svc.service 已停止并禁用"
  else
    skip "warp-svc.service 未加载"
  fi

  # 卸载 deb 包
  if dpkg -l cloudflare-warp >/dev/null 2>&1; then
    info "卸载 cloudflare-warp 软件包..."
    DEBIAN_FRONTEND=noninteractive apt-get purge -y -qq cloudflare-warp
    ok "cloudflare-warp 软件包已卸载"
  else
    skip "cloudflare-warp 软件包未安装"
  fi

  # 清理 apt 源与密钥
  for f in \
    /etc/apt/sources.list.d/cloudflare-client.list \
    /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg; do
    if [ -f "$f" ]; then
      rm -f "$f"
      ok "已删除 $f"
    else
      skip "$f 不存在"
    fi
  done
  apt-get update -qq 2>/dev/null || true

  # 残留用户数据（--purge 时彻底清理）
  if [ "$PURGE" -eq 1 ]; then
    for path in /var/lib/cloudflare-warp /var/log/cloudflare-warp /etc/cloudflare-warp; do
      if [ -e "$path" ]; then
        rm -rf "$path"
        ok "已删除 $path"
      else
        skip "$path 不存在"
      fi
    done
  else
    if [ -d /var/lib/cloudflare-warp ]; then
      info "保留 /var/lib/cloudflare-warp（如需删除请加 --purge）"
    fi
  fi

  ok "WARP 卸载完成"
else
  info "===== 2/2: 跳过 WARP（--keep-warp）====="
fi
echo ""

# ============================================================================
# 最终核对
# ============================================================================
info "========== 卸载核对 =========="
# 清 bash command hash 缓存，避免 purge 后 command -v 仍命中旧路径造成误报
hash -r 2>/dev/null || true

if [ "$KEEP_XRAY" -eq 0 ]; then
  if [ -f /usr/local/bin/xray ]; then
    echo "  ✗ xray 二进制仍存在"
  else
    echo "  ✓ xray 二进制已移除"
  fi
  if pgrep -x xray >/dev/null 2>&1; then
    echo "  ✗ xray 进程仍在运行"
  else
    echo "  ✓ xray 进程已停止"
  fi
fi

if [ "$KEEP_WARP" -eq 0 ]; then
  # 用文件存在性判断，避免 bash hash 缓存导致 command -v 误报
  if [ -x /usr/bin/warp-cli ] || [ -x /opt/cloudflare-warp/warp-cli ]; then
    echo "  ✗ warp-cli 仍存在"
  else
    echo "  ✓ warp-cli 已移除"
  fi
  if pgrep -x warp-svc >/dev/null 2>&1; then
    echo "  ✗ warp-svc 进程仍在运行"
  else
    echo "  ✓ warp-svc 进程已停止"
  fi
fi

echo ""
ok "卸载流程结束。建议确认以上全部为 ✓。"
```

xray-dns-hosts-updater.py

```python
#!/usr/bin/env python3
"""
xray-dns-hosts-updater.py

自动解析 xray config.json 中 dns.hosts 里的域名，
将最新 IPv4 地址写回配置文件，并在配置变更时重启 xray 服务。

用法:
    python3 /root/xray-dns-hosts-updater.py           # 使用默认配置路径
    python3 /root/xray-dns-hosts-updater.py /path/to/config.json

建议配合 cron 每天定时执行。
"""

import json
import socket
import subprocess
import sys
import os
import shutil
import tempfile
import datetime

# ─── 配置 ───────────────────────────────────────────────────────────
DEFAULT_CONFIG = "/usr/local/etc/xray/config.json"
BACKUP_SUFFIX  = ".bak"
LOG_TIME_FMT   = "%Y-%m-%d %H:%M:%S"

# DNS 解析失败时的最大重试次数
MAX_RETRIES = 2
# ────────────────────────────────────────────────────────────────────


def log(msg: str):
    """带时间戳的日志输出"""
    ts = datetime.datetime.now().strftime(LOG_TIME_FMT)
    print(f"[{ts}] {msg}")


def resolve_ipv4(domain: str) -> str | None:
    """
    解析域名的 IPv4 地址。
    优先使用 getent ahosts（走系统 DNS），失败时回退到 socket.getaddrinfo。
    返回第一个有效 IPv4 地址，解析失败返回 None。
    """
    # 方法1: getent ahosts（使用系统 nsswitch 配置的 DNS）
    for attempt in range(1, MAX_RETRIES + 1):
        try:
            result = subprocess.run(
                ["getent", "ahosts", domain],
                capture_output=True, text=True, timeout=10,
            )
            if result.returncode == 0:
                for line in result.stdout.strip().splitlines():
                    parts = line.split()
                    if len(parts) >= 2 and parts[1] == "STREAM":
                        ip = parts[0]
                        # 简单校验是 IPv4
                        if "." in ip and not ip.startswith("127."):
                            return ip
        except (subprocess.TimeoutExpired, OSError):
            pass

        # 方法2: socket.getaddrinfo 回退
        try:
            infos = socket.getaddrinfo(domain, None, socket.AF_INET, socket.SOCK_STREAM)
            for info in infos:
                ip = info[4][0]
                if not ip.startswith("127."):
                    return ip
        except (socket.gaierror, OSError):
            pass

        if attempt < MAX_RETRIES:
            log(f"  {domain}: 解析失败，第 {attempt} 次重试...")

    return None


def restart_xray():
    """重启 xray 服务"""
    try:
        result = subprocess.run(
            ["systemctl", "restart", "xray"],
            capture_output=True, text=True, timeout=30,
        )
        if result.returncode == 0:
            log("xray 服务已重启")
        else:
            log(f"xray 重启失败: {result.stderr.strip()}")
    except Exception as e:
        log(f"xray 重启异常: {e}")


def update_config(config_path: str) -> bool:
    """
    读取配置，解析所有 dns.hosts 域名，更新 IP，写回文件。
    返回 True 表示配置有变更，False 表示无变更。
    """
    # 读取原始配置（保留键顺序需要 Python 3.7+ dict 有序特性）
    with open(config_path, "r", encoding="utf-8") as f:
        config = json.load(f)

    hosts = config.get("dns", {}).get("hosts", {})
    if not hosts:
        log("配置中未找到 dns.hosts，跳过")
        return False

    changed = False

    for domain, old_ips in list(hosts.items()):
        # 跳过非列表值（以防万一）
        if not isinstance(old_ips, list):
            log(f"  {domain}: 值类型异常 ({type(old_ips).__name__})，跳过")
            continue

        new_ip = resolve_ipv4(domain)

        if new_ip is None:
            log(f"  {domain}: 解析失败，保留原 IP {old_ips}")
            continue

        old_ip = old_ips[0] if old_ips else ""

        if new_ip != old_ip:
            log(f"  {domain}: {old_ip} -> {new_ip}")
            hosts[domain] = [new_ip]
            changed = True
        else:
            log(f"  {domain}: {new_ip} (未变化)")

    if not changed:
        log("所有 IP 均未变化，无需更新")
        return False

    # 备份原文件
    backup_path = config_path + BACKUP_SUFFIX
    shutil.copy2(config_path, backup_path)
    log(f"已备份原配置到 {backup_path}")

    # 原子写入：先写临时文件，再 rename 覆盖
    dir_name = os.path.dirname(config_path)
    try:
        fd, tmp_path = tempfile.mkstemp(dir=dir_name, suffix=".tmp")
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            json.dump(config, f, indent=2, ensure_ascii=False)
            f.write("\n")
        os.replace(tmp_path, config_path)
        log(f"配置已写入 {config_path}")
    except Exception:
        # 清理临时文件
        if os.path.exists(tmp_path):
            os.unlink(tmp_path)
        raise

    return True


def main():
    config_path = sys.argv[1] if len(sys.argv) > 1 else DEFAULT_CONFIG

    if not os.path.isfile(config_path):
        log(f"配置文件不存在: {config_path}")
        sys.exit(1)

    log(f"开始更新 {config_path}")

    try:
        changed = update_config(config_path)
    except Exception as e:
        log(f"更新过程出错: {e}")
        sys.exit(1)

    if changed:
        restart_xray()
    else:
        log("无需重启 xray")

    log("完成")


if __name__ == "__main__":
    main()
```
