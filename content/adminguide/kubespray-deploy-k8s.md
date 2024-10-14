+++
title = "Kubespray 部署 K8s"
weight = 2
+++


## 准备

本文介绍中共使用了四台物理机, 其中一台用户部署 k8s 的机器，另外三台作为 kubernetes 的节点

|  机器   | ip  | 系统 | 用户名 | 备注
|  ----  | ----  | --- | ---- | ---- |
| 部署机  | 任意 | Ubuntu-Server 22.04 | k8s | 部署k8s机器，非k8s集群节点 |
| kube-master-119  | 172.16.0.100 | Ubuntu-Server 22.04  | k8s | 初始化 k8s master 节点 |
| kube-node-120  | 172.16.0.117 | Ubuntu-Server 22.04 | k8s | 初始化 k8s node |
| kube-node-105  | 172.16.0.105 | Ubuntu-Server 22.04 | k8s | 后续加入的 k8s node |
| kube-node-103  | 172.16.0.103 | Ubuntu-Server 22.04 | k8s | 后续加入的 k8s node |
| kube-node-106  | 172.16.0.106 | Ubuntu-Server 22.04 | k8s | 后续加入的 k8s node |
| kube-node-128  | 172.16.0.128 | Ubuntu-Server 22.04 | k8s | 后续加入的 k8s node |


## kubespray 配置

我们将期望的 kubernetes 最终状态写到 kubespray 的配置中（包括节点、网络、运行时等），然后使用 kubespray 工具自动化部署 Kubernetes。

### 自定义kubespary配置

首先我们需要使用 git 拉取 [kubespray](https://github.com/kubernetes-sigs/kubespray) 的默认配置，复制一份配置清单，然后基于默认配置定制配置。

```bash
git clone https://github.com/kubernetes-sigs/kubespray
cd kubespray
# 拷贝集群清单
cp -rfp inventory/sample inventory/mycluster
```

####  inventory

```shell
vim inventory/mycluster/inventory.ini
```

```ini
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
kube-master-119 ansible_ssh_host=172.16.0.119 ansible_ssh_user=k8s ip=172.16.0.119 mask=/24
kube-node-120 ansible_ssh_host=172.16.0.120 ansible_ssh_user=k8s ip=172.16.0.120 mask=/24
kube-node-105 ansible_ssh_host=172.16.0.105 ansible_ssh_user=k8s ip=172.16.0.105 mask=/24
kube-node-103 ansible_ssh_host=172.16.0.103 ansible_ssh_user=k8s ip=172.16.0.103 mask=/24
kube-node-106 ansible_ssh_host=172.16.0.106 ansible_ssh_user=k8s ip=172.16.0.106 mask=/24
kube-node-128 ansible_ssh_host=172.16.0.128 ansible_ssh_user=k8s ip=172.16.0.128 mask=/24

# ## configure a bastion host if your nodes are not directly reachable
# [bastion]
# bastion ansible_host=x.x.x.x ansible_user=some_user

[kube_control_plane]
kube-master-119

[etcd]
kube-master-119

[kube_node]
kube-node-120
kube-node-105
kube-node-103
kube-node-106
kube-node-128

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```

#### 集群网络

```shell
vim inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```

使用高性能的 cilium
```yaml
# Choose network plugin (cilium, calico, kube-ovn, weave or flannel. Use cni for generic cni plugin)
# Can also be set to 'cloud', which lets the cloud provider setup appropriate routing
kube_network_plugin: cilium

```

#### 运行时


使用默认的 containerd

```shell
vim inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```

```yaml
## Container runtime
## docker for docker, crio for cri-o and containerd for containerd.
## Default: containerd
container_manager: containerd
```

容器运行目录(暂不修改)

```shell
vim inventory/mycluster/group_vars/all/containerd.yml
```

```yaml
# containerd_storage_dir: "/var/lib/containerd"
# containerd_state_dir: "/run/containerd"
```

~~配置容器 registry~~

```shell
vim inventory/mycluster/group_vars/all/containerd.yml
```

```yaml
containerd_registries_mirrors:
  - prefix: docker.io
    mirrors:
      - host: http://hub-mirror.c.163.com
        capabilities: ["pull", "resolve"]
        skip_verify: false
```

{{% notice style="note" %}}
当前国内镜像中心处于不可用状态，大家自行解决
{{% /notice %}}


#### 集群配置

自动更新集群证书， 默认一年有效

```shell
vim inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```

```yaml
## Automatically renew K8S control plane certificates on first Monday of each month
auto_renew_certificates: true

# Can be docker_dns, host_resolvconf or none
resolvconf_mode: none
```

打开日志报错

inventory/mycluster/group_vars/all/all.yml
```yaml
## Used to control no_log attribute
unsafe_show_logs: true
```

#### 镜像国内源

[kubespray/docs/mirror.md at master · kubernetes-sigs/kubespray (github.com)](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/mirror.md)

```bash
# Use the download mirror
cp inventory/mycluster/group_vars/all/offline.yml inventory/mycluster/group_vars/all/mirror.yml
sed -i -E '/# .*\{\{ files_repo/s/^# //g' inventory/mycluster/group_vars/all/mirror.yml
tee -a inventory/mycluster/group_vars/all/mirror.yml <<EOF
gcr_image_repo: "gcr.m.daocloud.io"
kube_image_repo: "k8s.m.daocloud.io"
docker_image_repo: "docker.m.daocloud.io"
quay_image_repo: "quay.m.daocloud.io"
github_image_repo: "ghcr.m.daocloud.io"
files_repo: "https://files.m.daocloud.io"
EOF
```

```bash
# Use the download mirror
cp inventory/mycluster/group_vars/all/offline.yml inventory/mycluster/group_vars/all/mirror.yml
sed -i -E '/# .*\{\{ files_repo/s/^# //g' inventory/mycluster/group_vars/all/mirror.yml
tee -a inventory/mycluster/group_vars/all/mirror.yml <<EOF
gcr_image_repo: "gcr.linuzb.xyz"
kube_image_repo: "k8s.linuzb.xyz"
docker_image_repo: "docker.linuzb.xyz"
quay_image_repo: "quay.linuzb.xyz"
github_image_repo: "ghcr.linuzb.xyz"
files_repo: "https://files.m.daocloud.io"
EOF
```

{{% notice style="warning" %}}
当前国内镜像中心处于不可用状态，大家自行解决
{{% /notice %}}

### kubespray docker 部署

我们可以直接使用 kubespray 的 docker 镜像，用于部署 kubernetes。

直接挂载本地目录

```shell
docker run --rm -it --mount type=bind,source="$(pwd)"/inventory/mycluster,dst=/inventory \
  quay.io/kubespray/kubespray:v2.24.1 bash
```

#### 查看集群配置

```bash
ansible-inventory -i /inventory/inventory.ini --list
```

#### 运行部署

```bash
# 要输入两次密码
ansible-playbook -i /inventory/inventory.ini cluster.yml --user k8s --ask-pass --become --ask-become-pass
```

{{% notice style="tip" %}}
在 docker 中挂载 ssh 证书，可以免输入密码。
{{% /notice %}}

#### 扩容节点

后续我们如果需要再增加节点，则需要先更新 `vim inventory/mycluster/inventory.ini` 的配置（增加新节点信息）。

然后重新进入 kubespray docker 容器中。

最后执行以下命令扩容节点。

```bash
ansible-playbook -i /inventory/inventory.ini scale.yml \
  --user=k8s --ask-pass --become --ask-become-pass -b \
   --limit=kube-node-128
```

{{% notice style="tip" %}}
您可以使用--limit=NODE_NAME限制 Kubespray 以避免干扰集群中的其他节点。 在没有使用--limitplaybook会运行facts.yml刷新所有节点的fact缓存。
{{% /notice %}}


#### 缩容节点

如果有节点不再需要了，我们可以将其移除集群，通常步骤是:

- 1.`kubectl cordon NODE` 驱逐节点，确保节点上的服务飘到其它节点上去，参考[安全维护或下线节点](https://imroc.cc/kubernetes/best-practices/ops/securely-maintain-or-offline-node.html)。
- 2.停止节点上的一些 k8s 组件 (kubelet, kube-proxy) 等。
- 3.kubectl delete NODE 将节点移出集群。
- 4.如果节点是虚拟机，并且不需要了，可以直接销毁掉。

前3个步骤，也可以用 kubespray 提供的`remove-node.yml`这个 playbook 来一步到位实现:

```sh
ansible-playbook \
  -i inventory/mycluster/inventory.ini \
  --user=k8s --ask-pass --become --ask-become-pass -b \
  -e "node=kube-node-114" \
  remove-node.yml
```

`-e` 里写要移出的节点名列表，如果您要删除的节点不在线，您应该将`reset_nodes=false`和添加`allow_ungraceful_removal=true`到您的额外变量中

## 后置操作

### 获取kubeconfig

部署完成后，可以在master节点上的 `/root/.kube/config` 路径获取到 kubeconfig

获取到kubeconfig后，将 https://127.0.0.1:6443 修改成kube-apiserver负载均衡器的地址:端口,或者其中一台master。

```bash
alias k='kubectl --kubeconfig /home/k8s/Projects/kube-cert/kubeconfig'
```

### 节点支持 GPU

参考文档 [GPU node]({{% ref "gpu-node" %}})


## 参考文档
- [使用Kubespray 部署Kubernetes集群 | Sunday博客 | Sunday Blog (sundayhk.com)](https://www.sundayhk.com/post/kubespray/)
- [使用kubespray安装kubernetes | kubernetes-notes (huweihuang.com)](https://k8s.huweihuang.com/project/setup/installer/install-k8s-by-kubespray)

也可参考其它方式 [easzlab/kubeasz: 使用Ansible脚本安装K8S集群](https://github.com/easzlab/kubeasz)