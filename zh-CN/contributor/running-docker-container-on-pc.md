# 在鸿蒙 PC 中运行 Docker 容器

鸿蒙 PC 原生环境无法使用 Docker 和 Podman 等工具，在“融合开发引擎”中也因为缺少内核选项而无法使用 Docker，但 Podman 是可用的。我们可以在“融合开发引擎”中使用 Podman 来运行容器。

从安装 Podman 到运行容器的完整操作流程如下，所有命令均在“融合开发引擎”中执行：

```sh
# 创建 Podman 配置文件
sudo mkdir -p /etc/containers

sudo tee /etc/containers/storage.conf << 'EOF' > /dev/null
[storage]
# 1. 驱动设置为 vfs
driver = "vfs"
# 2. 设置 runroot
runroot = "/run/containers/storage"
# 3. 设置数据持久化目录
graphroot = "/var/lib/containers/storage"

[storage.options]
# 4. 【关键】确保这一行被注释掉（前面加 #），或者是空的
# mount_program = "/usr/bin/fuse-overlayfs"
EOF

# 安装 Podman
sudo dnf update -y
sudo dnf install podman -y

# 创建并挂载 cgroup
sudo mkdir -p /sys/fs/cgroup
sudo mount -t cgroup2 cgroup2 /sys/fs/cgroup

# 拉取镜像并运行容器，以 ubuntu 容器为例（注意网络模式必须是 host）
sudo podman pull ubuntu:latest
sudo podman run \
  -itd \
  --name=my-test \
  --network=host \
  --cgroup-manager=cgroupfs \
  --pids-limit=-1 \
  ubuntu:latest
sudo podman exec -it my-test sh
```
