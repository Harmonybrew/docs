# 设备农场部署指南

## 1 方案简介

在 Harmonybrew 项目中，有一条 [auto_dailytest](https://atomgit.com/Harmonybrew/ci/blob/main/src/auto_dailytest.py) 流水线。这条流水线会每天在鸿蒙 PC 真机上测试当天更新过的 formula。

由于厂商并未提供云化的鸿蒙 PC 环境，当前我们通过设备农场的方式来处理测试设备的问题。

整体的方案是这样的：

- CodeArts Pipeline 平台将 CI 任务下发到 Linux 执行机，Linux 执行机会执行 auto_dailytest.py 脚本。
- 在 auto_dailytest.py 脚本中，我们通过 ssh 命令去操作鸿蒙 PC，执行 `brew test` 等命令。
- 由于鸿蒙 PC 没有公网 IP，因此鸿蒙 PC 的 ssh 端口需要通过内网穿透工具来暴露给 Linux 执行机。

```mermaid
graph TD
    %% 节点定义
    subgraph CI ["CI 环境"]
        Pipeline["CodeArts Pipeline"] -->|下发 CI 任务| Executer["Linux 执行机<br/>(auto_dailytest.py)"]
    end

    subgraph Server ["内网穿透服务器 (公网 IP)"]
        RatholeServer["rathole 服务端<br/>隧道控制通道监听 0.0.0.0:57392<br/>业务端口监听 0.0.0.0:22001-22004"]
    end

    subgraph Farm ["鸿蒙 PC 设备农场 (内网环境)"]
        direction TB
        PC1["鸿蒙 PC 1<br/>sshd: 9022"]
        PC2["鸿蒙 PC 2<br/>sshd: 9022"]
        PC3["鸿蒙 PC 3<br/>sshd: 9022"]
        PC4["鸿蒙 PC 4<br/>sshd: 9022"]
    end

    %% 连线关系 (注意这里给包含括号的文本加了双引号)
    Executer -->|"SSH 操作 (22001-22004)"| RatholeServer

    PC1 -.->|"通过 57392 建立隧道"| RatholeServer
    PC2 -.->|"通过 57392 建立隧道"| RatholeServer
    PC3 -.->|"通过 57392 建立隧道"| RatholeServer
    PC4 -.->|"通过 57392 建立隧道"| RatholeServer

    RatholeServer -.-> |转发 22001 到 9022| PC1
    RatholeServer -.-> |转发 22002 到 9022| PC2
    RatholeServer -.-> |转发 22003 到 9022| PC3
    RatholeServer -.-> |转发 22004 到 9022| PC4
```

## 2 前置准备

### 2.1 准备设备

需要准备如下设备：

- 带有公网 IP 的 Linux 服务器（作为内网穿透服务器）
- 鸿蒙 PC 真机，多台（作为执行机）

### 2.2 规划端口

准备好设备后，需要规划端口映射关系，比如这样：

| 鸿蒙 PC | sshd 端口 | 内网穿透服务器端口 |
| ------- | --------- | ------------------ |
| pc_1    | 9022      | 22001              |
| pc_2    | 9022      | 22002              |
| pc_3    | 9022      | 22003              |
| pc_4    | 9022      | 22004              |

### 2.3 放通安全组

修改内网穿透服务器的安全组，放通 57392 和 22001~22004 端口。仅对受信任的公网出口 IP（CI 执行机出口、鸿蒙 PC 出口）放通，切勿对 0.0.0.0 放通。出口 IP 不固定时应在变更后及时同步安全组规则。

## 3 配置内网穿透服务器

首先需要在 Linux 服务器上下载 rathole

```sh
mkdir -p /opt/rathole/bin

# Linux x86_64 环境：下载 GNU 版本
curl -LO https://github.com/rathole-org/rathole/releases/download/v0.5.0/rathole-x86_64-unknown-linux-gnu.zip
unzip rathole-x86_64-unknown-linux-gnu.zip
mv rathole /opt/rathole/bin/rathole
rm rathole-x86_64-unknown-linux-gnu.zip

# Linux aarch64 环境：下载 musl 静态链接版本（也适用于 GNU 发行版）
curl -LO https://github.com/rathole-org/rathole/releases/download/v0.5.0/rathole-aarch64-unknown-linux-musl.zip
unzip rathole-aarch64-unknown-linux-musl.zip
mv rathole /opt/rathole/bin/rathole
rm rathole-aarch64-unknown-linux-musl.zip
```

为 Noise Protocol 生成一对 X25519 密钥

```sh
/opt/rathole/bin/rathole --genkey
```

创建 rathole 配置文件 `/opt/rathole/server.toml`

```ini
[server]
# 隧道控制通道对外监听端口
bind_addr = "0.0.0.0:57392"

[server.transport]
# 启用 Noise Protocol
type = "noise"

[server.transport.noise]
# 上面 `rathole --genkey` 生成的 Private Key
local_private_key = "your_server_private_key_here"

[server.services.pc_1_ssh]
# token 用 openssl rand -hex 16 生成，不同 service 各不相同
token = "your_secure_token_here"
# 需要对公网暴露的端口
bind_addr = "0.0.0.0:22001"

[server.services.pc_2_ssh]
token = "your_secure_token_here"
bind_addr = "0.0.0.0:22002"

[server.services.pc_3_ssh]
token = "your_secure_token_here"
bind_addr = "0.0.0.0:22003"

[server.services.pc_4_ssh]
token = "your_secure_token_here"
bind_addr = "0.0.0.0:22004"
```

创建 rathole 专属用户，将相关文件分配给这个用户（权限最小化原则）

```sh
useradd -r -s /bin/false rathole
chown -R rathole:rathole /opt/rathole
```

创建 systemd 配置文件 `/etc/systemd/system/rathole.service`

```ini
[Unit]
Description=Rathole Server
After=network.target
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
Type=simple
User=rathole
Group=rathole
WorkingDirectory=/opt/rathole
ExecStart=/opt/rathole/bin/rathole -s /opt/rathole/server.toml
Restart=always
RestartSec=10

# ===== 日志与 IO =====
StandardOutput=journal
StandardError=journal
SyslogIdentifier=rathole
StandardInput=null

# ===== 资源限制 =====
LimitNOFILE=65536
TasksMax=256
LimitCORE=0
MemoryMax=512M
LimitNPROC=64

# ===== 文件系统隔离 =====
ReadOnlyPaths=/
ReadWritePaths=/opt/rathole
PrivateTmp=true
InaccessiblePaths=/home /root /run/user

# ===== namespace 隔离 =====
PrivateMounts=true
PrivateIPC=true

# ===== 内核接口收紧 =====
NoNewPrivileges=true
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources @cpu-emulation @debug @module @mount @swap @clock @setuid @chown @raw-io
CapabilityBoundingSet=
AmbientCapabilities=
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# ===== 写权限全局收紧 =====
ProtectSystem=strict
DevicePolicy=closed
ProtectControlGroups=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectKernelLogs=true
ProtectHostname=true
ProtectClock=true
ProtectProc=invisible

# ===== 进程硬化补充 =====
LockPersonality=true
RestrictRealtime=true
RestrictSUIDSGID=true
RestrictNamespaces=true

[Install]
WantedBy=multi-user.target
```

启动 rathole 服务端

```sh
systemctl daemon-reload
systemctl enable rathole
systemctl start rathole
```

## 4 配置鸿蒙 PC

### 4.1 打开安全开关

进入“设置” -> “系统” -> “开发者选项”，打开“开发者选项”开关。

进入“设置” -> “隐私和安全” -> “高级”，打开“运行来自非应用市场的扩展程序”开关。

> 如果你的系统不显示“开发者选项”菜单，则需要通过隐藏开关来开启：打开“设置” -> “关于本机”（即进入“设置”时的默认界面），找到“软件版本”，连续点击 7 次，出现弹窗后点击“确认重启并开启”即可。

### 4.2 下载工具

下载 openssh

```sh
cd ~
curl -fLO https://github.com/Harmonybrew/ohos-openssh/releases/download/10.4p1/openssh-10.4p1-ohos-arm64.tar.gz
tar -zxf openssh-10.4p1-ohos-arm64.tar.gz
rm openssh-10.4p1-ohos-arm64.tar.gz
```

> openssh 必须放在家目录下，因为编译时设置的 prefix 就是 `/storage/Users/currentUser`，放置到其他目录它将无法正常工作。

下载 rathole

```sh
cd ~
curl -LO https://harmonybrew.atomgit.com/api/formula.jws.json
RATHOLE_BOTTLE=$(cat formula.jws.json | grep -oE 'rathole-[0-9\._]+\.arm64_ohos\.bottle(\.[0-9]+)?\.tar\.gz')
curl -LO "https://harmonybrew.atomgit.com/bottles/$RATHOLE_BOTTLE"
tar -zxf $RATHOLE_BOTTLE
rm formula.jws.json $RATHOLE_BOTTLE
```

### 4.3 准备配置文件

准备一份 rathole 的配置文件 `~/rathole/client.toml`

```ini
[client]
# 云服务器的公网 IP 和 rathole 通信端口（1.2.3.4 替换为实际 IP）
remote_addr = "1.2.3.4:57392"

[client.transport]
# 启用 Noise Protocol，必须与服务端一致
type = "noise"

[client.transport.noise]
# 服务端 `rathole --genkey` 生成的 Public Key
remote_public_key = "your_server_public_key_here"

[client.services.pc_1_ssh]
# token 必须与服务端一致
token = "your_secure_token_here"
# 本机的 sshd 端口
local_addr = "127.0.0.1:9022"
```

> 不同执行机的配置文件内容不一样。比如服务名称，分别是 `pc_1_ssh`、`pc_2_ssh`、`pc_3_ssh`、`pc_4_ssh`。此外 token 也各不相同。

### 4.4 启动服务

启动 sshd

```sh
~/openssh-10.4p1-ohos-arm64/sbin/sshd -p 9022
```

启动 rathole

```sh
cd ~/rathole
nohup ~/rathole/0.5.0/bin/rathole -c ~/rathole/client.toml &
cd ~
```

启动完服务后，请勿关闭 HiShell。可以把界面最小化，但不能关闭。关闭 HiShell 可能导致后台进程被杀。

> 如果要停止服务，可以用 pkill 命令：`pkill sshd && pkill rathole`。

### 4.5 配置 ssh 公钥

将 Linux 执行机的 ssh 公钥添加到鸿蒙 PC 上的 `~/.ssh/authorized_keys` 文件中

```sh
mkdir -p ~/.ssh && echo "ssh-ed25519 AAAA..." >> ~/.ssh/authorized_keys
```
