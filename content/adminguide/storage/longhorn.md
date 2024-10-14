+++
title = "Longhorn 分布式存储"
weight = 2
+++

## 背景

[longhorn](https://github.com/longhorn/longhorn?tab=readme-ov-file) 是基于 Kubernetes 构建的云原生分布式存储


## 依赖

[Installation Requirements](https://longhorn.io/docs/1.6.1/deploy/install/#installation-requirements)

### ubuntu 安装依赖

由于我们的应用部署在 Ubuntu 上，因此这里介绍在 Ubuntu 上安装依赖的过程。

首先使用依赖检查脚本，我们可以根据脚本的报错安装缺少的依赖。

```shell
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.6.1/scripts/environment_check.sh | bash
```

安装依赖

```shell
sudo apt-get install open-iscsi nfs-common jq
```

由于依赖还需要节点使用 kubectl 并联通 apiserver，因此需要[kubectl 安装](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)及 kubeconfig 配置

### 标记节点

我们给节点增加 label 用于标识那些节点部署 longhorn。

```shell
kubectl label node kube-node-120 role=longhorn
```

### 挂载存储

1. 首先使用 lvm 创建一块 lv
```shell
lvcreate -l 267437 -n longhorn ubuntu-vg
```
其中 `267437` 是 `ubuntu-vg` 中 `pe` 的个数，可以通过 `vgdisplay` 查看空闲 `pe` 个数。

2. 将 lv 格式化
```shell
sudo mkfs.ext4 /dev/ubuntu-vg/longhorn
```

其中，lv 的路径 即 `/dev/ubuntu-vg/longhorn` 可以使用 `lvdisplay` 命令查看

3. 挂载到目录
```shell
sudo mkdir -p /data/longhorn
sudo mount /dev/ubuntu-vg/longhorn /data/longhorn/
```

4. 生成 fstab
```shell
sudo vim /etc/fstab
# mount /dev/mapper/ubuntu--vg-longhorn
/dev/mapper/ubuntu--vg-longhorn /data/longhorn ext4 defaults 0 0
```

其中 lvm 的创建可以参考文档[系统运维|Linux LVM简明教程](https://linux.cn/article-3218-1.html)

## 部署

我们这里使用 helm 部署。

### helm 部署

参考 Longhorn helm 部署的官方文档[Longhorn | Documentation](https://longhorn.io/docs/1.6.1/deploy/install/install-with-helm/)

```bash
helm pull longhorn/longhorn --version 1.6.2

helm upgrade --install longhorn ./longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --version 1.6.1 \
  --values config.yaml
```



{{% notice style="note" %}}
需要提前挂载 `/data/longhorn` 目录
{{% /notice %}}

其中，`config.yaml`的配置如下
```yaml
defaultSettings:
  # 配置默认数据存放地址
  defaultDataPath: /data/longhorn

service:
  ui:
    # -- Service type for Longhorn UI. (Options: "ClusterIP", "NodePort", "LoadBalancer", "Rancher-Proxy")
    type: NodePort
    # -- NodePort port number for Longhorn UI. When unspecified, Longhorn selects a free port between 30000 and 32767.
    nodePort: 30180
  manager:
    # -- Service type for Longhorn Manager.
    type: NodePort
    # -- NodePort port number for Longhorn Manager. When unspecified, Longhorn selects a free port between 30000 and 32767.
    nodePort: 30181

longhornManager:
  nodeSelector:
    role: longhorn


longhornDriver:
  nodeSelector:
    role: longhorn
```

## 页面

http://{your-node-ip}:30180/#/dashboard

### 故障恢复

#### 从 replica 中恢复
- [[FEATURE] Restoring volume from single replica · Issue #469 · longhorn/longhorn (github.com)](https://github.com/longhorn/longhorn/issues/469)
- [Longhorn | Documentation](https://longhorn.io/docs/archives/1.1.1/advanced-resources/data-recovery/export-from-replica/)
- [Longhorn | Documentation](https://longhorn.io/docs/1.6.1/advanced-resources/data-recovery/export-from-replica/)

## 参考

- [17. Kubernetes - 持久化存储（Longhorn） - Dy1an - 博客园 (cnblogs.com)](https://www.cnblogs.com/Dy1an/p/17245825.html)
