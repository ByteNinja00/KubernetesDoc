# 二进制安装Kubernetes集群
二进制安装kubernetes集群，将整个集群的安装从自动化安装工具中解耦出来，每个组件都使用二进制文件独立安装启动，了解自动化工具如：kubeadm 安装隐藏的细节。更方便了解整个集群安装和启动的过程。
## 1. 集群主机清单
|主机名|IPv4地址|架构角色|硬件资源|集群组件|开放端口|
|-----|--------|-------|-------|--------|-------|
|binary-master|10.224.2.21|控制平面|2核2G|kube-apiserver、kube-controller-manager、kube-scheduler|6443、2379-2380、10250、10259、10257|
|binary-nodes|10.224.2.22|工作节点|2核2G|kubelet、kube-proxy|10250、30000-32767、10256|
## 2. 前置准备
[参考](/install/kubeadm-boot-install.md#2-前置准备)
## 3. 安装容器运行时
[参考](/install/kubeadm-boot-install.md#3-安装容器运行时)
## 4. Kubernetes相关组件下载
- [服务端二进制下载](https://dl.k8s.io/v1.32.0/kubernetes-server-linux-amd64.tar.gz)
- [客户端二进制下载](https://dl.k8s.io/v1.32.0/kubernetes-client-linux-amd64.tar.gz)
- [节点端二进制下载](https://dl.k8s.io/v1.32.0/kubernetes-node-linux-amd64.tar.gz)
- [ETCD二进制下载](https://github.com/etcd-io/etcd/releases/download/v3.5.18/etcd-v3.5.18-linux-amd64.tar.gz)
## 5. 集群证书
Kubernetes集群组件通信使用TLS证书通信加密，TLS在TCP/IP协议的传输层进行握手协商。
- **服务端证书**
    - api-server端点(endpoint)服务器端的证书。
    - etcd服务端的证书
    - kubelet服务端的证书
    - 可选的前端代理的服务器证书

- **客户端证书**
    - api-server向每个kubelet服务验证的客户端的证书。
    - api-server向etcd服务验证的客户端证书。
    - kube-controller-manager向api-server服务验证的客户端证书。
    - kube-scheduler向api-server服务验证的客户端证书。
    - kube-proxy向api-server服务验证的客户端证书。
    - 集群管理(kubectl)工具的客户端证书。
    - 可选前端代理客户端证书。

### 5.1. cfssl证书生成工具
cfssl证书生成工具是CLoudFlare团队github项目，[官方GitHub项目地址](https://github.com/cloudflare/cfssl)。

两种方式获取：
1. 从[官方GitHub仓库下载预发布的二进制文件](https://github.com/cloudflare/cfssl/releases)。
2. 从源代码手动编译安装。

本文以手动编译安装生成二进制可执行文件，进行集群证书的签发。
- 下载[GO编译](https://go.dev/dl/go1.23.5.linux-amd64.tar.gz)
```
 sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.23.5.linux-amd64.tar.gz
```
- 增加环境变量
```
export PATH=$PATH:/usr/local/go/bin
```
- 克隆cfssl仓库
```
git -c http.proxy=http://192.168.10.4:10808 clone https://github.com/cloudflare/cfssl.git
```
> [!NOTE]
> -c 选项是指定代理服务器，因国内网络有时无法访问GitHub，需要使用网络代理。