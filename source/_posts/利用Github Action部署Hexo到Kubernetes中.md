---
title: 利用Github Action部署Hexo到Kubernetes中
categories:
- 操作记录
tags:
- Github
- Kubernetes
- Workflow
- Docker
---

最近看上了一个`Hexo`博客的主题，就打算自己也开一个博客。惊奇的发现github上居然已经有了CI/CD功能（是我火星了）。于是乎，一个通过GitHub Action持续部署博客到Kubernetes中的想法就出现了。通过这种方式，我们可以实现0停机时间更新Hexo、弹性扩缩、健康检查和故障转移等等。。。不过其实都没啥意义，一个博客而已，跑在1c1g的机器上，没必要引入这种重型工具。

但这架不住我闲得慌，也就有了现在这一篇记录。

## 事前准备
- 一个Github账号
- 一个已经配置好的hexo博客。这方面教程太多了，不赘述了
- 一个云主机，配置无所谓
- 一定的Linux操作知识
- 经得住折腾

## 部署Kubernetes？还是K3s吧
最后我还是没有使用Kubernetes，而是转为使用了`K3s`。原因很简单，Kubernetes他实在是太大了，光是一个Etcd就够我这个可怜的1c1g主机喝一壶了。虽然K8s（Kubernetes）可能是不能用了，但我们还有K3s啊。

### 啥是K3s
[K3s](https://k3s.io/)是Rancher Lab在18年7月发行的一个基于K8s的容器编排工具。虽然在使用上和K8s几乎一模一样，但是为了减少资源的使用，k3s删除了很多云服务相关的插件（ Cloud Provider）和存储插件。同时，k3s使用sqlite作为默认的数据库，这可以进一步减少k3s的资源使用率，对于单机部署来说自然是极好的。当然，k3s也支持配置etcd以及其他数据库，如mysql、postgresql。

而在最近（2020年1月13号），k3s正式release了1.0版本。这意味着K3S已经可以正常的使用了。所以我也是第一时间的在自己的云主机上部署了一套来玩。不过值得一提的是，k3s中的容器并非是使用docker来运行的，而是`containderd`。如果想要使用docker作为container runtime，需要在启动参数中指定。

### 在服务器中安装K3s
#### 脚本安装
通常情况下，我们只需要运行一行`curl -sfL https://get.k3s.io | sh -`就可以完成K3s的安装了。

#### 手动安装K3S
假如我们不幸遇到了一些网络问题，无法顺利的下载k3s二进制文件的话。那么我们可以现在k3s的[release](https://github.com/rancher/k3s/releases/latest)页面中找到自己需要的版本，并想办法把它下载下来。

下载下来以后，我们进入到文件所在的目录，并通过scp命令，将这个文件上传到我们的机器中。
```sh
# windows中可能需要先开启openssh功能，如果不能开启openssh的话，我们可以安装git bash来运行这个命令
scp k3s root@xxx.xxx.xxx
```
上传完后，登录到我们的主机中，我们需要为其赋予启动权限，并将K3S移动到`/usr/local/bin`

```sh
# 为k3s赋予可执行权限
chmod +x k3s
# 移动文件到/usr/local/bin中
mv k3s /usr/local/bin
# 后台运行k3s，这样启动k3s会使用默认的配置，更多配置请看https://rancher.com/docs/k3s/latest/en/installation/install-options/
(k3s server &)
```

### 