# Storage
## 镜像、层与容器
Docker镜像由一系列只读层堆叠而成，每个层代表Dockerfile中的一条指令（FROM, COPY, RUN 等，CMD, LABEL等元数据操作不创建新层）。

层是增量的，只包含与上一层的差异。删除文件也会创建一个新层，并且被删除的文件在之前的层中依然存在，会占用总镜像空间。

容器 = 镜像的只读层 + 一个顶部的薄可写层（容器层）。所有对运行中容器的更改都发生在这个可写层。

当容器被删除时，其可写层也会被删除，底层镜像保持不变。多个容器可以共享同一个基础镜像的只读层，同时拥有各自的容器层。

## 写时复制
共享：多个容器或镜像层访问相同文件时，只需存储一份基础数据，节省空间和网络传输（拉取/推送镜像时）。

复制：只有当某个层需要修改一个文件时，该文件才会被复制到该层并进行修改。第一次修改会引发 copy_up 操作，可能带来性能开销（尤其对大文件、深目录结构），但后续修改无此开销。


## 存储与挂载行为
``` shell
# 1.创建一个名为 "myapp-data" 的卷
docker volume create myapp-data
# 运行一个容器，并将卷挂载到容器内的 /data 路径
docker run -d --name mysql-container -v myapp-data:/var/lib/mysql mysql:8.0

# 2.直接挂载
docker run -d -v /home/user/myapp:/app nginx:alpine

# 3.挂载tmpfs(最大的作用是提供内存级别的快速读写，可降低某些进程对被分配docker内存的占用)
docker run -d --tmpfs /app/temp-sessions nginx:alpine

# 4.挂载Named pipes
docker run -d --mount type=npipe,source=\\.\pipe\sql\query,target=\\.\pipe\sql\query my-windows-app

```

### 挂载目录的行为
* 挂载到非空目录：如果将卷挂载到容器内已有文件的目录，原有文件会被遮盖（隐藏），直到卷被卸载。
* 挂载空卷：如果将空卷挂载到容器内已有文件的目录，Docker 默认会将容器目录的内容复制到卷中（可用 volume-nocopy 选项禁用此行为）。
* 匿名卷：未命名的卷，具有随机生成的唯一名称。除非使用 --rm 选项启动容器（此时匿名卷会随容器删除），否则也会持久化。

### 核心存储驱动 (Storage Driver)
* 首选驱动：overlay2。适用于绝大多数场景（99%），是当前稳定性和性能的综合最优解。
* 备选驱动：btrfs, zfs, vfs。仅在你有特定文件系统优势需求时才考虑。
* 作用范围：容器层。它管理容器镜像层和运行时可写层的联合挂载，主要负责容器生命周期的存储。

### 持久化数据存储 (Persistent Data)
* 对于需要持久化的数据（如数据库文件、应用程序状态），必须使用卷（Volume）或绑定挂载（Bind Mount），而非依赖容器层。
* Volume (卷)：生产环境首选。由 Docker 管理，与容器生命周期解耦，适合数据持久化和共享。
* Bind Mount (绑定挂载)：将宿主机特定目录直接挂载到容器中。简单直观，但可移植性差。
* Docker 支持通过卷驱动（Volume Drivers）挂载远程或分布式存储，实现数据共享和持久化。支持类型：包括 local（默认）、nfs、cifs/samba 以及各种云存储（如 sftp, aws-ebs, azurefile）和分布式文件系统（如 ceph, glusterfs）。

### volume挂载子目录(volume-subpath)
``` shell
$ docker run --rm \
  --mount src=logs,dst=/logs \
  alpine mkdir -p /logs/app1 /logs/app2
$ docker run -d \
  --name=app1 \
  --mount src=logs,dst=/var/log/app1,volume-subpath=app1 \
  app1:latest
$ docker run -d \
  --name=app2 \
  --mount src=logs,dst=/var/log/app2,volume-subpath=app2 \
  app2:latest

```

## 优化建议
* 为写入密集型应用或需要持久化的数据使用卷（Volumes），而不是依赖容器的可写层。
* 优化Dockerfile（如使用多阶段构建、合并 RUN 指令、谨慎处理文件删除）以减少镜像层数和总大小。
* 存储驱动的作用范围是容器层，在高频写入的场景，例如数据库data，依然推荐使用volume
* 首选的存储的驱动是overlay2（99%情况下），但是也支持btrfs、zfs、vfs
* 行业标准的容器运行时 containerd 使用快照程序（snapshotters） 来存储镜像和容器数据，在docker中可以通过配置使用

# Network
docker会默认创建iptables chains:DOCKER-USER、DOCKER-FORWARD、DOCKER、DOCKER-ISOLATION-STAGE-1、DOCKER-ISOLATION-STAGE-2、DOCKER-INGRESS

## 限制到容器的外部连接
要仅允许特定 IP 或网络访问容器，请在 DOCKER-USER 过滤器链的顶部插入一条否定规则。例如，以下规则丢弃来自除 192.0.2.2 之外的所有 IP 地址的数据包：
```shell
$ iptables -I DOCKER-USER -i ext_if ! -s 192.0.2.2 -j DROP
$ iptables -I DOCKER-USER -i ext_if ! -s 192.0.2.0/24 -j DROP
$ iptables -I DOCKER-USER -m iprange -i ext_if ! --src-range 192.0.2.1-192.0.2.3 -j DROP
```
## 网络驱动
**bridge（桥接）**：默认的网络驱动程序。

**host（主机）**：移除容器和 Docker 主机之间的网络隔离，直接使用主机的网络。

**overlay（覆盖）**：将多个 Docker 守护进程连接在一起，并使 Swarm 服务和容器能够跨节点通信。

**ipvlan**：IPvlan 网络让用户能够完全控制 IPv4 和 IPv6 地址分配。

**macvlan**：Macvlan 网络允许您为容器分配 MAC 地址，使其在您的网络上显示为物理设备。在处理期望直接连接到物理网络而不是通过 Docker 主机的网络栈进行路由的遗留应用程序时，使用 macvlan 驱动程序有时是最佳选择。

**none（无）**：将容器与主机和其他容器完全隔离。

## docker Swarm
Docker Swarm 是 Docker 官方的原生集群管理和编排工具，它允许您将多个 Docker 主机组成一个单一的虚拟系统，从而轻松部署和管理分布式应用。

```shell
# 初始化 Swarm 集群
docker swarm init

# 将其他节点加入集群
docker swarm join --token <token> <manager-ip>:2377

# 创建服务
docker service create --name web --replicas 3 -p 80:80 nginx

# 扩展服务
docker service scale web=5

# 滚动更新
docker service update --image nginx:latest web
```

# Containers

## 一些命令
--restart：支持Docker自动重启
--filter：对docker的结果进行过滤，类似 docker images --filter reference=alpine
--format：格式化日志输出，实际语法使用可以参考AI即可
docker stats：查看容器运行时指标
docker completion：为 Docker CLI 生成 shell 补全脚本

## 限制容器对内存/CPU的访问

--memory-reservation:对内存进行软限制
--memory-swappiness：调整内存使用交换内存的百分比
--kernel-memory：可以允许使用的最大内核内存大小
--cpus=<value>：配置cpu使用量

对于资源限制，当在 cgroup v2 和 systemd 环境下运行时才支持。如果 docker info 显示 Cgroup Driver 为 none，则表示条件不满足。

## 配置Docker代理
不要使用 ENV Dockerfile 指令来为构建指定代理设置，使用环境变量设置代理会将配置嵌入到镜像中。如果代理是内部代理，则从该镜像创建的容器可能无法访问它。

修改~/.docker/config.json文件
```json
{
 "proxies": {
   "default": {
     "httpProxy": "http://proxy.example.com:3128",
     "httpsProxy": "https://proxy.example.com:3129",
     "noProxy": "*.test.example.com,.example.org,127.0.0.0/8"
   },
   "tcp://docker-daemon1.example.com": {
     "noProxy": "*.internal.example.net"
   }
 }
}
```

```shell
$ docker build --build-arg HTTP_PROXY="http://proxy.example.com:3128" .
$ docker run --env HTTP_PROXY="http://proxy.example.com:3128" redis
```

## Docker CLI 的 OpenTelemetry 支持
### 搭建 OpenTelemetry 后端环境
创建配置文件：在本地目录创建三个文件。
* compose.yaml：定义服务。
* otelcol.yml：配置 OTel 收集器。
* prom.yml：配置 Prometheus。
启动服务：
```shell
docker compose up -d
```
此时，以下服务在后台运行：
* OpenTelemetry 收集器：在 localhost:4317 监听，准备接收 Docker CLI 发送的数据。
* Prometheus：在 localhost:9091 提供界面，并定期从收集器抓取数据。

### 配置 Docker CLI
告诉 Docker CLI 将遥测数据发送到哪里。
```shell
# 在终端中设置环境变量（当前会话有效）
export DOCKER_CLI_OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

### 执行 Docker 命令（生成数据）
现在，任何在此终端中执行的 docker 命令都会自动生成指标并发送到收集器。
```shell
docker version
```
### 在 Prometheus 中查看结果
打开浏览器，访问 http://localhost:9091（Prometheus 的图形界面）。

在 “Graph” 页面的查询框中，输入文档中提到的指标名称：command_time_milliseconds_total。

# Security
在审视 Docker 安全时，有四个主要方面需要考虑：

* kernel的固有安全及其对namespaces和cgroups的支持
* Docker daemon 本身的攻击面
* 容器配置文件的漏洞，无论是默认配置还是用户自定义配置
* kernel的"固有的"安全特性以及它们对容器产生的影响。

## Kernel namespaces
当您使用 docker run 启动容器时，Docker 在后台会为容器创建一组命名空间和控制组。

命名空间提供了第层隔离，在容器内运行的进程不可见，更不用说影响在另一个容器中或在主机系统中运行的其他进程。不同的容器使用自己独立的网络，通过docker提供的network服务互相通信

## cgroup
cgroup是容器化技术的基石之一，负责对CPU、内存、磁盘IO的资源限制。可以确保单个容器不会因耗尽其中某种资源而导致系统崩溃。
```shell
sudo mkdir /sys/fs/cgroup/mydemo
echo 100M | sudo tee /sys/fs/cgroup/mydemo/memory.max
echo $$ | sudo tee /sys/fs/cgroup/mydemo/cgroup.procs
```

## Docker daemon
守护进程会使用root启动，除非使用rootless模式。
守护进程容易在被docker API调用时受影响。
守护进程在docker load 或者 docker pull 过程中，也存在一定的安全风险。
如果您在服务器上运行 Docker，建议在该服务器上专门运行 Docker，并将所有其他服务移至由 Docker 控制的容器中。

## Linux kernel 能力
容器内的root用户并非完全的root行为，某些能力是被限制的，例如：
* CAP_SYS_ADMIN： 这是非常强大的能力，近似于 root。默认被拒绝，意味着容器内的“root”不能执行挂载文件系统、交换区管理等操作。
* CAP_NET_RAW： 允许创建原始套接字，可用于进行网络欺骗攻击（如伪造 IP 包）。默认被拒绝。
* CAP_SYS_MODULE： 允许加载和卸载内核模块。这是极其危险的操作，默认被拒绝。
* CAP_SYS_CHROOT, CAP_MKNOD 等文件系统相关能力也受到严格限制。

即使应用程序存在漏洞，攻击者利用漏洞在容器内获取了root权限，其能做的事情也有限。并且攻击者很难从“容器内的 root”权限升级到“主机上的 root”权限。

## Docker签名认证
todo：https://docs.docker.com/engine/security/trust/


## rootless模式
Rootless 模式是 Docker 的一种运行方式，允许普通用户（非 root 用户）在没有管理员权限的情况下运行 Docker 守护进程和容器。

Rootless 模式为 Docker 提供了更好的安全性和更细粒度的权限控制，特别适合在多用户环境和安全要求较高的场景中使用。

### Rootless 模式的工作原理
Ubuntu 24.04 及更高版本默认启用了受限的无特权用户命名空间，必须手动为 rootlesskit 添加 AppArmor 配置文件。


Rootless Docker 使用以下技术来实现非特权运行：
* 用户命名空间（User Namespaces）：将容器内的 root 用户映射到主机上的非 root 用户
* RootlessKit：提供网络和端口转发等功能的工具包
* Slirp4netns：用户级别的网络栈实现

### 安装
```shell
# 安装必要的依赖
sudo apt-get install uidmap dbus-user-session
# 下载并安装
curl -fsSL https://get.docker.com/rootless | sh
# 设置环境变量
export PATH=/home/$(whoami)/bin:$PATH
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock

# 作为普通用户启用并配置"用户级"的 Docker 守护进程
systemctl --user start docker
systemctl --user enable docker
sudo loginctl enable-linger $(whoami) 

```

### 几个需要重点提醒的目录和环境变量
* 必须设置以下环境变量：$HOME, $XDG_RUNTIME_DIR
* /run/user/$UID/docker.sock
* ~/.local/share/docker
* ~/.config/docker/


### rootless的dind
```shell
# 无根模式的dind，此处即使使用了特权模式，但由于是基于无根模式，更加安全
docker run -d --name dind-rootless --privileged docker:25.0-dind-rootless
```

### 优势
* 容器逃逸时只能获得普通用户权限
* 无法修改系统级配置
* 无法访问其他用户的文件

## Docker的AppArmor
AppArmor（Application Armor）是一种 Linux 安全模块，它通过限制应用程序的权限来增强系统安全性。

主要规则：
```shell
# 允许读取大多数系统文件
/usr/** r,
/bin/** r,
/lib/** r,

# 限制写入敏感目录
deny /bin/** wl,
deny /boot/** wl,
deny /dev/** wl,
deny /etc/** wl,
deny /home/** wl,
deny /lib/** wl,
deny /proc/** wl,
deny /sys/** wl,
deny /usr/** wl,

# 允许基本的网络通信
network inet tcp,
network inet udp,
network inet icmp,

# 禁止原始套接字和包操作
deny network raw,
deny network packet,

# 限制 ptrace 等调试操作
deny ptrace,

# 限制信号发送给其他进程
signal (receive) peer=unconfined,
signal (send) peer=docker-default,

# 允许必要的基础能力
capability chown,
capability dac_override,
capability setuid,
capability setgid,
capability net_bind_service,

# 拒绝危险的能力
deny capability sys_module,    # 加载内核模块
deny capability sys_admin,     # 系统管理权限

```

```shell
# 查看已加载的 AppArmor 配置文件
sudo aa-status | grep docker-default

# 查看具体的配置文件内容
sudo cat /sys/kernel/security/apparmor/profiles | grep docker-default

# 使用自定义配置文件运行容器
docker run --security-opt "apparmor=my-custom-profile" nginx

# 完全禁用 AppArmor
docker run --security-opt "apparmor=unconfined" nginx
```

