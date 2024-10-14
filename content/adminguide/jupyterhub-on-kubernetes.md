+++
title = "部署 jupyterhub"
weight = 1
+++


## 前置准备

- 首先需要部署 kubernetes 集群，参考[KUBESPRAY 部署 K8S]({{% ref "kubespray-deploy-k8s" %}} "deply")
- [Helm | Installing Helm](https://helm.sh/docs/intro/install/) 安装 helm

### 获取 kubeconfig

kubeconfig 是一个文件，该文件中包含了集群的连接信息。我们一般使用 `kubectl` 和 `helm` 工具来访问和操作集群。

请联系管理员获取。

### 存储：Longhorn provider

- ceph
- cubefs

TODO： 补充文档

### 认证

- casdoor

TODO： 补充文档

## 使用 helm 安装/更新 jupyterhub

本文中介绍的功能配置全部放在 `config.yaml` 和 `singleuser-image.yaml` 文件中，这两个文件会合并，最终会覆盖 helm chart 中默认的 `values.yaml`

其中 `singleuser-image.yaml` 主要存放了应用镜像配置。我们发布应用主要修改这个文件。

```
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/

helm repo update

# 下载到本地，方便我们调试
helm pull jupyterhub/jupyterhub --version 3.3.6

# 解压 helm 包
tar -zxvf jupyterhub-3.3.6.tgz

# 修改 config.yaml 从本地安装
helm upgrade --cleanup-on-fail \
  --install jupyterhub ./jupyterhub \
  --namespace jhub \
  --create-namespace \
  --version=3.3.6 \
  --values config.yaml \
  --values singleuser-image.yaml
```

完整的 helm values 文件参考 [values.yaml](https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/main/jupyterhub/values.yaml)

### config.yaml

TODO: 补充 config.yaml

### jupyterhub 运行其他服务

#### vscode

[How to configure Jupyterhub to run code-server? - JupyterHub - Jupyter Community Forum](https://discourse.jupyter.org/t/how-to-configure-jupyterhub-to-run-code-server/11578)

##### jetbrans

[tiaden/jupyter-projector-proxy (github.com)](https://github.com/tiaden/jupyter-projector-proxy)

## 本地调试

本地运行 single-user-notebook

```shell
docker run \
    -p 8888:8888 \
    -v /home/linuzb/Projects/huggingface/chatglm2-6b:/home/jovyan/data/huggingface/chatglm2-6b \
    --gpus all \
    --rm \
    registry.cn-hangzhou.aliyuncs.com/linuzb/pytorch-notebook:cuda11-pytorch-2.2.1-4cb74da
```

## 参考

可以看下 binderhub
- [Integrating BinderHub with JupyterHub: Empowering users to manage their own environments | 2i2c](https://2i2c.org/blog/2024/jupyterhub-binderhub-gesis/)
- [jupyterhub/binderhub: Run your code in the cloud, with technology so advanced, it feels like magic! (github.com)](https://github.com/jupyterhub/binderhub)