+++
title = "Argocd 发布应用"
weight = 4
+++

本文将介绍如何发布应用到集群。即，如何在应用选择页面更新或者添加新的应用。

![aplications](/images/select-applications.png)


## 前置准备

- 通过 [自定义镜像]({{% ref "custom-images" %}}) 拿到需要新增或者更新的镜像名。
- 联系管理员申请 [ai-infra-project](https://github.com/linuzb/ai-infra-project) 仓库权限
- 熟悉git使用，参考 [如何参与开源项目 - 细说 GitHub 上的 PR 全过程](https://www.cnblogs.com/daniel-hutao/p/open-a-pr-in-github.html)
- 了解 yaml 的基本语法， 参考 [YAML 语言教程](https://www.ruanyifeng.com/blog/2016/07/yaml.html)

流程如下:

1. 通过 [自定义镜像]({{% ref "custom-images" %}}) 拿到需要新增或者更新的镜像名。
2. 将镜像名添加到 [ai-infra-project](https://github.com/linuzb/ai-infra-project) 仓库
3. 通过应用发布平台发布
4. 发布完成后，重新进入应用选择界面即可看到最新应用。

## 将镜像名添加到仓库

应用环境的基本配置在 `jupyterhub/singleuser-image.yaml`

![singleuser-config](/images/singleuser-config.png)

应用配置主要结构如下:

1. 该字段为前端显示的应用名，尽量使用可以区分的名字，例如使用主要软件和版本号
2. 该字段为应用实际使用的镜像名，我们将 [自定义镜像]({{% ref "custom-images" %}}) 拿到的镜像名添加到这里。 注：如果是通过 [自定义镜像]({{% ref "custom-images" %}}) 获取的镜像，镜像名的前缀 `registry.wls.linuzb.xyz/linuzb/` 不需要更改，仅更改最后的部分。
3. 如果需要增加新的应用，则参考图中注释3增加。

应用配置列表还包含其它配置：

- CPU 和内存配置
- GPU 数量配置
- GPU 类型配置

熟悉 yaml 结构之后可自行更改

## 提交配置

参考 [如何参与开源项目 - 细说 GitHub 上的 PR 全过程](https://www.cnblogs.com/daniel-hutao/p/open-a-pr-in-github.html) 教程将配置提交到远程。

注意事项:

1. 每次更新都要用一个新的分支，切不可直接提交到 main 分支
2. 一定要使用 pr 的方式合并的 main 分支

## 应用发布

我们在 home 页面找到应用发布，点击并登录。用户名密码找管理员申请。

![argocd-jupyterhub](/images/argocd-jupyterhub.png)

发布流程:

1. 检查应用发布状态，当我们有最新配置未发布到集群时，这里会显示 `OutOfSync`, 即未同步
2. 点击 `SYNC` 按钮，将应用发布到集群。
3. 当应用状态为 `Synced` 时，说明集群状态以及和发布配置相同，即发布完成。


在点击 `SYNC` 按钮之后，左侧会弹出一个发布框
![argocd-synced](/static/images/argocd-jupyterhub-sync.png)

1. 需要发布的配置，这里如果不理解可以不动。
2. 再次点击 `SYNCHRONIZE` 按钮即可
3. 等待应用状态变为 `Synced`.
4. 进入 [jupyterhub](https://hub.lab.linuzb.xyz/) 应用选择页面选择最新应用启动。

## 问题

1. 如果应用长时间未发布完成，可能是由于镜像太大，导致集群拉取镜像超时。

临时解决方案：应用回滚，即重新提交配置，将更改的部分还原，再次走发布流程，即可回滚到上一状态。然后联系管理员解决。

管理员方案：首先在其它外网环境拉取本次发布的镜像，将镜像推送到本地镜像中心。再次走发布流程。