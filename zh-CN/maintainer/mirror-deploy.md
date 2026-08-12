# 镜像站部署指南

## 1 方案概览

Harmonybrew 的资源分布在这些 URL 下：

| URL | 用途 | brew 环境变量 |
|---|---|---|
| `https://harmonybrew.atomgit.com/install.sh` | 安装脚本 | — |
| `https://harmonybrew.atomgit.com/brew/brew.tar.gz` | brew 程序包 | `HOMEBREW_BOTTLE_DOMAIN` |
| `https://harmonybrew.atomgit.com/api/<file>.json` | Formula/Cask API | `HOMEBREW_API_DOMAIN` |
| `https://harmonybrew.atomgit.com/bottles/<file>.tar.gz` | 预编译 bottle | `HOMEBREW_BOTTLE_DOMAIN` |
| `https://atomgit.com/Harmonybrew/brew.git` | brew 代码仓 | `HOMEBREW_BREW_GIT_REMOTE` |
| `https://atomgit.com/Harmonybrew/homebrew-core.git` | homebrew-core 代码仓 | `HOMEBREW_CORE_GIT_REMOTE` |

**目标**：用一台 nginx 服务器反代上述全部资源，让鸿蒙用户只需指向一个镜像地址即可安装和使用 Harmonybrew。终端用户**无需设置任何代理变量**。

两种部署形态：

- **直连版**——镜像服务器可直通互联网，nginx 直接反代上游 HTTPS 并**带文件缓存**（bottle 长期缓存、API 短 TTL）。最简单。
- **企业内网版**——镜像服务器在企业内网，需经出口代理（带 `username:password` 认证）才能出网。在 nginx 与上游之间插入本地 Python 正向代理脚本 `forwardproxy.py`，由它携带企业代理认证出网，终端用户依然无感。

## 2 前置条件

| 依赖项 | 要求 |
|---|---|
| 操作系统 | Linux |
| nginx | 1.18+（标准包即可，无需第三方模块） |
| 工具 | `curl`、`git` |
| 权限 | root 或 sudo（写 `/etc/nginx/conf.d/`、reload nginx） |
| 网络 | 直连版需能访问 `harmonybrew.atomgit.com`、`atomgit.com`；企业版见 [§7](#7-企业内网部署经出口代理出网) |
| 端口 | 80（默认，假设镜像服务器为专用机，80 空闲） |

安装依赖包（若未装）：

```bash
apt update && apt install -y nginx curl git    # Debian/Ubuntu
# 或: yum install -y nginx curl git                 # RHEL/CentOS
```

## 3 五分钟快速部署（直连版）

```bash
# 1. 建目录（静态说明页 + 缓存目录）
mkdir -p /var/www/harmonybrew-mirror /var/cache/nginx/harmonybrew
chown -R www-data:www-data /var/cache/nginx 2>/dev/null || chown -R nginx:nginx /var/cache/nginx

# 2. 移除 nginx 默认站点（新装 nginx 时 /etc/nginx/sites-enabled/default 会占用 80 端口，
#    导致我们的配置 listen 80 冲突，必须先删）
rm -f /etc/nginx/sites-enabled/default

# 3. 写 nginx 配置（见 §5.1，复制到 /etc/nginx/conf.d/harmonybrew-mirror.conf）
#    写根说明页（见 §5.2，复制到 /var/www/harmonybrew-mirror/index.html）

# 4. 校验 + reload
nginx -t && nginx -s reload

# 5. 自检（应全部 200，/bottles/ 为 404 属正常）
MIRROR=http://127.0.0.1
curl -s -o /dev/null -w "%{http_code}\n" $MIRROR/health                                    # 200
curl -s -o /dev/null -w "%{http_code}\n" $MIRROR/install.sh                                # 200
curl -sI $MIRROR/brew/brew.tar.gz | head -1                                                # 200
curl -sI $MIRROR/api/formula.jws.json | head -1                                            # 200
git clone --depth 1 $MIRROR/git/homebrew-core.git /tmp/core-test && rm -rf /tmp/core-test  # 成功
```

至此镜像站已在 `:80` 上线。客户端使用方式见 [§6](#6-客户端使用)。


## 4 镜像路径说明

镜像 nginx 监听 **80**（假设镜像服务器为专用机，80 端口空闲）。

| 镜像路径 | 上游 | brew 环境变量取值 |
|---|---|---|
| `http://<mirror>/` | 说明页（本地静态） | — |
| `http://<mirror>/install.sh` | `harmonybrew.atomgit.com/install.sh` | — |
| `http://<mirror>/brew/brew.tar.gz` | `harmonybrew.atomgit.com/brew/brew.tar.gz` | `HOMEBREW_BOTTLE_DOMAIN=http://<mirror>/bottles` |
| `http://<mirror>/api/` | `harmonybrew.atomgit.com/api/` | `HOMEBREW_API_DOMAIN=http://<mirror>/api` |
| `http://<mirror>/bottles/` | `harmonybrew.atomgit.com/bottles/` | `HOMEBREW_BOTTLE_DOMAIN=http://<mirror>/bottles` |
| `http://<mirror>/git/brew.git` | `atomgit.com/Harmonybrew/brew.git` | `HOMEBREW_BREW_GIT_REMOTE=http://<mirror>/git/brew.git` |
| `http://<mirror>/git/homebrew-core.git` | `atomgit.com/Harmonybrew/homebrew-core.git` | `HOMEBREW_CORE_GIT_REMOTE=http://<mirror>/git/homebrew-core.git` |
| `http://<mirror>/health` | 本地健康检查 | — |

> git 仓镜像路径设计为 `/git/brew.git`、`/git/homebrew-core.git`（带 `.git` 后缀），与 brew 默认 remote 形态一致，用户无需额外调整。


## 5 完整 nginx 配置

### 5.1 直连版配置

保存到 `/etc/nginx/conf.d/harmonybrew-mirror.conf`：

```nginx
# Harmonybrew 镜像站 - 直连版（带缓存）
# 部署: cp 此文件 /etc/nginx/conf.d/ && nginx -t && nginx -s reload
# 缓存策略: bottle 超长 TTL（365 天, 永不覆盖）, api/install.sh/brew.tar.gz 短 TTL（10 分钟）, git 不缓存

# ===== 缓存目录（放 conf.d 会被 include 进 http{} 块） =====
proxy_cache_path /var/cache/nginx/harmonybrew
                 levels=1:2
                 keys_zone=hb_cache:10m
                 max_size=50g
                 inactive=30d
                 use_temp_path=off;

upstream hb_harmonybrew { server harmonybrew.atomgit.com:443; keepalive 16; }
upstream hb_atomgit     { server atomgit.com:443;           keepalive 16; }

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    access_log /var/log/nginx/harmonybrew-mirror.access.log;
    error_log  /var/log/nginx/harmonybrew-mirror.error.log warn;

    client_max_body_size 0;
    gzip off;

    # ===== 公共缓存指令 =====
    proxy_cache hb_cache;
    proxy_cache_lock on;              # 并发同文件只回源一次, 防击穿
    proxy_cache_revalidate on;        # 过期后用 If-Modified-Since 轻量回源
    proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;  # 仅上游故障时返旧兜底, 不在过期时返旧
    add_header X-Cache-Status $upstream_cache_status always;

    # ===== git brew 仓: 不缓存（git smart http 动态协议） =====
    location /git/brew.git/ {
        proxy_cache off;
        proxy_pass https://hb_atomgit/Harmonybrew/brew.git/;
        proxy_ssl_server_name on;
        proxy_ssl_name atomgit.com;
        proxy_set_header Host atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Accept $http_accept;
        proxy_set_header Accept-Encoding $http_accept_encoding;
        proxy_set_header Cookie $http_cookie;
        proxy_set_header Git-Protocol $http_git_protocol;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }

    # ===== git homebrew-core 仓: 不缓存 =====
    location /git/homebrew-core.git/ {
        proxy_cache off;
        proxy_pass https://hb_atomgit/Harmonybrew/homebrew-core.git/;
        proxy_ssl_server_name on;
        proxy_ssl_name atomgit.com;
        proxy_set_header Host atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Accept $http_accept;
        proxy_set_header Accept-Encoding $http_accept_encoding;
        proxy_set_header Cookie $http_cookie;
        proxy_set_header Git-Protocol $http_git_protocol;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }

    # ===== API: 短 TTL（10 分钟）, 频繁更新需及时刷新 =====
    location /api/ {
        proxy_pass https://hb_harmonybrew/api/;
        proxy_ssl_server_name on;
        proxy_ssl_name harmonybrew.atomgit.com;
        proxy_set_header Host harmonybrew.atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_cache_valid 200 10m;
        proxy_cache_valid 301 302 1h;
        proxy_cache_valid any 1m;
        proxy_read_timeout 120s;
    }

    # ===== Bottle: 永不覆盖, 超长 TTL（365 天）, 过期概念本就无意义 =====
    location /bottles/ {
        proxy_pass https://hb_harmonybrew/bottles/;
        proxy_ssl_server_name on;
        proxy_ssl_name harmonybrew.atomgit.com;
        proxy_set_header Host harmonybrew.atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering on;
        proxy_cache_valid 200 365d;
        proxy_cache_valid 206 365d;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }

    # ===== install.sh: 短 TTL（10 分钟） =====
    location = /install.sh {
        proxy_pass https://hb_harmonybrew/install.sh;
        proxy_ssl_server_name on;
        proxy_ssl_name harmonybrew.atomgit.com;
        proxy_set_header Host harmonybrew.atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        default_type application/x-sh;
        proxy_cache_valid 200 10m;
        proxy_cache_valid any 1m;
    }

    # ===== brew.tar.gz: 短 TTL（10 分钟） =====
    location /brew/ {
        proxy_pass https://hb_harmonybrew/brew/;
        proxy_ssl_server_name on;
        proxy_ssl_name harmonybrew.atomgit.com;
        proxy_set_header Host harmonybrew.atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering on;
        proxy_cache_valid 200 10m;
        proxy_cache_valid 206 10m;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }

    # ===== 根说明页（本地静态） =====
    location = / {
        proxy_cache off;
        root /var/www/harmonybrew-mirror;
        try_files /index.html =200;
        default_type text/html;
    }

    # ===== 健康检查 =====
    location = /health {
        proxy_cache off;
        access_log off;
        default_type text/plain;
        return 200 'ok\n';
    }
}
```

### 5.2 根说明页

保存到 `/var/www/harmonybrew-mirror/index.html`：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head><meta charset="utf-8"><title>Harmonybrew Mirror</title></head>
<body>
<h1>Harmonybrew Mirror Site</h1>
<p>Harmonybrew（鸿蒙版 Homebrew）镜像站，反代自
<a href="https://harmonybrew.atomgit.com">harmonybrew.atomgit.com</a>
与 <a href="https://atomgit.com/Harmonybrew">atomgit.com/Harmonybrew</a>。</p>
<h2>使用方法</h2>
<h3>全新安装 Harmonybrew</h3>
<pre><code>export HOMEBREW_API_DOMAIN=http://YOUR_MIRROR/api
export HOMEBREW_BOTTLE_DOMAIN=http://YOUR_MIRROR/bottles
export HOMEBREW_BREW_GIT_REMOTE=http://YOUR_MIRROR/git/brew.git
export HOMEBREW_CORE_GIT_REMOTE=http://YOUR_MIRROR/git/homebrew-core.git
zsh -c "$(curl -fsSL http://YOUR_MIRROR/install.sh)"</code></pre>
<p>安装完成后，将上述 export 操作写入 ~/.zshrc。</p>
<h3>已安装 Harmonybrew，换成镜像源</h3>
<p>将以下 export 操作写进 ~/.zshrc，然后 source ~/.zshrc（或新开终端）使其生效，再 brew update。</p>
<pre><code>export HOMEBREW_API_DOMAIN=http://YOUR_MIRROR/api
export HOMEBREW_BOTTLE_DOMAIN=http://YOUR_MIRROR/bottles
export HOMEBREW_BREW_GIT_REMOTE=http://YOUR_MIRROR/git/brew.git
export HOMEBREW_CORE_GIT_REMOTE=http://YOUR_MIRROR/git/homebrew-core.git</code></pre>
</body>
</html>
```

### 5.3 缓存策略

缓存让多用户共享一份回源数据，同一文件只需向上游取一次，后续命中本地即可返回。本方案按文件更新特征分两类缓存：

| 路径 | 更新特征 | 缓存策略 | TTL |
|---|---|---|---|
| `/bottles/` | **永不覆盖**，只发新文件 | 超长 TTL，命中即返回本地 | **365 天** |
| `/api/*.json` | 频繁更新（formula/cask 索引） | 短 TTL，过期后回源覆盖 | **10 分钟** |
| `/install.sh`、`/brew/brew.tar.gz` | 偶尔更新 | 短 TTL | 10 分钟 |
| `/git/*` | 动态协议（git smart http） | **不缓存**，透传 | — |

关键指令（已写在 §5.1 配置里）：

```nginx
proxy_cache_path /var/cache/nginx/harmonybrew levels=1:2 keys_zone=hb_cache:10m max_size=50g inactive=30d use_temp_path=off;
proxy_cache_lock on;            # 并发同文件只回源一次, 防击穿
proxy_cache_revalidate on;      # 过期后用 If-Modified-Since 轻量回源, 未变返 304 不重传
proxy_cache_use_stale error timeout http_5xx;  # 仅上游故障时返旧兜底, 保可用; 不在过期时返旧
proxy_cache_valid 200 365d;     # bottle: 永不覆盖, 超长 TTL
proxy_cache_valid 200 10m;      # api/install.sh/brew.tar.gz: 10 分钟
add_header X-Cache-Status $upstream_cache_status always;  # 观察命中: MISS/HIT/EXPIRED/STALE
```

**如何确认缓存生效**：响应头 `X-Cache-Status` 第一次是 `MISS`（回源），之后是 `HIT`（命中本地）。过期后变 `EXPIRED`（回源覆盖）。`STALE` 仅在回源失败（上游 5xx/超时）时出现，返旧兜底保可用。

**容量规划**：`max_size=50g` 是缓存上限（所有缓存文件加起来的体积上限，超出后 nginx 自动按 LRU 淘汰最旧缓存项）。缓存服务器磁盘通常宽裕，按实际磁盘空间与服务用户规模再调。

**清理缓存**（如需强制刷新某文件）：

```bash
# 清全部缓存
rm -rf /var/cache/nginx/harmonybrew/* && nginx -s reload
```


## 6 客户端使用

### 6.1 新安装

**必须设置全部四个环境变量**（两个 DOMAIN + 两个 GIT_REMOTE），否则 brew 运行时下载 bottle 和读 API 会直连上游，绕过镜像。这是所有 Homebrew 镜像源的通用要求，这几个变量要全程存在。

```bash
export HOMEBREW_API_DOMAIN=http://<镜像服务器IP或域名>/api
export HOMEBREW_BOTTLE_DOMAIN=http://<镜像服务器IP或域名>/bottles
export HOMEBREW_BREW_GIT_REMOTE=http://<镜像服务器IP或域名>/git/brew.git
export HOMEBREW_CORE_GIT_REMOTE=http://<镜像服务器IP或域名>/git/homebrew-core.git
zsh -c "$(curl -fsSL http://<镜像服务器IP或域名>/install.sh)"
```

安装完成后，将上述 `export` 写入 `~/.mkshrc`（容器 mksh）或 `~/.zshrc`（PC zsh）持久化，确保后续每次 `brew` 调用都走镜像。

### 6.2 已安装用户切换镜像源

同样需要设全四个环境变量：

```bash
export HOMEBREW_API_DOMAIN=http://<镜像服务器IP或域名>/api
export HOMEBREW_BOTTLE_DOMAIN=http://<镜像服务器IP或域名>/bottles
export HOMEBREW_BREW_GIT_REMOTE=http://<镜像服务器IP或域名>/git/brew.git
export HOMEBREW_CORE_GIT_REMOTE=http://<镜像服务器IP或域名>/git/homebrew-core.git
brew update
```

将上述 `export` 写入 `~/.mkshrc`（容器 mksh）或 `~/.zshrc`（PC zsh）持久化。

### 6.3 确认镜像生效

```bash
brew config | grep -iE "DOMAIN|REMOTE"    # 应显示指向镜像的四个变量
tail -f /var/log/nginx/harmonybrew-mirror.access.log   # install/bottle/api/git 请求都应出现
```


## 7 企业内网部署

企业内网服务器无法直连互联网，需经出口代理（常带 `username:password` 认证）出网。本方案在 nginx 与上游之间插入一个 **Python 正向代理脚本** `forwardproxy.py`，由它统一携带企业代理认证出网，**终端用户完全无感**——照常指向 `http://镜像服务器IP或域名`，无需在客户端设任何代理变量。

### 7.1 拓扑

```
终端用户(内网) ──http──> nginx(:80) ──http──> forwardproxy.py(:8888,仅本地)
                                             │ 携带企业代理认证
                                             ▼
                                      企业出口代理(proxy.example.com:8080)
                                             ▼
                                   互联网上游(atomgit.com / harmonybrew.atomgit.com)
```

**关键机制**：nginx 不直连上游 HTTPS，而是 `proxy_pass http://127.0.0.1:8888`（本地 Python 脚本）；脚本收到请求后按 nginx 透传的 `Host` 头确定上游，对 HTTPS 上游发起 CONNECT 隧道，HTTP 上游直接转发；若设了 `-u` 参数则把请求转给企业出口代理并携带 Basic 认证。

### 7.2 forwardproxy.py 脚本

保存到 `/opt/harmonybrew-mirror/forwardproxy.py`（仅 Python 3 标准库，无需 pip install）：

```python
#!/usr/bin/env python3
# ============================================================================
# forwardproxy.py — 极简 HTTP/HTTPS 正向代理（仅标准库）
#
# 用途:
#   企业内网镜像站场景，nginx 把所有上游请求转发到本脚本，
#   本脚本再挂到企业出口代理（携带认证）出网，终端用户无感。
#
# 功能:
#   - 处理 HTTP 请求（GET/POST/...）：转发到上游或企业代理
#   - 处理 HTTPS 请求（CONNECT）：建立双向隧道，可选经企业代理
#   - 企业上游代理支持 Basic 认证（http://user:pass@host:port）
#   - 并发处理（多线程）
#
# 用法:
#   python3 forwardproxy.py                         # 监听 0.0.0.0:8888, 直连出网
#   python3 forwardproxy.py -p 8888                 # 指定端口
#   python3 forwardproxy.py -u http://user:pass@proxy.example.com:8080   # 挂企业代理
#   python3 forwardproxy.py -b 127.0.0.1            # 仅本地（推荐，配合 nginx）
#
# 配合 nginx: nginx proxy_pass http://127.0.0.1:8888 + Host 上游域名
#   本脚本收到请求后，按 Host 头连对应上游；若设了 -u，则走企业代理并携带认证
# ============================================================================
import argparse
import base64
import select
import socket
import ssl
import threading
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlsplit

BUF = 65536


def parse_upstream(url):
    """解析 http://[user:pass@]host:port -> (host, port, auth_header_or_None)"""
    if not url:
        return None
    s = urlsplit(url)
    host = s.hostname
    port = s.port or 8080
    auth = None
    if s.username:
        token = base64.b64encode(f"{s.username}:{s.password or ''}".encode()).decode()
        auth = f"Proxy-Authorization: Basic {token}\r\n"
    return (host, port, auth)


class ProxyHandler(BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"
    # 静默默认日志，避免刷屏；可按需打开
    def log_message(self, fmt, *args):
        pass

    # ---------- HTTPS: CONNECT 隧道 ----------
    def do_CONNECT(self):
        host, port = self.path.split(":") if ":" in self.path else (self.path, "443")
        port = int(port)
        upstream = self.server.upstream  # None 或 (host,port,auth)

        try:
            if upstream:
                # 经企业代理: 先连企业代理, 发 CONNECT 并携带认证, 等 200 响应
                uh, up, uauth = upstream
                remote = socket.create_connection((uh, up), timeout=30)
                req = f"CONNECT {host}:{port} HTTP/1.1\r\nHost: {host}:{port}\r\n"
                if uauth:
                    req += uauth
                req += "Connection: keep-alive\r\n\r\n"
                remote.sendall(req.encode())
                # 读代理响应（只读到响应头）
                resp = b""
                while b"\r\n\r\n" not in resp:
                    chunk = remote.recv(BUF)
                    if not chunk:
                        break
                    resp += chunk
                if b" 200" not in resp.split(b"\r\n", 1)[0]:
                    self.send_error(502, f"upstream proxy CONNECT failed: {resp[:120]}")
                    remote.close()
                    return
            else:
                # 直连上游
                remote = socket.create_connection((host, port), timeout=30)
        except Exception as e:
            self.send_error(502, f"connect failed: {e}")
            return

        # 隧道建立, 告诉客户端 OK, 然后双向透传
        self.send_response(200, "Connection Established")
        self.end_headers()

        conns = [self.connection, remote]
        try:
            while True:
                r, _, _ = select.select(conns, [], conns, 60)
                if not r:
                    break
                for s in r:
                    data = s.recv(BUF)
                    if not data:
                        conns = []
                        break
                    (remote if s is self.connection else self.connection).sendall(data)
                if not conns:
                    break
        finally:
            remote.close()

    # ---------- HTTP: GET/POST/... 转发 ----------
    def do_METHOD(self):
        upstream = self.server.upstream
        host_header = self.headers.get("Host", "")
        if self.path.startswith("http://") or self.path.startswith("https://"):
            s = urlsplit(self.path)
            target_host = s.hostname
            target_port = s.port or (443 if self.path.startswith("https") else 80)
            path = s.path or "/"
            if s.query: path += "?" + s.query
        else:
            if ":" in host_header:
                target_host, target_port = host_header.rsplit(":", 1); target_port = int(target_port)
            else:
                target_host, target_port = host_header, 80
            path = self.path
        if upstream:
            uh, up, uauth = upstream
            target_host, target_port = uh, up
            path = f"http://{host_header}{self.path}"

        headers = f"{self.command} {path} HTTP/1.1\r\n"
        has_host = False
        for k, v in self.headers.items():
            kl = k.lower()
            if kl == "proxy-authorization": continue
            if kl == "host": has_host = True
            headers += f"{k}: {v}\r\n"
        if not has_host and host_header: headers += f"Host: {host_header}\r\n"
        if upstream and uauth: headers += uauth
        headers += "Connection: close\r\n\r\n"

        body = b""
        cl = int(self.headers.get("Content-Length", 0) or 0)
        if cl > 0:
            remaining = cl
            while remaining > 0:
                chunk = self.rfile.read(min(BUF, remaining))
                if not chunk: break
                body += chunk; remaining -= len(chunk)

        resp = self._fetch(target_host, target_port, headers.encode(), body)

        # 关键: 上游（尤其 atomgit）对 HTTP 请求会 302 到 HTTPS.
        # 若透传 302, git 客户端会跟随重定向直连 HTTPS, 企业内网连不上 443 就失败.
        # 脚本自己跟随重定向: 用 CONNECT 隧道取 HTTPS 内容, 对客户端透明返回 200.
        for _ in range(10):
            status, loc = self._parse_redirect(resp)
            if status is None: break
            ls = urlsplit(loc)
            if ls.scheme != "https": break
            rh, rp = ls.hostname, ls.port or 443
            rpath = ls.path or "/"
            if ls.query: rpath += "?" + ls.query
            tunnel = self._open_tunnel(rh, rp, upstream)
            if tunnel is None: break
            rheaders = f"{self.command} {rpath} HTTP/1.1\r\nHost: {rh}\r\n"
            for k, v in self.headers.items():
                kl = k.lower()
                if kl in ("proxy-authorization","host","connection","content-length","transfer-encoding"): continue
                rheaders += f"{k}: {v}\r\n"
            rheaders += f"Host: {rh}\r\nConnection: close\r\n"
            if body: rheaders += f"Content-Length: {len(body)}\r\n"
            rheaders += "\r\n"
            resp = self._fetch_on_tunnel(tunnel, rheaders.encode() + body)
            tunnel.close()

        try: self.wfile.write(resp)
        except Exception: pass

    def _fetch(self, host, port, req_bytes, body):
        try: remote = socket.create_connection((host, port), timeout=30)
        except Exception as e:
            return f"HTTP/1.1 502 connect failed: {e}\r\nContent-Length: 0\r\n\r\n".encode()
        remote.sendall(req_bytes + body)
        chunks = []
        try:
            while True:
                d = remote.recv(BUF)
                if not d: break
                chunks.append(d)
        except Exception: pass
        finally: remote.close()
        return b"".join(chunks)

    def _open_tunnel(self, host, port, upstream):
        try:
            if upstream:
                uh, up, uauth = upstream
                remote = socket.create_connection((uh, up), timeout=30)
                req = f"CONNECT {host}:{port} HTTP/1.1\r\nHost: {host}:{port}\r\n"
                if uauth: req += uauth
                req += "Connection: keep-alive\r\n\r\n"
                remote.sendall(req.encode())
                r = b""
                while b"\r\n\r\n" not in r:
                    c = remote.recv(BUF)
                    if not c: break
                    r += c
                if b" 200" not in r.split(b"\r\n", 1)[0]:
                    remote.close(); return None
            else:
                remote = socket.create_connection((host, port), timeout=30)
            # HTTPS 隧道需在 socket 上包 SSL
            ctx = ssl.create_default_context()
            ctx.check_hostname = False
            ctx.verify_mode = ssl.CERT_NONE
            remote = ctx.wrap_socket(remote, server_hostname=host)
            return remote
        except Exception: return None

    def _fetch_on_tunnel(self, tunnel, req_bytes):
        tunnel.sendall(req_bytes)
        chunks = []
        try:
            while True:
                d = tunnel.recv(BUF)
                if not d: break
                chunks.append(d)
        except Exception: pass
        return b"".join(chunks)

    @staticmethod
    def _parse_redirect(resp):
        head = resp.split(b"\r\n\r\n", 1)[0]
        lines = head.split(b"\r\n")
        if not lines: return None, None
        try: status = int(lines[0].split()[1])
        except Exception: return None, None
        if status not in (301, 302, 303, 307, 308): return None, None
        for l in lines[1:]:
            if l.lower().startswith(b"location:"):
                return status, l.split(b":", 1)[1].strip().decode()
        return None, None

    # 所有方法都走 do_METHOD
    do_GET = do_POST = do_HEAD = do_PUT = do_DELETE = do_OPTIONS = do_METHOD


def main():
    p = argparse.ArgumentParser(description="极简 HTTP/HTTPS 正向代理")
    p.add_argument("-b", "--bind", default="127.0.0.1", help="监听地址，默认 127.0.0.1")
    p.add_argument("-p", "--port", type=int, default=8888, help="监听端口，默认 8888")
    p.add_argument("-u", "--upstream", default="", help="企业上游代理 http://[user:pass@]host:port")
    args = p.parse_args()

    upstream = parse_upstream(args.upstream) if args.upstream else None
    server = ThreadingHTTPServer((args.bind, args.port), ProxyHandler)
    server.upstream = upstream
    print(f"forwardproxy on {args.bind}:{args.port}  upstream={upstream or 'direct'}")
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        server.shutdown()


if __name__ == "__main__":
    main()
```

### 7.3 部署步骤

```bash
# 1. 建目录（静态说明页 + 缓存目录）
mkdir -p /var/www/harmonybrew-mirror /var/cache/nginx/harmonybrew
chown -R www-data:www-data /var/cache/nginx 2>/dev/null || chown -R nginx:nginx /var/cache/nginx

# 2. 删 nginx 默认站点避免 80 端口冲突
rm -f /etc/nginx/sites-enabled/default

# 3. 放脚本到固定位置
mkdir -p /opt/harmonybrew-mirror
cp forwardproxy.py /opt/harmonybrew-mirror/
chmod +x /opt/harmonybrew-mirror/forwardproxy.py

# 4. 写企业版 nginx 配置到 /etc/nginx/conf.d/harmonybrew-mirror.conf（见 §7.4）
#    写根说明页到 /var/www/harmonybrew-mirror/index.html（同 §5.2）

# 5. 写 systemd unit（把 -u 参数换成你的企业代理地址，无需认证就删掉末尾的 -u http://... 参数）
tee /etc/systemd/system/forwardproxy.service > /dev/null <<'EOF'
[Unit]
Description=Harmonybrew Mirror Forward Proxy
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/harmonybrew-mirror/forwardproxy.py -b 127.0.0.1 -p 8888 -u http://username:password@proxy.example.com:8080
Restart=always
RestartSec=3
StandardOutput=append:/var/log/forwardproxy.log
StandardError=append:/var/log/forwardproxy.log

[Install]
WantedBy=multi-user.target
EOF

# 6. 启用脚本（开机自启、崩溃自恢复）
systemctl daemon-reload
systemctl enable --now forwardproxy
systemctl status forwardproxy    # 看运行状态，应 active (running)
ss -tlnp | grep 8888                  # 应见 LISTEN，否则按 §7.6 排查

# 7. reload nginx
nginx -t && nginx -s reload

# 8. 自检（应全部 200，/bottles/ 为 404 属正常）
MIRROR=http://127.0.0.1
curl -s -o /dev/null -w "%{http_code}\n" $MIRROR/health                                    # 200
curl -s -o /dev/null -w "%{http_code}\n" $MIRROR/install.sh                                # 200
curl -sI $MIRROR/brew/brew.tar.gz | head -1                                                # 200
curl -sI $MIRROR/api/formula.jws.json | head -1                                            # 200
git clone --depth 1 $MIRROR/git/homebrew-core.git /tmp/core-test && rm -rf /tmp/core-test  # 成功
```

> 步骤 4 的 unit 文件参数说明见 §7.5。

### 7.4 企业版 nginx 配置

与直连版（§5.1）的唯一区别：所有 `proxy_pass` 从 `https://上游域名` 改为 `http://本地脚本`，其余透传规则（UA/Cookie/Host/Range）和缓存策略**完全一致**。保存到 `/etc/nginx/conf.d/harmonybrew-mirror.conf`：

```nginx
# Harmonybrew 镜像站 - 企业内网版（带缓存）
# 与直连版的唯一区别: proxy_pass 指向本地 forwardproxy.py（127.0.0.1:8888）,
#                     由脚本携企业代理认证出网. 缓存策略相同.

proxy_cache_path /var/cache/nginx/harmonybrew
                 levels=1:2
                 keys_zone=hb_cache:10m
                 max_size=50g
                 inactive=30d
                 use_temp_path=off;

upstream hb_local_proxy { server 127.0.0.1:8888; keepalive 32; }

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    access_log /var/log/nginx/harmonybrew-mirror.access.log;
    error_log  /var/log/nginx/harmonybrew-mirror.error.log warn;

    client_max_body_size 0;
    gzip off;

    proxy_cache hb_cache;
    proxy_cache_lock on;
    proxy_cache_revalidate on;
    proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
    add_header X-Cache-Status $upstream_cache_status always;

    # ===== git brew 仓: 不缓存 =====
    location /git/brew.git/ {
        proxy_cache off;
        proxy_pass http://hb_local_proxy/Harmonybrew/brew.git/;
        proxy_set_header Host atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Cookie $http_cookie;
        proxy_set_header Git-Protocol $http_git_protocol;
        proxy_set_header Accept $http_accept;
        proxy_set_header Accept-Encoding $http_accept_encoding;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }

    # ===== git homebrew-core 仓: 不缓存 =====
    location /git/homebrew-core.git/ {
        proxy_cache off;
        proxy_pass http://hb_local_proxy/Harmonybrew/homebrew-core.git/;
        proxy_set_header Host atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Cookie $http_cookie;
        proxy_set_header Git-Protocol $http_git_protocol;
        proxy_set_header Accept $http_accept;
        proxy_set_header Accept-Encoding $http_accept_encoding;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }

    # ===== API: 短 TTL（10 分钟） =====
    location /api/ {
        proxy_pass http://hb_local_proxy/api/;
        proxy_set_header Host harmonybrew.atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_cache_valid 200 10m;
        proxy_cache_valid 301 302 1h;
        proxy_cache_valid any 1m;
        proxy_read_timeout 120s;
    }

    # ===== Bottle: 永不覆盖, 超长 TTL（365 天） =====
    location /bottles/ {
        proxy_pass http://hb_local_proxy/bottles/;
        proxy_set_header Host harmonybrew.atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering on;
        proxy_cache_valid 200 365d;
        proxy_cache_valid 206 365d;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }

    # ===== install.sh: 短 TTL（10 分钟） =====
    location = /install.sh {
        proxy_pass http://hb_local_proxy/install.sh;
        proxy_set_header Host harmonybrew.atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        default_type application/x-sh;
        proxy_cache_valid 200 10m;
        proxy_cache_valid any 1m;
    }

    # ===== brew.tar.gz: 短 TTL（10 分钟） =====
    location /brew/ {
        proxy_pass http://hb_local_proxy/brew/;
        proxy_set_header Host harmonybrew.atomgit.com;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_pass_request_headers on;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering on;
        proxy_cache_valid 200 10m;
        proxy_cache_valid 206 10m;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }

    # ===== 根说明页（本地静态） =====
    location = / {
        proxy_cache off;
        root /var/www/harmonybrew-mirror;
        try_files /index.html =200;
        default_type text/html;
    }

    location = /health {
        proxy_cache off;
        access_log off;
        default_type text/plain;
        return 200 'ok\n';
    }
}
```

### 7.5 systemd unit 参数说明

§7.3 步骤 4 写入的 unit 文件各字段含义：

| 字段 | 说明 |
|---|---|
| `ExecStart` | 脚本启动命令。`-u` 指企业上游代理（`http://user:pass@host:port`），无需认证就删掉末尾的 `-u http://...` 参数；`-b 127.0.0.1` 仅本地监听（配合 nginx，勿改）；`-p 8888` 监听端口，与 §7.4 nginx 的 `upstream hb_local_proxy` 端口须一致 |
| `Restart=always` + `RestartSec=3` | 崩溃后 3 秒自动恢复 |
| `StandardOutput`/`StandardError` | 日志统一写 `/var/log/forwardproxy.log`，排查看此文件 |
| `WantedBy=multi-user.target` | 开机自启 |

改完 unit 后需 `systemctl daemon-reload && systemctl restart forwardproxy` 生效。

### 7.6 故障排查

| 现象 | 排查 |
|---|---|
| nginx 日志 `502 Bad Gateway` | 脚本未起或 8888 未监听：`systemctl status forwardproxy`、`ss -tlnp \| grep 8888` |
| 脚本日志 `407 Proxy Authentication Required` | `-u` 的用户名密码错；或企业代理要求 NTLM/Kerberos（本脚本仅支持 Basic，需在企业代理侧开 Basic 或换 squid） |
| 脚本日志 `Connection refused` | `-u` 的企业代理地址/端口错 |
| git clone 卡住或超时 | 企业代理对长连接/git 协议有限流；脚本已设 60s 隧道超时，可改源码 `select(..., 60)` 调大 |
| API/bottle 慢 | 企业代理带宽小；考虑调大 `proxy_cache_path` 的 `max_size` 延长缓存命中 |

### 7.7 客户端使用

终端用户**无需设置任何 HTTP_PROXY/HTTPS_PROXY**，只指向镜像——企业代理认证完全在镜像服务器侧（forwardproxy.py）透明完成：

```bash
export HOMEBREW_API_DOMAIN=http://<镜像服务器IP或域名>/api
export HOMEBREW_BOTTLE_DOMAIN=http://<镜像服务器IP或域名>/bottles
export HOMEBREW_BREW_GIT_REMOTE=http://<镜像服务器IP或域名>/git/brew.git
export HOMEBREW_CORE_GIT_REMOTE=http://<镜像服务器IP或域名>/git/homebrew-core.git
zsh -c "$(curl -fsSL http://<镜像服务器IP或域名>/install.sh)"
```

安装完成后，将上述 `export` 写入 `~/.mkshrc`（容器 mksh）或 `~/.zshrc`（PC zsh）持久化，确保后续每次 `brew` 调用都走镜像。


## 8 常见问题

### 8.1 忘记设 `HOMEBREW_BOTTLE_DOMAIN` / `HOMEBREW_API_DOMAIN`

不设这两个变量，brew 安装阶段下载 portable-git/portable-ruby bottle 会直连上游。**要让 bottle 和 API 也走镜像，必须全程显式 export 全部四个环境变量**（见 §6.1/§6.2）。这是所有 Homebrew 镜像源的通用要求。

### 8.2 缓存调优

§5.1 配置已默认开启缓存（见 §5.3 策略说明）。常见调优：

- **改 TTL**：编辑对应 location 的 `proxy_cache_valid`，如 API 想更实时改 `200 5m`，bottle 想更久改 `200 90d`
- **改容量**：`proxy_cache_path` 的 `max_size`（默认 50g），按磁盘空间调
- **改缓存目录**：`proxy_cache_path` 第一个参数，建议放独立磁盘/SSD
- **内存索引**：`keys_zone=hb_cache:10m` 是缓存键的内存索引，约 8 万个 key 的容量，对绝大多数镜像规模绰绰有余，通常无需调整
- **确认命中**：看响应头 `X-Cache-Status`，或 `curl -I` 观察是否 `HIT`
- **强制刷新某文件**：删 `rm -rf /var/cache/nginx/harmonybrew/*` 后 reload

### 8.3 端口冲突

默认 listen 80，假设镜像服务器为专用机。若 80 被占用，改配置里的 `listen 80 default_server` 为其他端口（如 8080），或用 `server_name` 区分。


## 9 卸载

```bash
# 通用清理（直连版/企业版都做）
rm /etc/nginx/conf.d/harmonybrew-mirror.conf
rm -rf /var/www/harmonybrew-mirror /var/cache/nginx/harmonybrew
nginx -t && nginx -s reload

# 企业版额外清理
systemctl stop forwardproxy
systemctl disable forwardproxy
rm /etc/systemd/system/forwardproxy.service
systemctl daemon-reload
rm -rf /opt/harmonybrew-mirror
```


## 附录：方案设计要点速查

| 设计点 | 原因 |
|---|---|
| `upstream` + `keepalive` + `Connection ""` | 复用上游 TLS 连接，降低握手开销 |
| `proxy_pass` 带路径形式 `.../Harmonybrew/brew.git/` | nginx 自动重写 `/git/brew.git/` 为上游路径 |
| bottle/`brew.tar.gz` 透传 `Range` + buffering on | 支持断点续传，配合 proxy_cache 写入缓存 |
| 根路径返回静态说明页 | 满足 `install.sh` 开头对镜像根地址的 HEAD 探活（HTTP 通即放行） |
| `gzip off` | bottle 是已压缩 tar.gz，再 gzip 与 Range 冲突 |
| `proxy_cache` 分层 TTL | bottle 永不覆盖→超长 TTL（365d）；API/install.sh/brew.tar.gz 会更新→短 TTL（10m）；git 动态→不缓存 |
| `proxy_cache_lock` + `proxy_cache_revalidate` | 并发防击穿+过期后用 If-Modified-Since 轻量回源（未变返 304 不重传） |
| `proxy_cache_use_stale error timeout http_5xx` | 仅上游故障时返旧兜底保可用；不在过期时返旧，过期即回源取新（短 TTL 资源需及时刷新） |
| 企业版用 forwardproxy.py 而非重编译 nginx | 仅 Python 3 标准库零依赖，`-u http://user:pass@host:port` 命令行传认证 |
