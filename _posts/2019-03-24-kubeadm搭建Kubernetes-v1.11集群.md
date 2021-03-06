---
layout:     post
title:      kubeadm安装 Kubernetes v1.11 集群
subtitle:   kubeadm 快速搭建 Kubernetes v1.11 集群
date:       2019-03-24
author:     jianweixs
header-img: img/post-bg-ios9-web.jpg
catalog: 	 true
tags:
    - kubernetes
    - kubeadm
    - 容器
---
# kubeadm安装高可用集群

kubeadm作为kubernetes主推的部署工具。



## 资源情况

| Role          | IP             |
| ------------- | -------------- |
| master & node | 10.143.253.213 |
| master & node | 10.143.253.214 |
| master & node | 10.143.253.215 |



## 初始化系统

所有机器都需要初始化容器执行引擎（ docker）和 kubelet。这是因为 kubeadm 依赖 kubelet 来启动 Master 组件，比如 kube-apiserver、kube-manager-controller、kube-scheduler、kube-proxy 等。

## 安装 master

在初始化 master 时，只需要执行 kubeadm init 命令即可，比如

```
kubeadm init --config=kube-config.yaml
```

这个命令会自动

- 系统状态检查
- 生成 token
- 生成自签名 CA 和 client 端证书
- 生成 kubeconfig 用于 kubelet 连接 API server
- 为 Master 组件生成 Static Pod manifests，并放到 `/etc/kubernetes/manifests` 目录中
- 配置 RBAC 并设置 Master node 只运行控制平面组件
- 创建附加服务，比如 kube-proxy 和 core-dns

## 一、前期准备

安装版本：

​	kubernetes v1.11.0

配置yum源：

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

安装用到的镜像名称，需要从gcr.io上pull下来，上传至私有镜像仓库

```shell
dockerhub-new.caiwu.corp:15000/kube-apiserver-amd64:v1.11.0
dockerhub-new.caiwu.corp:15000/kube-controller-manager-amd64:v1.11.0
dockerhub-new.caiwu.corp:15000/kube-scheduler-amd64:v1.11.0
dockerhub-new.caiwu.corp:15000/kube-proxy-amd64:v1.11.0
```

同步resolv.conf

```
$ cat /etc/resolv.conf
# Generated by NetworkManager
nameserver 10.100.10.254
search 213.caiwu.corp
```

此处配置自己内网的DNS。

Kubernetes 1.8开始要求关闭系统的Swap，如果不关闭，默认配置下kubelet将无法启动。可以通过kubelet的启动参数`--fail-swap-on=false`更改这个限制。

> 关闭系统的Swap方法如下:
>
> ```
>   swapoff -a
> ```
>
> 修改 /etc/fstab 文件，注释掉 SWAP 的自动挂载，使用`free -m`确认swap已经关闭。 swappiness参数调整，修改/etc/sysctl.d/k8s.conf添加下面一行：
>
> ```
>   vm.swappiness=0
> ```
>
> 执行`sysctl -p /etc/sysctl.d/k8s.conf`使修改生效。

配置sysctl.conf

```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

## 二、master1节点部署

安装用到的组件

```
yum install kubelet-1.11.0 kubectl-1.11.0 kubernetes-cni kubeadm-1.11.0
```

kubeadm config.yaml配置

```shell
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.0
apiServerCertSANs:
- "zc.k8s.caiwu.corp"
api:
  controlPlaneEndpoint: "zc.k8s.caiwu.corp:6443"
auditPolicy:
  logDir: /var/log/kubernetes/audit
  logMaxAge: 2
  path: ""
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
etcd:
  local:
    dataDir: /var/lib/etcd
    image: ""
imageRepository: dockerhub-new.caiwu.corp:15000
etcd:
  local:
    extraArgs:
      listen-client-urls: "https://127.0.0.1:2379,https://10.143.253.213:2379"
      advertise-client-urls: "https://10.143.253.213:2379"
      listen-peer-urls: "https://10.143.253.213:2380"
      initial-advertise-peer-urls: "https://10.143.253.213:2380"
      initial-cluster: "k8s-213=https://10.143.253.213:2380"
    serverCertSANs:
      - k8s-213
      - 10.143.253.213
    peerCertSANs:
      - k8s-213
      - 10.143.253.213
kubeProxy:
  config:
    bindAddress: 0.0.0.0
    clientConnection:
      acceptContentTypes: ""
      burst: 10
      contentType: application/vnd.kubernetes.protobuf
      kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
      qps: 5
    clusterCIDR: ""
    configSyncPeriod: 15m0s
    conntrack:
      max: null
      maxPerCore: 32768
      min: 131072
      tcpCloseWaitTimeout: 1h0m0s
      tcpEstablishedTimeout: 24h0m0s
    enableProfiling: false
    healthzBindAddress: 0.0.0.0:10256
    hostnameOverride: ""
    iptables:
      masqueradeAll: false
      masqueradeBit: 14
      minSyncPeriod: 0s
      syncPeriod: 30s
    ipvs:
      ExcludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      syncPeriod: 30s
    metricsBindAddress: 127.0.0.1:10249
    mode: ""
    nodePortAddresses: null
    oomScoreAdj: -999
    portRange: ""
    resourceContainer: /kube-proxy
    udpIdleTimeout: 250ms
kubeletConfiguration:
  baseConfig:
    address: 0.0.0.0
    authentication:
      anonymous:
        enabled: false
      webhook:
        cacheTTL: 2m0s
        enabled: true
      x509:
        clientCAFile: /etc/kubernetes/pki/ca.crt
    authorization:
      mode: Webhook
      webhook:
        cacheAuthorizedTTL: 5m0s
        cacheUnauthorizedTTL: 30s
    cgroupDriver: cgroupfs
    cgroupsPerQOS: true
    clusterDNS:
    - 10.96.0.10
    clusterDomain: cluster.local
    containerLogMaxFiles: 5
    containerLogMaxSize: 10Mi
    contentType: application/vnd.kubernetes.protobuf
    cpuCFSQuota: true
    cpuManagerPolicy: none
    cpuManagerReconcilePeriod: 10s
    enableControllerAttachDetach: true
    enableDebuggingHandlers: true
    enforceNodeAllocatable:
    - pods
    eventBurst: 10
    eventRecordQPS: 5
    evictionHard:
      imagefs.available: 15%
      memory.available: 100Mi
      nodefs.available: 10%
      nodefs.inodesFree: 5%
    evictionPressureTransitionPeriod: 5m0s
    failSwapOn: true
    fileCheckFrequency: 20s
    hairpinMode: promiscuous-bridge
    healthzBindAddress: 127.0.0.1
    healthzPort: 10248
    httpCheckFrequency: 20s
    imageGCHighThresholdPercent: 85
    imageGCLowThresholdPercent: 80
    imageMinimumGCAge: 2m0s
    iptablesDropBit: 15
    iptablesMasqueradeBit: 14
    kubeAPIBurst: 10
    kubeAPIQPS: 5
    makeIPTablesUtilChains: true
    maxOpenFiles: 1000000
    maxPods: 110
    nodeStatusUpdateFrequency: 10s
    oomScoreAdj: -999
    podPidsLimit: -1
    port: 10250
    registryBurst: 10
    registryPullQPS: 5
    resolvConf: /etc/resolv.conf
    rotateCertificates: true
    runtimeRequestTimeout: 2m0s
    serializeImagePulls: true
    staticPodPath: /etc/kubernetes/manifests
    streamingConnectionIdleTimeout: 4h0m0s
    syncFrequency: 1m0s
    volumeStatsAggPeriod: 1m0s
networking:
  dnsDomain: cluster.local
  podSubnet: "10.251.0.0/16"
  serviceSubnet: 10.96.0.0/12
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-213
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
unifiedControlPlaneImage: ""
```

kubelet配置

```
###
# kubernetes kubelet (minion) config

# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=10.143.253.14"

# The port for the info server to serve on
# KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=10.143.253.14"

# location of the api-server
KUBELET_API_SERVER="--api-servers=http://10.100.139.250:80"

# pod infrastructure container
#KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"

# Add your own!
#KUBELET_ARGS="--register-node=true --host-network-sources=* --cluster_dns=10.244.0.100 --cgroup-driver=systemd --cluster_domain=cluster.local --log-dir=/var/log/kubernetes"
KUBELET_ARGS="--register-node=true --host-network-sources=* --cluster_dns=10.244.0.100 --cgroup-driver=systemd --cluster_domain=cluster.local --log-dir=/var/log/kubernetes --kube-reserved=cpu=200m,memory=1G --eviction-hard=memory.available<1Gi,nodefs.available<1Gi,imagefs.available<30Gi"
```

执行kubeadm init命令初始化集群，如果kubeadm-config.yaml不在当前路径下，需要指定 kubeadm-config.yml 的路径 ：

`kubeadm init --config kubeadm-config.yaml`

将所需文件复制到其他控制平面节点

您运行时创建了以下证书和其他所需文件`kubeadm init`。将这些文件复制到其他控制平面节点：

- `/etc/kubernetes/pki/ca.crt`
- `/etc/kubernetes/pki/ca.key`
- `/etc/kubernetes/pki/sa.key`
- `/etc/kubernetes/pki/sa.pub`
- `/etc/kubernetes/pki/front-proxy-ca.crt`
- `/etc/kubernetes/pki/front-proxy-ca.key`
- `/etc/kubernetes/pki/etcd/ca.crt`
- `/etc/kubernetes/pki/etcd/ca.key`

将admin kubeconfig复制到其他控制平面节点：

- `/etc/kubernetes/admin.conf`

示例脚本：

```
USER=root # customizable
CONTROL_PLANE_IPS="10.143.253.214 10.143.253.215"
for host in ${CONTROL_PLANE_IPS}; do
    scp /etc/kubernetes/pki/ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.pub "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/etcd/ca.crt "${USER}"@$host:etcd-ca.crt
    scp /etc/kubernetes/pki/etcd/ca.key "${USER}"@$host:etcd-ca.key
    scp /etc/kubernetes/admin.conf "${USER}"@$host:
done
```

## 二、master2节点部署

创建第二个不同的`kubeadm-config.yaml`模板文件

```
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.0
apiServerCertSANs:
- "zc.k8s.caiwu.corp"
api:
  controlPlaneEndpoint: "zc.k8s.caiwu.corp:6443"
auditPolicy:
  logDir: /var/log/kubernetes/audit
  logMaxAge: 2
  path: ""
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
etcd:
  local:
    dataDir: /var/lib/etcd
    image: ""
imageRepository: dockerhub-new.caiwu.corp:15000
etcd:
  local:
    extraArgs:
      listen-client-urls: "https://127.0.0.1:2379,https://10.143.253.214:2379"
      advertise-client-urls: "https://10.143.253.214:2379"
      listen-peer-urls: "https://10.143.253.214:2380"
      initial-advertise-peer-urls: "https://10.143.253.214:2380"
      initial-cluster: "k8s-213=https://10.143.253.213:2380,k8s-214=https://10.143.253.214:2380"
      initial-cluster-state: existing
    serverCertSANs:
      - k8s-214
      - 10.143.253.214
    peerCertSANs:
      - k8s-214
      - 10.143.253.214
kubeProxy:
  config:
    bindAddress: 0.0.0.0
    clientConnection:
      acceptContentTypes: ""
      burst: 10
      contentType: application/vnd.kubernetes.protobuf
      kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
      qps: 5
    clusterCIDR: ""
    configSyncPeriod: 15m0s
    conntrack:
      max: null
      maxPerCore: 32768
      min: 131072
      tcpCloseWaitTimeout: 1h0m0s
      tcpEstablishedTimeout: 24h0m0s
    enableProfiling: false
    healthzBindAddress: 0.0.0.0:10256
    hostnameOverride: ""
    iptables:
      masqueradeAll: false
      masqueradeBit: 14
      minSyncPeriod: 0s
      syncPeriod: 30s
    ipvs:
      ExcludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      syncPeriod: 30s
    metricsBindAddress: 127.0.0.1:10249
    mode: ""
    nodePortAddresses: null
    oomScoreAdj: -999
    portRange: ""
    resourceContainer: /kube-proxy
    udpIdleTimeout: 250ms
kubeletConfiguration:
  baseConfig:
    address: 0.0.0.0
    authentication:
      anonymous:
        enabled: false
      webhook:
        cacheTTL: 2m0s
        enabled: true
      x509:
        clientCAFile: /etc/kubernetes/pki/ca.crt
    authorization:
      mode: Webhook
      webhook:
        cacheAuthorizedTTL: 5m0s
        cacheUnauthorizedTTL: 30s
    cgroupDriver: cgroupfs
    cgroupsPerQOS: true
    clusterDNS:
    - 10.96.0.10
    clusterDomain: cluster.local
    containerLogMaxFiles: 5
    containerLogMaxSize: 10Mi
    contentType: application/vnd.kubernetes.protobuf
    cpuCFSQuota: true
    cpuManagerPolicy: none
    cpuManagerReconcilePeriod: 10s
    enableControllerAttachDetach: true
    enableDebuggingHandlers: true
    enforceNodeAllocatable:
    - pods
    eventBurst: 10
    eventRecordQPS: 5
    evictionHard:
      imagefs.available: 15%
      memory.available: 100Mi
      nodefs.available: 10%
      nodefs.inodesFree: 5%
    evictionPressureTransitionPeriod: 5m0s
    failSwapOn: true
    fileCheckFrequency: 20s
    hairpinMode: promiscuous-bridge
    healthzBindAddress: 127.0.0.1
    healthzPort: 10248
    httpCheckFrequency: 20s
    imageGCHighThresholdPercent: 85
    imageGCLowThresholdPercent: 80
    imageMinimumGCAge: 2m0s
    iptablesDropBit: 15
    iptablesMasqueradeBit: 14
    kubeAPIBurst: 10
    kubeAPIQPS: 5
    makeIPTablesUtilChains: true
    maxOpenFiles: 1000000
    maxPods: 110
    nodeStatusUpdateFrequency: 10s
    oomScoreAdj: -999
    podPidsLimit: -1
    port: 10250
    registryBurst: 10
    registryPullQPS: 5
    resolvConf: /etc/resolv.conf
    rotateCertificates: true
    runtimeRequestTimeout: 2m0s
    serializeImagePulls: true
    staticPodPath: /etc/kubernetes/manifests
    streamingConnectionIdleTimeout: 4h0m0s
    syncFrequency: 1m0s
    volumeStatsAggPeriod: 1m0s
networking:
  dnsDomain: cluster.local
  podSubnet: "10.251.0.0/16"
  serviceSubnet: 10.96.0.0/12
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-214
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
unifiedControlPlaneImage: ""
```

1.将复制的文件移动至配置文件所在的目录

```
$ USER=root # customizable
$ mkdir -p /etc/kubernetes/pki/etcd
$ mv /home/${USER}/ca.crt /etc/kubernetes/pki/
$ mv /home/${USER}/ca.key /etc/kubernetes/pki/
$ mv /home/${USER}/sa.pub /etc/kubernetes/pki/
$ mv /home/${USER}/sa.key /etc/kubernetes/pki/
$ mv /home/${USER}/front-proxy-ca.crt /etc/kubernetes/pki/
$ mv /home/${USER}/front-proxy-ca.key /etc/kubernetes/pki/
$ mv /home/${USER}/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt
$ mv /home/${USER}/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key
$ mv /home/${USER}/admin.conf /etc/kubernetes/admin.conf
```

2.运行kubeadm阶段命令来引导kubelet：

```shell
$ kubeadm alpha phase certs all --config kubeadm-config.yaml
$ kubeadm alpha phase kubelet config write-to-disk --config kubeadm-config.yaml
$ kubeadm alpha phase kubelet write-env-file --config kubeadm-config.yaml
$ kubeadm alpha phase kubeconfig kubelet --config kubeadm-config.yaml
$ systemctl start kubelet
```

3.运行命令将节点添加到etcd集群：

```shell
$ export CP0_IP=10.143.253.213
$ export CP0_HOSTNAME=k8s-213
$ export CP1_IP=10.142.253.214
$ export CP1_HOSTNAME=k8s-213
$ export KUBECONFIG=/etc/kubernetes/admin.conf
$ kubectl exec -n kube-system etcd-${CP0_HOSTNAME} -- etcdctl --ca-file /etc/kubernetes/pki/etcd/ca.crt --cert-file /etc/kubernetes/pki/etcd/peer.crt --key-file /etc/kubernetes/pki/etcd/peer.key --endpoints=https://${CP0_IP}:2379 member add ${CP1_HOSTNAME} https://${CP1_IP}:2380
$ kubeadm alpha phase etcd local --config kubeadm-config.yaml
```

- 在将节点添加到正在运行的集群之后，以及在将新节点加入etcd集群之前，此命令会导致etcd集群在短时间内不可用。


4.部署控制平面组件并将节点标记为主节点：

  ```shell
$ kubeadm alpha phase kubeconfig all --config kubeadm-config.yaml
$ kubeadm alpha phase controlplane all --config kubeadm-config.yaml
$ kubeadm alpha phase mark-master --config kubeadm-config.yaml
  ```

## 四、master3节点部署

创建第三个不同的`kubeadm-config.yaml`模板文件

```
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.0
apiServerCertSANs:
- "zc.k8s.caiwu.corp"
api:
  controlPlaneEndpoint: "zc.k8s.caiwu.corp:6443"
auditPolicy:
  logDir: /var/log/kubernetes/audit
  logMaxAge: 2
  path: ""
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
etcd:
  local:
    dataDir: /var/lib/etcd
    image: ""
imageRepository: dockerhub-new.caiwu.corp:15000
etcd:
  local:
    extraArgs:
      listen-client-urls: "https://127.0.0.1:2379,https://10.143.253.215:2379"
      advertise-client-urls: "https://10.143.253.215:2379"
      listen-peer-urls: "https://10.143.253.215:2380"
      initial-advertise-peer-urls: "https://10.143.253.213:2380"
      initial-cluster: "k8s-213=https://10.143.253.213:2380,k8s-214=https://10.143.253.214:2380,k8s-215=https://10.143.253.215:2380""
      initial-cluster-state: existing
    serverCertSANs:
      - k8s-215
      - 10.143.253.215
    peerCertSANs:
      - k8s-215
      - 10.143.253.215
kubeProxy:
  config:
    bindAddress: 0.0.0.0
    clientConnection:
      acceptContentTypes: ""
      burst: 10
      contentType: application/vnd.kubernetes.protobuf
      kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
      qps: 5
    clusterCIDR: ""
    configSyncPeriod: 15m0s
    conntrack:
      max: null
      maxPerCore: 32768
      min: 131072
      tcpCloseWaitTimeout: 1h0m0s
      tcpEstablishedTimeout: 24h0m0s
    enableProfiling: false
    healthzBindAddress: 0.0.0.0:10256
    hostnameOverride: ""
    iptables:
      masqueradeAll: false
      masqueradeBit: 14
      minSyncPeriod: 0s
      syncPeriod: 30s
    ipvs:
      ExcludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      syncPeriod: 30s
    metricsBindAddress: 127.0.0.1:10249
    mode: ""
    nodePortAddresses: null
    oomScoreAdj: -999
    portRange: ""
    resourceContainer: /kube-proxy
    udpIdleTimeout: 250ms
kubeletConfiguration:
  baseConfig:
    address: 0.0.0.0
    authentication:
      anonymous:
        enabled: false
      webhook:
        cacheTTL: 2m0s
        enabled: true
      x509:
        clientCAFile: /etc/kubernetes/pki/ca.crt
    authorization:
      mode: Webhook
      webhook:
        cacheAuthorizedTTL: 5m0s
        cacheUnauthorizedTTL: 30s
    cgroupDriver: cgroupfs
    cgroupsPerQOS: true
    clusterDNS:
    - 10.96.0.10
    clusterDomain: cluster.local
    containerLogMaxFiles: 5
    containerLogMaxSize: 10Mi
    contentType: application/vnd.kubernetes.protobuf
    cpuCFSQuota: true
    cpuManagerPolicy: none
    cpuManagerReconcilePeriod: 10s
    enableControllerAttachDetach: true
    enableDebuggingHandlers: true
    enforceNodeAllocatable:
    - pods
    eventBurst: 10
    eventRecordQPS: 5
    evictionHard:
      imagefs.available: 15%
      memory.available: 100Mi
      nodefs.available: 10%
      nodefs.inodesFree: 5%
    evictionPressureTransitionPeriod: 5m0s
    failSwapOn: true
    fileCheckFrequency: 20s
    hairpinMode: promiscuous-bridge
    healthzBindAddress: 127.0.0.1
    healthzPort: 10248
    httpCheckFrequency: 20s
    imageGCHighThresholdPercent: 85
    imageGCLowThresholdPercent: 80
    imageMinimumGCAge: 2m0s
    iptablesDropBit: 15
    iptablesMasqueradeBit: 14
    kubeAPIBurst: 10
    kubeAPIQPS: 5
    makeIPTablesUtilChains: true
    maxOpenFiles: 1000000
    maxPods: 110
    nodeStatusUpdateFrequency: 10s
    oomScoreAdj: -999
    podPidsLimit: -1
    port: 10250
    registryBurst: 10
    registryPullQPS: 5
    resolvConf: /etc/resolv.conf
    rotateCertificates: true
    runtimeRequestTimeout: 2m0s
    serializeImagePulls: true
    staticPodPath: /etc/kubernetes/manifests
    streamingConnectionIdleTimeout: 4h0m0s
    syncFrequency: 1m0s
    volumeStatsAggPeriod: 1m0s
networking:
  dnsDomain: cluster.local
  podSubnet: "10.251.0.0/16"
  serviceSubnet: 10.96.0.0/12
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-215
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
unifiedControlPlaneImage: ""
```

1.将复制的文件移动到正确的位置：

```shell
  $ USER=root # customizable
  $ mkdir -p /etc/kubernetes/pki/etcd
  $ mv /home/${USER}/ca.crt /etc/kubernetes/pki/
  $ mv /home/${USER}/ca.key /etc/kubernetes/pki/
  $ mv /home/${USER}/sa.pub /etc/kubernetes/pki/
  $ mv /home/${USER}/sa.key /etc/kubernetes/pki/
  $ mv /home/${USER}/front-proxy-ca.crt /etc/kubernetes/pki/
  $ mv /home/${USER}/front-proxy-ca.key /etc/kubernetes/pki/
  $ mv /home/${USER}/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt
  $ mv /home/${USER}/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key
  $ mv /home/${USER}/admin.conf /etc/kubernetes/admin.conf
```

2.运行kubeadm阶段命令来引导kubelet：

```shell
  $ kubeadm alpha phase certs all --config kubeadm-config.yaml
  $ kubeadm alpha phase kubelet config write-to-disk --config kubeadm-config.yaml
  $ kubeadm alpha phase kubelet write-env-file --config kubeadm-config.yaml
  $ kubeadm alpha phase kubeconfig kubelet --config kubeadm-config.yaml
  $ systemctl start kubelet
```

3.运行命令将节点添加到etcd集群：

```shell
 $ export CP0_IP=10.143.253.213
 $ export CP0_HOSTNAME=k8s-213
 $ export CP2_IP=10.143.253.215
 $ export CP2_HOSTNAME=k8s-215
 $ export KUBECONFIG=/etc/kubernetes/admin.conf
 $ kubectl exec -n kube-system etcd-${CP0_HOSTNAME} -- etcdctl --ca-file /etc/kubernetes/pki/etcd/ca.crt --cert-file /etc/kubernetes/pki/etcd/peer.crt --key-file /etc/kubernetes/pki/etcd/peer.key --endpoints=https://${CP0_IP}:2379 member add ${CP2_HOSTNAME} https://${CP2_IP}:2380
 $ kubeadm alpha phase etcd local --config kubeadm-config.yaml
```

4.部署控制平面组件并将节点标记为主节点：

```shell
 $ kubeadm alpha phase kubeconfig all --config kubeadm-config.yaml
 $ kubeadm alpha phase controlplane all --config kubeadm-config.yaml
 $ kubeadm alpha phase mark-master --config kubeadm-config.yaml
```

## 五、增加node节点

安装需要的组件

```shell
$ yum install kubelet-1.11.0 kubernetes-cni
```

配置yum源：

```shell
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

同步resolv.conf

```shell
$ cat /etc/resolv.conf
# Generated by NetworkManager
nameserver 10.100.10.254
```

Kubernetes 1.8开始要求关闭系统的Swap，如果不关闭，默认配置下kubelet将无法启动。可以通过kubelet的启动参数`--fail-swap-on=false`更改这个限制。

> 关闭系统的Swap方法如下:
>
> ```
>   swapoff -a
> ```
>
> 修改 /etc/fstab 文件，注释掉 SWAP 的自动挂载，使用`free -m`确认swap已经关闭。 swappiness参数调整，修改/etc/sysctl.d/k8s.conf添加下面一行：
>
> ```
>   vm.swappiness=0
> ```
>
> 执行`sysctl -p /etc/sysctl.d/k8s.conf`使修改生效。

配置sysctl.conf

```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

将该节点加入集群（在master上执行该命令）：

```shell
kubeadm join zc.k8s.caiwu.corp:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:5a0f3c0457f0c042de3ca5663f3c6bff76c8b1fced6b198cad20849ee08ed8c7
```

## 六、安装flannel网络插件

详细文档参见https://github.com/coreos/flannel

```shell
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.251.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-amd64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: amd64
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: dockerhub-new.caiwu.corp:15000/flannel:v0.10.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: dockerhub-new.caiwu.corp:15000/flannel:v0.10.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --etcd-endpoints=http://10.100.139.8:2379,http://10.100.139.9:2379,http://10.100.139.251:2379
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: true
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
```

该文件中替换镜像为私有仓库镜像，

`dockerhub-new.caiwu.corp:15000/flannel:v0.10.0-amd64`

修改网段

```
 net-conf.json: |
    {
      "Network": "10.251.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

指定要连接的etcd集群

--etcd-endpoints=http://10.100.139.8:2379,http://10.100.139.9:2379,http://10.100.139.251:2379

执行创建命令：

`kubectl create -f flannel.yaml`

## 七、部署kubernetes-metrics-server

[Metrics Server](https://github.com/kubernetes-incubator/metrics-server) 是集群范围资源使用数据的聚合器，Metrics Server 从每个节点上的 [Kubelet](https://k8smeetup.github.io/docs/admin/kubelet/) 公开的 Summary API 中采集指标信息。

资源使用情况的度量（如容器的 CPU 和内存使用）可以通过 Metrics API 获取。注意

- Metrics API 只可以查询当前的度量数据，并不保存历史数据
- Metrics API URI 为 `/apis/metrics.k8s.io/`，在 [k8s.io/metrics](https://github.com/kubernetes/metrics) 维护
- 必须部署 `metrics-server` 才能使用该 API，metrics-server 通过调用 Kubelet Summary API 获取数据

部署Metrics Server:

```shell
git clone https://github.com/kubernetes/kube-state-metrics.git
kubectl create -f kube-state-metrics/kubernetes
kubectl get pods -o wide --all-namespaces
```