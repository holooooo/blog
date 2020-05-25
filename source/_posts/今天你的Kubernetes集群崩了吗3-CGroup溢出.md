---
title: 今天你的Kubernetes集群崩了吗3-cgroups溢出
date: 2020-05-25
updated: 2020-05-25
categories:
  - Record
tags:
  - Kubernetes
  - Crash
  - Linux Kernel
---

> 本问题只会在 K8s1.8 版本以上和较低的内核版本的场景中出现，不过目前已知在 4.4 中依旧会出现。如果还在用较低版本内核的同志们就需要注意一下这个问题了。内核版本高的可以忽略了。

有阵子发现很多 worker 节点用着用着就会提示剩余空间不足的错误，具体如下所示：

```
mkdir /sys/fs/cgroups/memory/docker/406cfca0c0a597091854c256a3bb2f09261ecbf86e98805414752150b11eb13a: no space left on device
```

这个时候任何新的 Pod 就无法成功的部署在该节点上了，并且越是使用的频繁的节点越容易出现这一异常。可实际上通过`df`命令查看硬盘使用情况的时候却发现这个目录所在的分区可能连 1%的空间都没用到。如果运气不好，一个小集群里出现好几个这样的机器，那本不富裕的家族就要雪上加霜了。

想要理解这个问题，需要从容器的虚拟化实现原理开始。不过由于我本身也就是一个初入 Kubernetes 的新人，所以我会尽量记录下事情的前因后果，避免他披个马甲我就认不出来了。

## Docker 虚拟化原理

通常当我们想要介绍 Docker 的时候，我们会怎么说？无论是怎么描述 Docker，我们总是会提到 Docker 的一个最重要的特性之一：轻量级。Docker 之所以是轻量级的虚拟化技术，是因为 Docker 并非是建立于 hypervisor 之上的虚拟机应用。通过下图我们可以看到 Docker 和基于 hypervisor 的虚拟机技术之间的区别：

![Docker 工作原理](/images/docker工作原理.png)

在上图中我们可以看到，Docker 容器在运行时并不会带入一个新的操作系统，可以这么说 Docker 容器就是一个带有特殊设置的普通进程。但既然容器只是一个普通进程，那么为什么容器并不能直接访问到宿主机上的资源呢？而且通过设置，容器也不能无限制的使用宿主机的资源。为了实现容器计算资源和进程工作区的隔离， Docker 中 Namespace 和 cgroups 这两项技术。

### 资源隔离

Namespace 和 cgroups 都是 linux 内核中的功能。

**Namespace**: Namespace 可以为进程提供内核资源上的隔离。通过 Namespace，我们可以实现进程、网络、用户、UTS 等多种资源的隔离。其中 Docker 使用了以下几种 Namespace:

- pid: 进程隔离
- net: 网卡隔离
- ipc: IPC(进程间通信)隔离
- mnt: 挂载点(文件)隔离
- UTS: 内核版本，主机名，域名隔离

简单来说，有了 Namespace，我们就不需要担心容器会访问到不该访问的资源。

**cgroups**: 即 Linux Control Group。这项技术主要用来实现物理资源的使用上限，如 cpu、内存、磁盘和带宽等。docker 通常会通过 cgroup 来实现 cpu 和内存的限制。

## 故障成因

本次故障就是由于 cgroup 的 bug 所导致的。当我们为工作负载设置内存限制时，Docker_HOST 会在`/sys/fs/cgroups/memory/docker/`目录下为容器创建对应的目录，并在其中设置内存的使用上限。

memcg 是 linux 当中用来管理 cgroup 内存的模块。在 memcg 中，其 id 是一个非常有限的资源，其上限是 65535。通常情况下，伴随着 cgroup 的消亡，其 memcg struct 应该被释放，然而事与愿违，memcg id 始终没有被释放，最终导致内核认为的 cgroup 数量和实际 cgroup 数量并不一致，最终导致前文中错误日志产生。

## 处理方法

在遇到这种问题的时候，我们有很多的解决方式，最好的办法当然还是升级内核版本，一劳永逸的避免该类问题的复现。但是很多情况下我们并不能随意的去升级运行中的节点的内核，除了升级内核外还有另外几种方法。

粗暴的重启节点，也可以刷新其计数，使该问题暂时消失。但是时间稍微一长就可能还是会出现一样的情况。并且重启节点可能会导致 Pod 漂移等意外结果，这种方式也并不推荐。

第三种方式则是设置一个定时任务`6 */12 * * * root echo 3 > /proc/sys/vm/drop_caches`，定期将释放计数。但是这种方式会影响性能并可能导致一些状态为`dangling`的容器 cgroups 被释放。这也不是一个好的办法。

最后我们可以将内核参数设置为`cgroup.memory=nokmem`来避免memcg进行计数。也可以重新编译kuberntes相关组件来禁止使用相关特性。

如果条件允许的话，我认为升级内核还是最好的选择，或者升级 rhel 到两个月前发布的 7.8（危）。不过具体要使用什么方式来应对问题，需要依据实际情况来决定。各个方式都有其适用的场景

## 捞一下往期

- [今天你的 Kubernetes 集群崩了吗 1-Pod 无限创建](/Record/今天你的Kubernetes集群崩了吗1-Pod无限创建)
- [今天你的 Kubernetes 集群崩了吗 2-双 Master 节点](/Record/今天你的Kubernetes集群崩了吗2-双Master节点)

## 参考资料

- https://lkml.org/lkml/2016/7/13/587
- https://github.com/kubernetes/kubernetes/issues/61937
- https://github.com/kubernetes/kubernetes/issues/70324
- https://lore.kernel.org/patchwork/patch/690171/
- https://docs.docker.com/get-started/overview/#the-underlying-technology
