+++
title = "Kubernetes 节点 GPU 支持"
weight = 3
+++

## 引言

本文将介绍如何在 Kubernetes 的节点上支持 GPU.

## 部署

接下来将介绍部署过程包含了以下几个部分
- 节点加入 Kubernetes，可以参考[k8s deploy]({{% ref "kubespray-deploy-k8s#" %}})
- 安装 GPU 驱动
- 配置 nvidia-contaienr 运行时
- 安装 nvidia-device-plugin

### 安装 GPU 驱动

可以使用 `ubuntu-drivers` 工具安装 NVIDIA 驱动程序。

运行 `ubuntu-drivers devices` 命令获取 NVIDIA显卡和可用驱动程序的信息。

通常，最好安装推荐的驱动程序，当你决定使用驱动程序版本后请使用 [`apt` 软件包管理器](https://www.myfreax.com/how-to-use-apt-command/)安装英伟达显卡驱动 `nvidia-driver-535-server`

由于我们安装的 Ubuntu-server，这里使用 server 版本

安装完成后，然后在终端运行命令 `sudo reboot` 重启系统。当返回到系统时，您可以在终端再次运行命令 `nvidia-smi` 启动监视工具查看图形卡/显卡的状态。

`nvidia-smi` 命令将显示所用驱动程序的版本以及有关NVIDIA显卡的其它信息。

```shell
sudo apt install nvidia-driver-535-server
sudo reboot
nvidia-smi
```

参考 [如何在 Ubuntu 20.04 安装 Nvidia 驱动程序 | myfreax](https://www.myfreax.com/how-to-nvidia-drivers-on-ubuntu-20-04/)

### 安装 nvidia-container-toolkit

我们需要配置 nvidia container用于节点支持GPU。

首先我们需要安装 nvidia container toolkit

```shell
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update

sudo apt-get install -y nvidia-container-toolkit
```

接下来需要配置containerd 的运行时，执行以下命令

```shell
sudo nvidia-ctk runtime configure --runtime=containerd
```

{{% notice style="note" %}}
注意： 执行完以上命令之后，检查 `plugins."io.containerd.grpc.v1.cri".containerd.default_runtime_name` 的值是否为 `nvidia`, 如果不是。需要在宿主机上修改 containerd 默认的 runtime 为 nvidia
{{% /notice %}}


```bash
sudo vim /etc/containerd/config.toml
```

```toml
[plugins."io.containerd.grpc.v1.cri".containerd]
  default_runtime_name = "nvidia"
```

最后重启 containerd 服务

```shell
sudo systemctl restart containerd
```

如果你使用的不是containerd，或者想查看详细过程请参考以下文档。

- [Installing the NVIDIA Container Toolkit — NVIDIA Container Toolkit 1.14.5 documentation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#configuring-docker)

#### 安装 nvidia-device-plugin

[NVIDIA/k8s-device-plugin: NVIDIA device plugin for Kubernetes (github.com)](https://github.com/NVIDIA/k8s-device-plugin)

使用 [helm部署](https://github.com/NVIDIA/k8s-device-plugin?tab=readme-ov-file#deployment-via-helm)


```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update

helm upgrade -i nvdp nvdp/nvidia-device-plugin \
  --namespace nvidia-device-plugin \
  --create-namespace \
  --version 0.14.5 \
  --values config.yaml
```