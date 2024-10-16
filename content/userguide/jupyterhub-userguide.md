+++
title = "Jupyterhub 用户手册"
weight = 2
+++

## 前置准备

本文是 jupyterhub on kubernetes 的使用文档，本文描述的环境基于 [jupyterhub on kubernetes]({{% ref "jupyterhub-on-kubernetes" %}})

## 访问

首先访问 homagepage 页面，打开 https://home.lab.linuzb.xyz, 然后找到 jupyterhub

![homepage-jupyterhub](/images/homepage-jupyterhub.png)

点击进入 jupyterhub 首页

![hub-login](/images/hub-login.png)

### 登录 && 注册

点击 `Singn in with WLS-Auth`，进入登录页面。

![casdoor-login](/images/casdoor-login.png)

新用户请选择立即注册，注册成功之后再进行登录。

{{% notice style="note" %}}
jupyterhub 采用白名单登录，首次注册成功后将用户名发送给管理员，管理员将用户名添加到白名单后方可登录。
{{% /notice %}}

## 应用选择

登录完成之后就会进入一个应用服务页面。该页面提供了预定义的一些应用环境和配置，用户可以根据自己的需求选择合适的配置和环境。

1. 首先选择用户环境 `Pytorch Env` 和 `Tensorflow Env`
2. 然后选择 CPU 和 内存
3. 然后选择 GPU 数量和类型
4. 最后选择应用。注意，该应用是提前做好的镜像，里面封装了各种软件。如果预定义的应用无法满足自己的需求，可以添加应用，具体方法请看如何[自定义镜像]({{% ref "custom-images" %}})。

选择完配置和应用之后，然后点击底部的 start 按钮，即可进入环境
![server-options](/images/server-options-0.png)

## 应用环境介绍

### 界面简介

应用创建完成后，就会跳转到jupyterlab页面
![jupyterlab](/images/jupyterlab-home.png)


介绍一下主要功能

1. 启动一个luncher，即页面右侧部分，在 luncher 中可以打开各种工具
2. 上传文件按钮，点击后弹出一个对话框，选择需要上传到文件即可。如果想上传一个目录，可以先在本地将目录打包成一个压缩包，然后上传。上传前可以先按照3. 切换到对应的工作目录。
3. 工作目录，双击可打开文件以及文件夹。上传文件以及打开vscode都是以工作目录为基础，例如，当前工作目录为`~/Projects/data`, 则上传目标路径即为`~/Projects/data`
4. 新建一个jupyter notebook
5. 以工作目录为基础打开vscode
6. 打开一个 terminal，可执行linux命令

### 自定义应用

如果应用中的软件无法满足自己的需求，想要定制自定义的应用，请参考 [自定义镜像]({{% ref "custom-images" %}})

### 使用 vscode 编写代码

打开方式：在luncher 中点击vscode 图标即可打开

Q: 如何返回jupyter 界面？

A: 由于vscode界面没做跳转链接，将浏览器url中的 `/vscode` 及之后的内容全部删除即可返回jupyter页面。或者从 home 主页重新进入。

#### 在 vscode 中切换目录

- 方法1： 返回jupyter 主界面，从左边工作栏切换目标项目目录，然后重新打开vscode

![vscode-cd](/images/vscode-change-workdir.png)

- 方法2： 直接在vscode菜单栏切换

![vscode-open-folder](/images/vscode-open-folder.png)

- 方法3：使用快捷键 `Ctrl+K Ctrl+O`

#### vscode 插件安装

和本地使用 vscode 安装插件的方式相同。

环境的销毁和重建不会影响插件。

#### vscode 用户配置

和插件一样，直接修改配置即可

### 命令行

已添加常用命令
- git
- zip, unzip
- curl

如果当前环境没有需要的软件或者 pip 包，可自行安装

```shell
# 使用 pip 安装 python 库
pip install numpy

# 使用 sudo 安装软件
sudo apt update
sudo apt install vim
```

{{% notice style="note" %}}
以上方法安装的软件只是临时的，一旦环境销毁重建后将不存在。如果想将软件或者包永久安装到环境中，请使用[自定义应用]({{% ref "custom-images" %}})的方法。
{{% /notice %}}

#### [可选]命令行美化 oh-my-zsh

我们使用 oh-my-zsh 来美化命令行和提供自动补全的能力。

详细请参考官方文档![install](https://ohmyz.sh/#install)

推荐主题
- ys

### 存储

以下内容了解即可。

由于该软件底层使用的是容器服务，一旦应用被删除（底层容器被删除），则容器中的文件都将被删除。本系统为用户提供两种持久化存储，即以下两个目录的内容除非用户手动删除，否则即使应用销毁后重建也不影响。

- `/home/{username}`,该目录默认500G空间，后端使用固态硬盘，推荐存放用于快速读取的文件。
- `/home/data`，该目录默认 1T 存储空间，后端使用机械硬盘，推荐存放不常用的冷数据。

需要长久保存的文件一定要保存在以上目录。

首次登录验证用户目录是否持久化：首先进入 `/home/{username}`或者`/home/data` 目录（在luncher中新开一个 terminal），输入命令 `cd ~/ && touch test.txt`, 该命令是进入当前用户的home目录，然后销毁环境，重新进入环境查看 `~/test.txt` 是否存在。如果存在，说明已经挂载持久化盘，如果不存在请联系管理员

### 网络


外网：未做外网限制，请注意流量消耗

内网：可以访问kubernetes 内网，即可以向 kubernetes 提交作业(分布式任务，可以使用多机多卡)，待补充。

### 销毁环境

销毁环境意味着当前运行的程序将终止，临时安装的软件将被删除，软件持久化请参考[自定义镜像]({{% ref "custom-images" %}})。但是写入 `/home/{username}` 和 `/home/data` 的数据仍然存在。

方法1：首先进入 jupyterlab 页面，点击 File->Hub Control Panel
![hub-control-link](/images/hub-control-panel-link.png)

然后进入新的界面
![stop-server](/images/stop-server.png)

双击 Stop My Server 按钮即可销毁环境

#### 自动销毁环境

为了提高资源利用率，本系统使用了超时自动销毁服务的功能。

当用户超过7d没有通过浏览器访问应用时，系统将自动销毁应用环境。用户再次登陆后需要重新选择并创建环境。

注意：
1. 超时时间时按照用户访问浏览器 url 的流量计算的，并非应用本身的活动
2. 应用的销毁并不会影响用户home目录，但是如果应用销毁时刚好有写用户home目录的操作，则可能会有影响

## 监控

监控页面可以查看应用程序的 CPU、内存、存储、网络以及 GPU 的使用情况。

请从 [homepage](https:/home.lzb.linuzb.xyz) 页面进入。

详细内容待补充。

