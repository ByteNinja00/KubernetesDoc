# 二进制安装
二进制安装kubernetes集群，将整个集群的安装从自动化安装工具中解耦出来，每个组件都使用二进制文件独立安装启动，了解自动化工具如：kubeadm 安装隐藏的细节。更方便了解整个集群安装和启动的过程。
## 1. 集群主机清单
|HostName|IPv4 Address|节点角色|硬件资源|集群组件|开放端口|
|:-------|:-----------|:------|---------|:------|:-------|
|k8s-master|192.168.13.10|控制平面|2核2G|kube-apiserver、kube-controller-manager、kube-scheduler、etcd|6443、2379-2380、10250、10259、10257|
|k8s-nodes|192.168.13.11|工作节点|2核2G|kubelet、kube-proxy|10250、30000-32767、10256|
## 2. 集群环境检查
在安装kubeadm之前，需要对集群环境进行一些检查，以便初始化集群当中不会因为未满足集群安装要求而中断或告警。
关于集群主机内核调优的sysctl参数，取决于具体恰当与否，官网[关于systctl参数说明。](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/sysctl-cluster/)
### 2.1. 检查集群主机MAC地址
> [!IMPORTANT]
> 如果是用的虚拟机克隆出来的主机，有可能会有MAC地址重复，如：KVM虚拟机配置文件复制进行克隆的就会发生这种情况。
```
ip addr show
```
### 2.2. 检查主机UUID
> [!IMPORTANT]
> 同样如此如果是因为复制虚拟机配置文件克隆出来的主机UUID也会重复。
```
sudo cat /sys/class/dmi/id/product_uuid
```
### 2.3. 关闭交换分区
虽然官方允计使用交换分区，但是交换分区是把磁盘存储的一部份空间，当作内存空间交换。CPU在进行数据访问时性能肯定大大下降，所以最好关闭。
- **查看是否己挂载SWAP分区**
```
sudo swapon --show
```
- **临时关闭**
```
sudo swapoff -a
```
- **注释分区表己挂载的SWAP分区**
```
#/swap.img none swap sw 0 0
```
### 2.4 设置主机名
- **控制平面主机**
```
sudo hostnamectl set-hostname k8s-master
```
- **工作节点主机**
```
sudo hostnamectl set-hostname k8s-node
```
### 2.5. 设置网络IP地址
配置文件地址: `/etc/netplan/50-cloud-init.yaml`
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
> [!NOTE]
> 以上配置为k8s-master主机IP，不同控制平面主机设置相应网络IP地址。