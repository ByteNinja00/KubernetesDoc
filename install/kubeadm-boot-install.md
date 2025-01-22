# kubeadm 安装集群
因为本文目前用于测试学习kubernetes集群，并未为其进行 **HA(High Availability)** 和 **LB(Load Balancing)** 扩展。

## 1. 集群主机资源清单：

|主机名|IPV4地址|构架角色|硬件资源|集群组件|开放端口|
|:------|:-------|:-------|:--------|:-------|:--------|
|k8s-master|10.224.2.10|控制平面|2核2G|kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy|6443、2379-2380、10250、10259、10257|
|k8s-node|10.224.2.11|工作节点|2核2G|kubelet、kube-proxy|10250、30000-32767、10256|

## 2. 前置准备
在安装kubeadm之前，需要对集群环境进行一些检查，以便初始化集群当中不会因为未满足集群安装要求而中断或告警。

关于集群主机内核调优的sysctl参数，取决于具体恰当与否，官网 **[关于systctl参数说明](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/sysctl-cluster/)**。

### 检查MAC集群主机MAC地址
如果是用的虚拟机克隆出来的主机，有可能会有MAC地址重复，如：KVM虚拟机配置文件复制进行克隆的就会发生这种情况。
```
ip addr show
```
### 检查主机UUID
同样如此如果是因为复制虚拟机配置文件克隆出来的主机UUID也会重复。
```
sudo cat /sys/class/dmi/id/product_uuid
```
