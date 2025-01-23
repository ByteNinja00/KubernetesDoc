# kubeadm 安装集群
因为本文目前用于测试学习kubernetes集群，并未为其进行 **HA(High Availability)** 和 **LB(Load Balancing)** 扩展。
> **注：本文所有操作均在 Ubuntu 24.04 LTS 发行版之上操作。**

## 1. 集群主机资源清单：

|主机名|IPV4地址|构架角色|硬件资源|集群组件|开放端口|
|:------|:-------|:-------|:--------|:-------|:--------|
|k8s-master|10.224.2.10|控制平面|2核2G|kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy|6443、2379-2380、10250、10259、10257|
|k8s-node|10.224.2.11|工作节点|2核2G|kubelet、kube-proxy|10250、30000-32767、10256|

## 2. 前置准备
在安装kubeadm之前，需要对集群环境进行一些检查，以便初始化集群当中不会因为未满足集群安装要求而中断或告警。

关于集群主机内核调优的sysctl参数，取决于具体恰当与否，官网 **[关于systctl参数说明](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/sysctl-cluster/)**。

### 2.1. 检查MAC集群主机MAC地址
如果是用的虚拟机克隆出来的主机，有可能会有MAC地址重复，如：KVM虚拟机配置文件复制进行克隆的就会发生这种情况。
```
ip addr show
```
### 2.2. 检查主机UUID
同样如此如果是因为复制虚拟机配置文件克隆出来的主机UUID也会重复。
```
sudo cat /sys/class/dmi/id/product_uuid
```
### 2.3. 关闭交换分区
虽然官方允计使用交换分区，但是交换分区是把磁盘存储的一部份空间，当作内存空间交换。CPU在进行数据访问时性能肯定大大下降，所以最好关闭。
- 查看系统当前是否有交换分区挂载：
```
sudo swapon --show
```
- 临时关闭
```
sudo swapoff -a
```
- 注释己挂的分区 `/etc/fstab`
```
#/swap.img none swap sw 0 0
```
### 2.4. 设置主机名
- 控制平面主机
```
sudo hostnamectl set-hostname k8s-master
```
- 工作节点主机
```
sudo hostnamectl set-hostname k8s-node
```
### 2.5. 网络地址设置
以下网络IP地址在 **k8s-master** 主机上配置，在设置工作节点主机网络替换相应规划的IP地址。

配置文件地址：`/etc/netplan/50-cloud-init.yaml`
```
network:
    ethernets:
        ens33:
            dhcp4: false
            addresses: [10.224.2.10/24]
            routes:
              - to: default
                via: 10.224.2.2
            nameservers:
              addresses: [10.224.2.2,223.5.5.5]
    version: 2
```
**更新网络配置**
```
sudo netplan apply
```
### 2.6. 运行依赖软件包
- ipvsadm：kube-prox 网络模式设置ipvs模式，ipvsadm工具可以查看网络规则。
- conntrack：这个包 Containerd 需要。
```
sudo apt-get -y install ipvsadm conntrack lrzsz
```
### 2.7. 配置主机名解析
配置主机名相互解析 `/etc/hosts`
```
sudo tee /etc/hosts <<EOF
10.224.2.10 k8s-master
10.224.2.11 k8s-node
EOF
```
> **控制平面和工作节点都要配置**

## 3. 安装容器运行时
容器运行时(CRI - Container Runtime Interface) 英译容器运行时接口，所有的Pod都运行在容器里，kubelet负责与CRI交互管理Pod的生命周期，启动、停止、监控容器。

Kubernetes支持多个发行版的容器运行时，包括近些年来非常流行的Docker引擎，但是本文使用的容器运行时选择 **Containerd**。
### 3.1. 启用网络转发
*net.ipv4.ip_forward* 负责主机网络在不同网络接口之间转发。
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
```
- **应用生效**
```
sudo sysctl --system
```
- **验证是否生效**
```
sysctl net.ipv4.ip_forward
```
### 3.2. 安装 Containerd
- **[下载](https://github.com/containerd/containerd/releases)：相应版本，并解压。**
```
tar zxvf containerd-2.0.2-linux-amd64.tar.gz && cd bin
```
- **安装**
```
sudo install ./* -o root -g root -m 0755 /usr/local/bin/
```
- **创建Systemd服务文件：`/usr/lib/systemd/system/containerd.service`**
```
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target dbus.service

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```
- **启动服务**
```
sudo systemctl enable container
sudo systemctl enable containerd.service --now
```
- **查看日志**
```
journalctl --unit containerd.service --no-pager
```
### 3.3. 安装runc
runc 是管理容器的命令行工具。
- **[下载](https://github.com/opencontainers/runc/releases)** 安装。
```
sudo install runc.amd64 -o root -g root -m 0755 /usr/local/bin/runc
```
### 3.4. 安装CNI插件
CNI是容器的网络接口插件，用来配置Linux容器中的网络接口。
- **[下载](https://github.com/containernetworking/plugins/releases)** 安装。
```
sudo mkdir -pv /opt/cni/bin
sudo tar zxvf cni-plugins-linux-amd64-v1.6.2.tgz -C /opt/cni/bin/
```
### 3.5. 配置 Containerd
这里有个坑，应该是官网没更新，如果Containerd版本是 **2.x** 及以上。使用的是[V3版本配置文件](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)。
- **首先生成默认配置文件**
```
containerd config default | sudo tee /etc/containerd/config.toml
```
- **修改cgroup驱动为Systemd**
```
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
  SystemdCgroup = true
```
- **修改pause容器**
```
[plugins.'io.containerd.cri.v1.images'.pinned_images]
      sandbox = 'registry.k8s.io/pause:3.10'
```
- **重启服务**
```
sudo systemctl restart containerd.service
```
## 4. 安装kubeadm
以下需要安装这几个可执行文件：
- **kubeadm：** 用于创建集群、重置、清理集群等一些操作的集群外的操作工具。
- **kubelet：** 因为控制平面的组件也是容器运行，所以kubelet由Systemd管理，kubelet是与容器运行时交互的组件。
- **kubectl：** 用于管理集群的客户端命令行工具，API调用细节的封装。

1. **安装kubernetes存储库所需要的包。**
```
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
2. **下载用于 Kubernetes 软件包仓库的公共签名密钥。**
```
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
3. **添加 Kubernetes apt 仓库。**
```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
4. **更新 apt 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本：**
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
## 5. 初始化安装集群
本文是在线安装，安装集群的时候会连接到 `registry.k8s.io` 下载集群所需要的容器镜像，国内网络需要科学上网，或者用aliyun的镜像站点。

这里推荐使用配置文件的方式进行安装，更利于操作审记。

### 5.1. 生成默认配置文件
```
kubeadm config print init-defaults > init-config.yaml
```
### 5.2. 修改配置文件
参考：**[init-config.yaml](/install/yaml/init-config.yaml)**
> **token: abcdef.0123456789abcdef** 这个最好生新一成一个。

token的组成形式为：[a-z0-9]{6}\.[a-z0-9]{16}，熟悉正则表达式看字面上应该容易理解：.号分隔之前6位由小写字母和数字组成，之后由16位小写字母和数字组成。
由以上可以生成令牌token:
```
echo $(openssl rand -hex 3).$(openssl rand -hex 8)

# 249b5c.d67582bf20149bba
```
### 5.3. 创建集群
