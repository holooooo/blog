---
title: 在Rancher中将导入集群改为自定义集群
categories:
- 技巧
- Kubernetes
tags:
- Rancher
- Kubernetes
- 技巧
---

# 将导入集群改为自定义集群

假设我们曾经通过rancher的自定义集群的方式创建了一个`自定义`集群。但因为种种原因这个集群最后变成了`导入`集群，从此不能再在rancher server的界面上自由的添加node资源，一键升级kubelet的版本。为了恢复这些本应有的权利，应当通过修改crd的方式从新将集群变为自定义集群。本文档中将详细介绍改动方法。

## 注意
- 本操作只在 Rancher 2.3.x 中复现过，对于其他版本请自行判断能否使用
- 导入集群应当原本是使用`RKE`创建的
- 改动操作有很大风险，可能会导致`无法操作集群`。在操作前应该做好备份
- 就算版本一致，这个文档的结果可能无法重现。在操作前应该做好备份
- 最好按照文档中的顺序操作，否则可能导致`无法操作集群`

## 整体步骤一览

1. 将该集群的全部nodes.management.cattle.io改为rkeNode
2. 将该集群的clusters.management.cattle.io改为自定义
3. 得到并修改rke所需的访问凭据
4. 导入rke所需的连接凭据
5. 导入该集群的全部nodes.management.cattle.io
6. 导入该集群的clusters.management.cattle.io

导入过程的顺序绝对不能乱，否则可能导致`无法操作集群`

## 具体步骤

### 将该集群的全部nodes.management.cattle.io改为rkeNode
1. 在rancher server所在的集群或者容器中通过`kubectl get nodes.management.cattle.io -o yaml -n 导入集群id`得到全部节点信息。得到的节点信息是一个k8s的list类型yaml，其中包含了全部节点的信息。
2. 将每一个节点的`spec.imported`的值改为`true`
3. 在每一个节点的`status`中增加rke设置。其模板如下：

```
status：
rkeNode:
    # 节点地址，可以在status.internalNodeStatus中找到
    address: 25.2.20.10
    # 节点名称，可以在status.internalNodeStatus中找到
    hostnameOverride: region2-worker05
    # 节点命名空间与id，可以在metadata中找到
    nodeName: c-j6b4r:machine-l8pzm
    # 节点端口号
    port: "16022"
    # 节点身份， 可以在spec中查看controlPlane、etcd、worker这个字段的值来判断
    role:
    - etcd
    - controlplane
    - worker
    user: root
```
### 将该集群的clusters.management.cattle.io改为自定义集群

1. 在rancher server所在的集群或者容器中通过`kubectl get clusters -o yaml 导入集群id`得到并保存集群的全部信息。
2. 将`spec.localClusterAuthEndpoint.enabled`的值改为`true`
3. 在`spec`中加入RKE设置，RKE设置模板如下
```
# 如果可以的话，最好用备份中的clusters.management.cattle.io的设置
rancherKubernetesEngineConfig:
addonJobTimeout: 30
authentication:
    strategy: x509|webhook
authorization: {}
bastionHost: {}
cloudProvider: {}
ignoreDockerVersion: true
ingress:
    provider: nginx
# kubernetesVersion最好和备份中一致，或者不要低于备份中的版本
kubernetesVersion: v1.14.8-rancher1-1
monitoring:
    provider: metrics-server
network:
    options:
    flannel_backend_type: vxlan
    plugin: canal
privateRegistries:
- isDefault: true
    url: xxxx.com
restore: {}
services:
    etcd:
    backupConfig:
        enabled: true
        intervalHours: 12
        retention: 6
        s3BackupConfig: null
    creation: 12h
    extraArgs:
        election-timeout: "5000"
        heartbeat-interval: "500"
    retention: 72h
    snapshot: false
    kubeApi:
    serviceNodePortRange: 30000-32767
    kubeController: {}
    kubelet: {}
    kubeproxy: {}
    scheduler: {}
sshAgentAuth: false
systemImages: {}
```
`status`如果可以的话最好直接复制备份中的版本过来。如果不行的话则按照以下步骤修改`status`中内容

将`status.apiEndpoint`修改为与备份中的自定义集群一致的`endpoint`。如果找不到自定义集群的信息，可以在导入集群`system`项目的`kube-system`命名空间下发现一个名为`full-cluster-state`的`configmap`。将这个configmap中full-cluster-state字段的address作为域名覆盖在`status.apiEndpoint`的域名上，再将其端口号从`443`改为`6443`

将`spec.rancherKubernetesEngineConfig`复制到`status.appliedSpec.rancherKubernetesEngineConfig`。并将`status.appliedSpec.localClusterAuthEndpoint.enabled`改为`true`

将第一步中全部节点的`status.rkeNode`字段以数组的格式复制到`status.appliedSpec.rancherKubernetesEngineConfig.nodes`字段下。其格式如下所示：
```
    nodes:
    - address: 25.2.20.10
        hostnameOverride: region2-worker05
        nodeName: c-skfdl:m-f446c7eb883d
        port: "22"
        role:
        - etcd
        - controlplane
        - worker
        user: root
    - address: 25.2.20.10
        hostnameOverride: region2-worker05
        nodeName: c-skfdl:m-f446c7eb883d
        port: "22"
        role:
        - etcd
        - controlplane
        - worker
        user: root
```

将`status.driver`字段的值改为`rancherKubernetesEngine`。如果不改会导致集群更新事件被忽略

### 得到并修改rke所需的访问凭据

1. 得到rke所需的访问凭据

rke在创建集群的过程中，会将各种所需信息保存在rancher server所在集群或容器中。在Rancher Server ETCD中的`cattle-system`命名空间下你可能可以看到一个命名格式为`c-自定义集群id`的密文。这个密文中包含全部RKE所需要的访问凭据。


    如果没有看到，就从过往rancher server的etcd备份中使用`kubectl get secret -n cattle-system c-自定义集群id`得到该信息。


    要是还是没有，可以考虑临时新建一个集群，以这个集群的密文为模板，对照着导入集群`system`项目的`kube-system`命名空间下一个名为`full-cluster-state`的configmap创建密文。不过不建议这样做。

2. 修改其中的`name`字段为导入集群的id。
3. 通过全局搜索`c-`，将密文中节点信息的`nodeName`改为第一步里全部节点信息中对应节点的`status.rkeNode.nodeName`。要是改错可能导致集群长期处于`Updating`状态。
4. 验证`clientCertificate`字段通过base64解码后是否与`full-cluster-state`中的`certificatePEM`一致。不一致就可以停止本次改动了。否则很可能导致rancher server重启之类结果的。

### 导入rke所需的访问凭据

在rancher server所在集群或容器中的`cattle-system`命名空间下创建一个命名为`c-自定义集群id`的密文，创建一个键为`cluster`，值为第三步结果的密文，并保存。

### 导入全部nodes.management.cattle.io

使用`kubectl edit nodes.management.cattle.io -n 导入集群id`修改节点信息

### 导入clusters.management.cattle.io

使用`kubectl edit clusters.management.cattle.io 导入集群id`修改集群信息

## 其他需要改动的

这样修改后，流水线和应用商店中已经部署的应用还有项目信息这些都是回不来的，需要在实际使用中逐个将其namespce字段改为新集群新项目的ID，对于这一部分往往都很简单，也没有什么危险性，就略过不讲了。

