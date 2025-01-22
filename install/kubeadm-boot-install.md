# kubeadm 安装集群
因为本文目前用于测试学习kubernetes集群，并未为其进行 **HA(High Availability)** 和 **LB(Load Balancing)** 扩展。

## 1. 集群主机资源清单：

|主机名|IPV4地址|构架角色|硬件资源|集群组件|开放端口|
|:------|:-------|:-------|:--------|:-------|:--------|
|k8s-master|10.224.2.10|控制平面|2核2G|kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy|6443、2379-2380、10250、10259、10257|
|k8s-node|10.224.2.11|工作节点|2核2G|kubelet、kube-proxy|10250、30000-32767、10256|

