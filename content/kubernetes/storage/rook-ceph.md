+++
title = "Rook 部署 ceph 集群"
weight = 3
+++

## 背景

[Ceph](https://ceph.com/en/) 是一种分布式存储系统，提供文件、块和对象存储，可部署在大规模生产集群中。

[Rook](https://rook.io/docs/rook/latest-release/Getting-Started/intro/) 是一个开源的云原生存储编排器，为 Ceph 存储提供平台、框架和支持，以便与云原生环境进行原生集成。

Rook 可自动部署和管理 Ceph，以提供自我管理、自我扩展和自我修复的存储服务。 Rook 操作员通过利用 Kubernetes 资源来部署、配置、调配、扩展、升级和监控 Ceph。

介绍来，我们将使用 rook 来部署 ceph 集群。

主要分为两步
1. 第一步，先部署 rook operator
2. 第二步，使用 rook 提供的 cluster(CRD) 来部署 ceph 集群。

> 简单来说，rook operator 定义了一些列 CRD，这些 CRD 描述了 Ceph 集群的最终部署状态，而 operator 的作用就是按照这种状态去帮我们部署 ceph 集群。因此，我们的主要运维工作就是部署和使用 rook operator。

## 前置准备

- 首先需要部署 kubernetes 集群，参考[KUBESPRAY 部署 K8S]({{% ref "kubespray-deploy-k8s" %}} "deply")
- 节点上要有未格式化的原始块存储设备，详细见 [Prerequisites](https://rook.io/docs/rook/latest-release/Getting-Started/Prerequisites/prerequisites/#ceph-prerequisites)，这里我们使用的是 LVM 逻辑卷。
  - 原始设备(Raw devices)（无分区或格式化文件系统）
  - 原始分区(Raw partitions)（无格式化文件系统）
  - LVM逻辑卷（无格式化文件系统）
  - 以块模式(block mode)在存储类(storage class)中提供的持久卷。

因为 LVM 逻辑卷可以动态调整分区大小，以下我们都使用 LVM 逻辑卷。

系统环境如下

| 节点      | 存储节点 | 磁盘 | 
| ----------- | ----------- | ----------- |
| kube-node-119      | 否       | - |
| kube-node-103   | 是        | /dev/sda |
| kube-node-105   | 是        | /dev/sda |
| kube-node-106   | 是        | /dev/sda |
| kube-node-120   | 否        | - |

因此我们需要给存储节点节点打 label

```shell
kubectl label nodes kube-node-103 node-role.kubernetes.io/ceph=ceph
kubectl label nodes kube-node-105 node-role.kubernetes.io/ceph=ceph
kubectl label nodes kube-node-106 node-role.kubernetes.io/ceph=ceph
```

## 创建 LVM

接下来，我们将为存储节点创建 lvm。首先使用 fdisk 创建磁盘分区

```shell
sudo fdisk /dev/sda

# 输入 d 删除分区

# 输入 n 创建分区

# 输入 t 改变类型
# 输入 8e 代表 lvm 的分区代码
```

### 创建 pv

```shell
sudo pvcreate /dev/sda1

# 显示 pv
sudo pvdisplay
```

### 创建 vg

```shell
sudo vgcreate ceph /dev/sda1

sudo vgdisplay
```


### 创建 lvm

先申请 1T 空间
```shell
sudo lvcreate -L 953861 -n osd ceph

sudo lvdispaly
```

## 部署 rook operator

这里我们使用 helm 部署 rook operator

```shell
# 为了避免 helm 网络影响，我们这里先下载 helm 包到本地
# 下载到本地的另一个好处是我们可以直接在本地查看默认的 values.yaml 和 template，便于测试
helm repo add rook-release https://charts.rook.io/release
helm pull rook-release/rook-ceph
  
helm upgrade --install \
    --create-namespace \
    --namespace rook-ceph \
    rook-ceph rook-release/rook-ceph \
    -f values.yaml
```

其中， `values.yaml` 主要修改以下部分, 主要是添加 node affinity，使 ceph 组件调度到存储节点。其中，节点label使用我们前面为存储节点打的 label。如果我们将存储节点只做存储，不做业务节点，那么还需要加上 Taint 和 Toleration。

```yaml
# Settings for whether to disable the drivers or other daemons if they are not
# needed
csi:

  # -- The node labels for affinity of the CSI provisioner deployment [^1]
  provisionerNodeAffinity: #key1=value1,value2; key2=value3
  # Set pluginTolerations and pluginNodeAffinity for plugin daemonset pods.
  # The CSI plugins need to be started on all the nodes where the clients need to mount the storage.
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node-role.kubernetes.io/ceph
              operator: In
              values:
                - ceph

  # -- The node labels for affinity of the CephCSI RBD plugin DaemonSet [^1]
  pluginNodeAffinity: # key1=value1,value2; key2=value3
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node-role.kubernetes.io/ceph
              operator: In
              values:
                - ceph

discover:
  #   - key: key
  #     operator: Exists
  #     effect: NoSchedule
  # -- The node labels for affinity of `discover-agent` [^1]
  nodeAffinity:
    nodeSelectorTerms:
      - matchExpressions:
          - key: node-role.kubernetes.io/ceph
            operator: In
            values:
              - ceph

nodeSelector:
  node-role.kubernetes.io/ceph: ceph
```

### 卸载 rook operator

```shell
helm delete --namespace rook-ceph rook-ceph-cluster
```

## 部署 cluster

上一步已经部署好 rook operator 了。因此接下来只需要部署 ceph 相关的 CR，那么 rook operator 就可以帮助我们部署 ceph 集群了。

这里我们使用 kubectl 的方式部署， 首先我们下载对应版本的配置

```shell
git clone --single-branch --branch v1.14.9 https://github.com/rook/rook.git
cd rook/deploy/examples
```

接下来，我们修改一下配置

```yaml
  # 这里我们将dashboard 的 ssl 关闭
  dashboard:
    enabled: true
    # serve the dashboard under a subpath (useful when you are accessing the dashboard via a reverse proxy)
    # urlPrefix: /ceph-dashboard
    # serve the dashboard at the given port.
    # port: 8443
    # serve the dashboard using SSL
    ssl: false
  
  # 添加 nodeSelector，仅部署到存储节点
  placement:
    all:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: node-role.kubernetes.io/ceph
              operator: In
              values:
              - ceph
  
  # 指定存储节点和对应的存储路径
  storage: # cluster level storage configuration and selection
    useAllNodes: false
    useAllDevices: false
    nodes:
      - name: kube-node-103
        devices: # specific devices to use for storage can be specified for each node
        - name: /dev/ceph/osd
      - name: kube-node-105
        devices: # specific devices to use for storage can be specified for each node
        - name: /dev/ceph/osd
      - name: kube-node-106
        devices: # specific devices to use for storage can be specified for each node
        - name: /dev/ceph/osd
```

最后使用 kubeclt 部署就可以了

```shell
kubectl apply -f deploy/examples/cluster.yaml
```

## 访问 dashboard

由于以上方式创建的 dashboard 的 Service 是 ClusterIP 模式，因此我们无法直接访问，这里我们创建一个 NodePort 的 Service，使用 Nodeport 的方式访问。

首先创建一个 service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
  name: rook-ceph-mgr-dashboard-nodeport
  namespace: rook-ceph
spec:
  ports:
  - name: http-dashboard
    port: 7000
    protocol: TCP
    targetPort: 7000
  selector:
    app: rook-ceph-mgr
    mgr_role: active
    rook_cluster: rook-ceph
  sessionAffinity: None
  type: NodePort
```

部署后，我们就可以使用 nodePort 访问 dashboard了。

其中默认的用户名为 `admin`,密码为
```shell
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```