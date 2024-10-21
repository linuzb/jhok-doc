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

待补充