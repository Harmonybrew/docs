# 流水线部署指南

## 1 前言

本文档供项目维护者或有二次开发需求（需要自己部署一整套 Harmonybrew 项目）的用户用作参考，普通用户无需阅读。

本项目的 CI 系统基于“华为云基础设施 + 开源流水线系统 [Buildbot](https://github.com/buildbot/buildbot)”构建。动手能力强的开发者可参考其逻辑，自行迁移至其他云厂商或其他流水线系统。

> 由于项目迭代频率高，本文档中的操作不保证能完美复现，仅供参考。

## 2 前置准备

### 2.1 ECS

需要准备一些服务器，承担不同的职责。

这里假设我们准备了 5 台服务器，我们分别给他们取不同的名字、分配不同的职责

| 主机     | 职责                                                               |
| -------- | ------------------------------------------------------------------ |
| master   | 作为 Buildbot 的 Master 节点，负责调度 CI 任务、提供 Web 页面      |
| worker-1 | 作为 Buildbot 的 Worker 节点，负责运行流水线脚本、执行编译构建任务 |
| worker-2 | 作为 Buildbot 的 Worker 节点，负责运行流水线脚本、执行编译构建任务 |
| worker-3 | 作为 Buildbot 的 Worker 节点，负责运行流水线脚本、执行编译构建任务 |
| worker-4 | 作为 Buildbot 的 Worker 节点，负责运行流水线脚本、执行编译构建任务 |

这些服务器需要满足以下条件：

1. Ubuntu 24.04 arm64
2. Region 在香港或国外（CI 执行过程中需要去 GitHub 下载源码和开发工具，中国大陆的服务器无法稳定访问 GitHub）
3. 使用 root 用户

### 2.2 域名

准备一个属于你自己的域名，且最好完成 ICP 备案。这个域名后续将用来作为 CDN 加速域名。

由于华为云现在已经不卖域名，你需要去其他云厂商或专门的域名服务商购买域名。

域名准备好后，还需要给你的域名申请一份 SSL 证书。如果你是个人开发者，为节省费用，建议使用 [Let's Encrypt](https://letsencrypt.org/) 提供的免费证书。

### 2.3 OBS 和 CDN

前往 OBS 控制台，创建一个 OBS 桶用来存储软件包，名字和 Region 随意。

前往 CDN 控制台，将你的域名配置为 CDN 加速域名，CDN 回源地址指向你的 OBS 桶。

CDN 服务范围的选择：

- 如果完成了 ICP 备案，可以选“中国大陆”或“全球”。
- 如果没完成 ICP 备案，只能选“中国大陆境外”。

CDN 加速域名配置完成之后，把你的 SSL 证书也部署到 CDN 上，以便后续使用 https 链接。

### 2.4 SWR

需要在 SWR 控制台里面创建一个组织。流水线构建出来的 Docker 镜像会上传到这个组织上。

### 2.5 环境变量

准备好以下环境变量的值，接下来的部署步骤会用上。

<table>
  <thead>
    <tr>
      <th>变量名</th>
      <th>变量值获取方法</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ATOMGIT_TOKEN</td>
      <td>创建一个 AtomGit 账号作为机器人账号，将其加入你 fork 的 homebrew-core 仓库，
         成为项目成员，并赋予维护者权限。之后进入个人中心创建访问令牌，这个变量记录的就是访问令牌的值。
      </td>
    </tr>
    <tr>
      <td>HUAWEICLOUD_SDK_AK</td>
      <td>在华为云控制台右上角找到“我的凭证”，进入该页面操作生成。</td>
    </tr>
    <tr>
      <td>HUAWEICLOUD_SDK_SK</td>
      <td>同上，AK/SK 是一对的。</td>
    </tr>
    <tr>
      <td>OBS_BUCKET</td>
      <td>在上面的步骤已经创建过 OBS 桶，请把上面创建的桶名记录下来。</td>
    </tr>
    <tr>
      <td>OBS_REGION</td>
      <td>在上面的步骤已经创建过 OBS 桶，请把所属 Region 记录下来。</td>
    </tr>
    <tr>
      <td>JWS_SIGNING_KEY</td>
      <td>
        手动创建一对 RSA 密钥。
        命令：<code>openssl genrsa -out private.pem 2048 &amp;&amp; openssl rsa -in private.pem -pubout -out public.pem</code>。
        生成出来的密钥对要保管好，里面的私钥就是 JWS_SIGNING_KEY 的值，
        公钥需要放到 brew 代码仓中，覆盖代码仓中原有的 homebrew-1.pem 文件。
      </td>
    </tr>
    <tr>
      <td>JWS_SIGNING_KEY_ID</td>
      <td>固定值 homebrew-1。</td>
    </tr>
    <tr>
      <td>HOMEBREW_ATOMGIT_API_TOKEN</td>
      <td>跟上面的 ATOMGIT_TOKEN 是同一个数据，无需重复创建。</td>
    </tr>
    <tr>
      <td>HOMEBREW_GITHUB_API_TOKEN</td>
      <td>在 GitHub 上面用自己的个人账号生成一个 token。</td>
    </tr>
    <tr>
      <td>SWR_REGION</td>
      <td>在上面的步骤已经创建过 SWR 组织，请把上面创建的组织名记录下来。</td>
    </tr>
    <tr>
      <td>SWR_ORGANIZATION</td>
      <td>在上面的步骤已经创建过 SWR 组织，请把所属 Region 记录下来。</td>
    </tr>
  </tbody>
</table>

## 3 配置 Master

Master 是调度中心，负责管理 Worker、接收 Webhook、分发构建任务。以下命令在 Master 服务器上执行。Web UI 和 Webhook 均通过 Master 对外暴露。

### 3.1 创建系统用户

以专用用户 `buildbot` 运行服务，隔离权限。

```bash
useradd -r -s /bin/bash -d /opt/buildbot -m buildbot
passwd -l buildbot
```

### 3.2 安装 Python 虚拟环境

独立隔离 Buildbot 依赖，不污染系统 Python。

```bash
mkdir -p /opt/buildbot/masters/default
python3 -m venv /opt/buildbot/.venv
/opt/buildbot/.venv/bin/pip install buildbot[bundle]
# 设置目录权限
chown -R buildbot:buildbot /opt/buildbot/
```

### 3.3 初始化 Master

生成 Master 骨架目录，包含默认配置文件 `master.cfg` 和 `buildbot.tac`。

```bash
/opt/buildbot/.venv/bin/buildbot create-master /opt/buildbot/masters/default
```

### 3.4 写入 master.cfg

下载 master.cfg 模板文件

```bash
curl -fsSL -o /opt/buildbot/masters/default/master.cfg https://raw.gitcode.com/Harmonybrew/ci/raw/main/buildbot/master.cfg
```

Master 核心配置包含 Worker、Scheduler、Builder 工厂、Web UI 等。

**填充模板**

将模板文件占位符用实际值进行填充，其中的密码用 `openssl rand` 命令生成。

```bash
worker_password=$(openssl rand -base64 14)
web_ui_password=$(openssl rand -hex 8)
sed -i "s/YOUR_WORKER_PASSWORD/${worker_password}/g" /opt/buildbot/masters/default/master.cfg
sed -i "s/YOUR_WEBUI_PASSWORD/${web_ui_password}/g" /opt/buildbot/masters/default/master.cfg

# 将占位符 YOUR_PUBLIC_IP 替换成 master 节点的公网 IP，这里假设这个值为 1.2.3.4
sed -i "s/YOUR_PUBLIC_IP/1.2.3.4/g" /opt/buildbot/masters/default/master.cfg

# 将占位符 YOUR_OBS_BUCKET 替换成章节 2.5 里面的 OBS_BUCKET，这里假设这个值为 harmonybrew
sed -i "s/YOUR_OBS_BUCKET/harmonybrew/g" /opt/buildbot/masters/default/master.cfg

# 将占位符 YOUR_OBS_REGION 替换成章节 2.5 里面的 OBS_REGION，这里假设这个值为 cn-north-4
sed -i "s/YOUR_OBS_REGION/cn-north-4/g" /opt/buildbot/masters/default/master.cfg

# 将占位符 YOUR_SWR_REGION 替换成章节 2.5 里面的 SWR_REGION，这里假设这个值为 cn-north-4
sed -i "s/YOUR_SWR_REGION/cn-north-4/g" /opt/buildbot/masters/default/master.cfg

# 将占位符 YOUR_SWR_ORGANIZATION 替换成章节 2.5 里面的 SWR_ORGANIZATION，这里假设这个值为 harmonybrew
sed -i "s/YOUR_SWR_ORGANIZATION/harmonybrew/g" /opt/buildbot/masters/default/master.cfg
```

### 3.5 写入 atomgit_webhook.py

该文件是自定义 Webhook Handler，用于解析 AtomGit merge_request 事件并过滤后注入 Buildbot Change。

```bash
curl -fsSL -o /opt/buildbot/masters/default/atomgit_webhook.py https://raw.gitcode.com/Harmonybrew/ci/raw/main/buildbot/atomgit_webhook.py
```

### 3.6 填写 secrets_store 中的密钥

密钥存储在 secrets_store 目录下的文件中，通过 `SecretInAFile` 读取，Buildbot 自动在日志中遮蔽。

```bash
# 创建目录
mkdir -p /opt/buildbot/masters/default/secrets_store
chown -R buildbot:buildbot /opt/buildbot/masters/default/secrets_store
```

以下每个文件写入真实密钥值（替换 `你的…` 占位符）：

```bash
# Webhook 验证 token
echo "$(openssl rand -hex 12)" | tee /opt/buildbot/masters/default/secrets_store/atomgit_webhook_token

# AtomGit API token
echo "章节 2.5 里面的 ATOMGIT_TOKEN" | tee /opt/buildbot/masters/default/secrets_store/atomgit_token

# 华为云 AK/SK（OBS 上传需要）
echo "章节 2.5 里面的 HUAWEICLOUD_SDK_AK" | tee /opt/buildbot/masters/default/secrets_store/huaweicloud_sdk_ak
echo "章节 2.5 里面的 HUAWEICLOUD_SDK_SK" | tee /opt/buildbot/masters/default/secrets_store/huaweicloud_sdk_sk

# JWS 签名密钥（auto_merge 需要）
echo "章节 2.5 里面的 JWS_SIGNING_KEY"    | tee /opt/buildbot/masters/default/secrets_store/jws_signing_key
echo "章节 2.5 里面的 JWS_SIGNING_KEY_ID" | tee /opt/buildbot/masters/default/secrets_store/jws_signing_key_id

# GitHub API token（auto_bump 需要）
echo "章节 2.5 里面的 HOMEBREW_GITHUB_API_TOKEN" | tee /opt/buildbot/masters/default/secrets_store/homebrew_github_api_token

# 统一设置密钥文件权限
chmod 600 /opt/buildbot/masters/default/secrets_store/*
```

### 3.7 安装 systemd 服务并启动

以 systemd 管理 Master 进程，支持开机自启和异常退出后自动重启。

```bash
tee /etc/systemd/system/buildbot@.service << 'UNIT_EOF'
[Unit]
Description=Buildbot master %i
After=network.target
StartLimitBurst=5
StartLimitIntervalSec=5m

[Service]
Type=simple
User=buildbot
Group=buildbot
Environment="PATH=/opt/buildbot/.venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
WorkingDirectory=/opt/buildbot/masters/%i
ExecStart=/opt/buildbot/.venv/bin/buildbot start --nodaemon
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
UNIT_EOF


# 启动
systemctl daemon-reload
systemctl enable buildbot@default
systemctl start buildbot@default
```

### 3.8 验证 Master

```bash
systemctl status buildbot@default
tail -f /opt/buildbot/masters/default/twistd.log
```

## 4 配置 Worker

Worker 是执行构建任务的节点，接收 Master 下发的命令执行。以下命令在 Worker 所在机器上执行。

### 4.1 创建系统用户和虚拟环境

> 如果 Worker 和 Master 在同一台机器上，系统用户和虚拟环境已创建，跳到 4.2。

在 Worker 所在机器上执行：

```bash
# 创建 buildbot 系统用户
useradd -r -s /bin/bash -d /opt/buildbot -m buildbot
passwd -l buildbot

# 安装 Python 虚拟环境
mkdir -p /opt/buildbot/workers
python3 -m venv /opt/buildbot/.venv
/opt/buildbot/.venv/bin/pip install buildbot-worker
# 设置目录权限
chown -R buildbot:buildbot /opt/buildbot/
```

### 4.2 创建 Worker 实例

`create-worker` 生成 Worker 骨架，`buildbot.tac` 中记录 Master 地址和认证信息。

```bash
worker=<WORKER_NAME>
worker_password=<YOUR_WORKER_PASSWORD>
master_ip=<MASTER_IP>

mkdir -p /opt/buildbot/workers/${worker:?}
/opt/buildbot/.venv/bin/buildbot-worker create-worker \
    /opt/buildbot/workers/${worker:?} \
    ${master_ip:?}:9989 \
    ${worker:?} \
    ${worker_password:?}

# 设置目录权限
chown -R buildbot:buildbot /opt/buildbot/workers/
```

- worker 的名称 <WORKER_NAME>（即 master.cfg 文件定义的四个 worker 之一：worker1/worker2/worker3/worker4） 和密码 <YOUR_WORKER_PASSWORD> 要与 master.cfg 文件里的保持一致。
- <MASTER_IP> 即第 3 章部署 master 填充的 YOUR_PUBLIC_IP，如果 worker 与 master 有局域网连接，就直接使用 master 的局域网 ip。

### 4.3 写入 Worker 信息

```bash
echo "admin@example.com" > /opt/buildbot/workers/${worker:?}/info/admin
echo "$(hostname)"       > /opt/buildbot/workers/${worker:?}/info/host
```

> Web UI 展示 worker 信息时包含这里的 admin 和 host。

### 4.4 安装 systemd 服务并启动

以 systemd 管理 Worker 进程，支持开机自启和异常退出后自动重启。

```bash
tee /etc/systemd/system/buildbot-worker@.service << 'UNIT_EOF'
[Unit]
Description=Buildbot worker %i
After=network.target
StartLimitBurst=5
StartLimitIntervalSec=5m

[Service]
Type=simple
User=buildbot
Group=buildbot
Environment="PATH=/opt/buildbot/.venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
WorkingDirectory=/opt/buildbot/workers/%i
ExecStart=/opt/buildbot/.venv/bin/buildbot-worker start --nodaemon
Restart=on-failure
RestartSec=10
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
UNIT_EOF


systemctl daemon-reload
systemctl enable buildbot-worker@${worker:?}
systemctl start buildbot-worker@${worker:?}
```

### 4.5 添加 Docker 支持

Worker 上执行 Docker 构建需要将 `buildbot` 用户加入 `docker` 组，并重启 worker 服务生效：

```bash
usermod -aG docker buildbot
systemctl restart buildbot-worker@${worker:?}
```

### 4.6 验证 Worker

```bash
systemctl status buildbot-worker@${worker:?}
tail -f /opt/buildbot/workers/${worker:?}/twistd.log
```

Master 上确认连接：

```bash
grep "attached" /opt/buildbot/masters/default/twistd.log
```

## 5 配置 nginx 反向代理

Buildbot Master 监听内部端口 18010，nginx 对外暴露 8010。可结合云服务器安全组限制 Web UI 请求来源。Webhook 路径和 Web UI 共用同一端口。

### 5.1 写入 buildbot.conf

nginx 反向代理配置：将外部 8010 端口转发到 Buildbot 内部 18010，并支持 WebSocket 升级。

```bash
tee /etc/nginx/conf.d/buildbot.conf << 'NGINX_EOF'
# /etc/nginx/conf.d/buildbot.conf
server {
    # 监听端口（对外暴露的端口）
    listen 8010;

    # 如果你的 Buildbot 要通过域名访问，把 _ 改成你的域名
    # 如果想暂时用 IP 访问，保留 _ 或写公网 IP
    server_name _;

    # ========== 主站点代理 ==========
    location / {
        proxy_pass http://127.0.0.1:18010;

        # --- 基础代理头（必选） ---
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # --- WebSocket 支持（对普通请求无害） ---
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # --- 超时设置（WebSocket 长连接必备） ---
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 3600s;  # WebSocket 空闲连接保持 1 小时

        # --- 缓冲控制（Buildbot 实时日志需要禁用缓冲） ---
        proxy_buffering off;
        proxy_cache off;
    }

    # ========== 静态资源缓存优化（可选） ==========
    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        proxy_pass http://127.0.0.1:18010;
        proxy_set_header Host $host;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # ========== 健康检查端点（可选） ==========
    location /health {
        proxy_pass http://127.0.0.1:18010/health;
        access_log off;
    }
}
NGINX_EOF
```

### 5.2 加载 nginx 配置并启动

检查配置语法后，根据 nginx 是否已运行选择热重载或安装启动。

**如果已有 nginx 运行中：**

```bash
# 检查新增配置语法
nginx -t

# 热重载（不中断现有服务）
nginx -s reload
```

**如果未安装 nginx：**

```bash
# 安装
apt install nginx -y

# 启动
systemctl enable nginx
systemctl start nginx
```

> IP 白名单由云服务器安全组控制，或修改 `location /` 添加 `allow / deny`。

## 6 对接 Webhook

Webhook URL：`http://1.2.3.4:8010/change_hook/atomgit?token=你的token` ，1.2.3.4 改成master公网ip，此处的 token 即在 secrets_store 目录写入的 atomgit_webhook_token

将以上 URL 写入 homebrew-core 仓库的 Webhook URL。

webhook 测试：

```bash
curl -v -X POST "http://1.2.3.4:8010/change_hook/atomgit?token=你的token&dry_run=1" \
  -H "Content-Type: application/json" \
  -d '{"event_type":"merge_request","user":{"name":"test"},"project":{"name":"test"},"repository":{"homepage":"http://example.com"},"object_attributes":{"title":"test","url":"http://example.com"}}'
```

> 该命令不实际触发流水线，正常返回码为 200。

## 7 后续修改配置

Master 配置修改后（如 master.cfg、atomgit_webhook.py），视情况进行热重载或重启：

```bash
# 验证语法正确
/opt/buildbot/.venv/bin/buildbot checkconfig /opt/buildbot/masters/default/master.cfg

# 仅 master.cfg 变化 → 热重载（不中断构建）
systemctl reload buildbot@default

# atomgit_webhook.py 有变化或添加了新的密钥文件 → 必须重启（执行中的构建会直接停止）
systemctl restart buildbot@default
```

Worker 配置（buildbot.tac）修改后：

```bash
systemctl restart buildbot-worker@worker1
```

## 8 快速排查

| 症状              | 检查                                                                                                                   |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Master 启动失败   | `tail /opt/buildbot/masters/default/twistd.log`；常见：语法错误、端口占用、密钥权限不是 600                            |
| Worker 连接失败   | Worker 日志 `tail /opt/buildbot/workers/worker1/twistd.log`；核对 Master IP、端口 9989、Worker 名称和密码              |
| Webhook 不触发    | `?token=` 是否匹配 `secrets_store/atomgit_webhook_token`；event_type 是否为 `merge_request`；label 是否为 `request-ci` |
| Webhook 403       | secrets_store/* 文件权限是否为 600                                                                                     |
| Web UI 登录后空白 | `authz` 只能用 `AnyControlEndpointMatcher`，不能用 `AnyEndpointMatcher`                                                |
| Web UI 连接超时   | `buildbotURL` 和 `allowed_origins` 必须是公网 IP（nginx 对外端口 8010）；nginx 是否已启动                              |
| 配置不生效        | Master 重启用 `restart` 而非 `reload`                                                                                  |
