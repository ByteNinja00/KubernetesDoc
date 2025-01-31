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
