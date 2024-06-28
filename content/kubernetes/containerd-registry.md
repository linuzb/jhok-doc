+++
title = "Containerd 镜像配置"
weight = 6
+++

## 引言

本文将介绍如何修改 containerd 的默认镜像仓库。

## 配置

首先我们打开 containerd 的默认配置文件

```shell
sudo vim /etc/containerd/config.toml
```

首先检查 `[plugins."io.containerd.grpc.v1.cri".registry]` 下面的配置，如果还存在 `[plugins."io.containerd.grpc.v1.cri".registry.mirrors]` 则把这项配置全部删除

例如, 该配置的作用是为 `docker.io` 的请求配置镜像站 `endpoint`, 其中 `endpoint` 是一个数组，containerd 会按顺序从数组中给出的镜像仓库拉取镜像。
```toml
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
```

现在我们切换到新的配置方式, [Contaienrd Configure Image Registry](https://github.com/containerd/containerd/blob/main/docs/cri/registry.md)

```toml
[plugins."io.containerd.cri.v1.images".registry]
   config_path = "/etc/containerd/certs.d"
```

{{% notice style="tip" %}}
"io.containerd.grpc.v1.cri" 替换成了 "io.containerd.cri.v1.images"
{{% /notice %}}

其中 config_path 指定了镜像仓库配置的目录, 目录结构如下。使用镜像地址作为目录名, 例如 `docker.io`.

```shell
/etc/containerd/
├── certs.d
│   ├── docker.io
│   │   └── hosts.toml
│   └── registry.cn
│       └── hosts.toml
├── config.toml
└── cri-base.json
```

以 `/etc/containerd/certs.d/docker.io/host.toml` 配置为例

```toml
server = "https://docker.io"

[host."https://docker.m.daocloud.io"]
  capabilities = ["pull","resolve"]
  skip_verify = false
  override_path = false

[host."https://registry-1.docker.io"]
  capabilities = ["pull","resolve"]
  skip_verify = false
  override_path = false
```

- `server` 制定了镜像仓库地址
- `host."https://docker.m.daocloud.io"` 制定了镜像仓库地址，用户可以添加多个镜像仓库，containerd 将按顺序拉取。

修改完配置之后，重启containerd

```shell
sudo systemctl daemon-reload
sudo systemctl restart containerd
```