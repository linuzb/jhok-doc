+++
archetype = "home"
title = "简介"
+++

Jupyterhub 允许用户通过网页与计算环境交互，用户可以使用浏览器实现在线或者离线的数据分析，从而摆脱本地计算环境的资源限制。有了 JupyterHub，你就可以管理多个用户的 jupyter 服务。

![jupyterlab](https://jupyter.org/assets/homepage/labpreview.webp)

[Kubernetes](https://kubernetes.io/) 是一个容器编排服务。简单来说，Kubernetes 可以管理一个集群（包含多个计算机），我们可以在 kubernetes 上部署应用，这些应用会被 Kubernetes 调度到集群中的某台机器上运行。用户只需要关心应用运行结果，而不必关心资源调度与分配的过程。

> 注: Kubernetes 又叫做 k8s, 原因是在 k 和 s 之间有 8 个字母。

那么我们是否可以将 Jupyterhub 和 Kubernetes 结合，用来支持多用户 jupyter 服务的申请，并且让Kubernetes 实现资源的管理与分配呢。

实际上，Jupyterhub 官方已经支持，这个项目叫做 [Zero to JupyterHub with Kubernetes](https://z2jh.jupyter.org/en/latest/index.html)，感兴趣的可以直接看 jupyterhub 的官方文档。

本文将基于 [Zero to JupyterHub with Kubernetes](https://z2jh.jupyter.org/en/latest/index.html) 官方文档较少的方法，定制一个适合多用户深度学习计算平台。
