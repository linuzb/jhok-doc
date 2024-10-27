+++
title = "镜像中心使用教程"
weight = 5
+++

## 简介

本项目为内部部署的镜像中心，提供镜像加速服务。


用户在拉取镜像时可以使用以下加速地址。

## docker 镜像加速

docker pull 命令会从远程 dockerhub 拉取镜像。例如 `docker pull grafana/grafana` 命令，实际上是 `docker pull docker.io/grafana/grafana`,默认情况下省略了仓库名 `docker.io`, 我们进行镜像加速就是将 `docker.io` 替换成我们的加速地址 `https://registry.wls.linuzb.xyz/proxy`

将以下配置添加到 `/etc/docker/daemon.json`

```json
{
  "registry-mirrors": [
    "https://registry.wls.linuzb.xyz/proxy"
  ]
}
```

后续再使用 `docker pull grafana/grafana` 命令就相当于 `docker pull registry.wls.linuzb.xyz/proxy/grafana/grafana`

如果我们不想持久化修改，可以直接使用命令 `docker pull registry.wls.linuzb.xyz/proxy/grafana/grafana` 临时使用。

## 其它镜像加速

除了 dockerhub 即 `docker.io` 还有其他站点的镜像仓库。

| 源站              | 替换为                   |
| --------------- | --------------------- |
| docker.io       | registry.wls.linuzb.xyz/proxy  |
| quay.io         | registry.wls.linuzb.xyz/quay-proxy    |
| k8s.gcr.io      | registry.wls.linuzb.xyz/k8s-gcr-proxy |
| registry.k8s.io | registry.wls.linuzb.xyz/k8s-proxy     |
| gcr.io          | registry.wls.linuzb.xyz/gcr-proxy     |
| ghcr.io         | registry.wls.linuzb.xyz/ghcr-proxy    |

例如，需要下载的镜像为 `docker pull quay.io/jupyterhub/jupyterhub`，加速地址为 `docker pull registry.wls.linuzb.xyz/quay-proxy/jupyterhub/jupyterhub`

## 上传镜像

### 下载镜像

首先在公网环境使用 docker 下载镜像

```shell
docker pull registry.cn-hangzhou.aliyuncs.com/linuzb/scipy-notebook:python-3.11-5d9bd54
```

然后将镜像保存成一个文件,文件名可以自定义，例如

```shell
docker save registry.cn-hangzhou.aliyuncs.com/linuzb/scipy-notebook:python-3.11-5d9bd54 > python-3.11-5d9bd54.tar
```

其中 `python-3.11-5d9bd54.tar` 文件就是导出的目标文件

### 登录镜像中心

在上传到镜像中心之前，需要先登录到镜像中心。

{{% notice style="note" %}}
该步骤需要在内网环境操作，并且登录操作只需要执行一次。
{{% /notice %}}

```shell
docker login --username=admin registry.wls.linuzb.xyz
```

然后输入密码即可。

### 上传镜像

以下步骤在内网环境操作。

首先从文件加载镜像，以 `python-3.11-5d9bd54.tar` 为例，注意使用正确的路径

```shell
docker load -i /path/to/python-3.11-5d9bd54.tar
```

加载之后的镜像名还是原来的镜像名，`registry.cn-hangzhou.aliyuncs.com/linuzb/scipy-notebook:python-3.11-5d9bd54`,我们需要将镜像名替换成私有镜像中心的镜像名

```shell
# Usage:  docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

docker tag registry.cn-hangzhou.aliyuncs.com/linuzb/scipy-notebook:python-3.11-5d9bd54 registry.wls.linuzb.xyz/library/scipy-notebook:python-3.11-5d9bd54
```

{{% notice style="note" %}}
该步骤将旧镜像名的前缀 `registry.cn-hangzhou.aliyuncs.com/linuzb` 替换成 `registry.wls.linuzb.xyz/library`, 这样镜像就可以推送到私有镜像中心，后面部分不变。
{{% /notice %}}

最后执行上传操作

```shell
docker push registry.wls.linuzb.xyz/library/scipy-notebook:python-3.11-5d9bd54
```

这样我们就可以在内部环境使用镜像 `registry.wls.linuzb.xyz/library/scipy-notebook:python-3.11-5d9bd54`