---
keywords:
- cloud native
- cloud native 101
- kubernetes
- container storage interface
- CSI
title: "Kubernetes Storage 101: 浅谈如何实现一个 CSI 插件"
subtitle: "深入解析 Kubernetes 存储"
description: "本文详细介绍了 Container Storage Interface (CSI) 的工作机制和集成方法，探讨了 Kubernetes 存储插件的关键概念和实现指南。文章涉及 CSI 的必要性、架构设计、生命周期管理以及如何实现高效稳定的 CSI 插件，是 Kubernetes 存储领域的深度解读。"
date: 2023-08-26T18:58:22+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Kubernetes Storage 101
- Kubernetes
---

![](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/c9dedf33-c414-400e-9dc7-d9db4102606f.png)

话接上回 [《Kubernetes Storage 101: 浅谈 K8s 存储概念，解锁数据驱动的力量》](https://mp.weixin.qq.com/s/4bzZYgVhVmvkIZpLgu9jLw)，在之前的文章中我们共同探讨了 K8s 存储概念的基础知识，为我们的学习之旅奠定了坚实的基础。

{{< article link="/posts/kubernetes-storage-101-concepts/" >}}

现在，我将继续和大家聊一聊关于 K8s 存储的一个重要组成部分：Container Storage Interface (CSI)。
在接下来的内容中，我们将会了解到 CSI 的工作原理、核心概念以及如何将其集成到你的容器化环境中。

<center>
    <img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/115f79a6-8713-46a1-98f1-7dfe7a34bd7c.png" width="10%" />
</center>

<center>
    <img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/cf0dbc88-42b2-4b17-9664-d9ff27d50e20.png" width="70%" />
</center>

## 为什么需要 CSI ？它解决了什么问题？

在学习 CSI 之前，了解其产生的背景以及它能够解决的问题我觉得是很有必要的。

### 为什么需要 CSI

虽然 Kubernetes 平台它本身支持了非常多的存储插件，但是毕竟也是有限的，永远**无法满足**用户日益增长的需求，比方说有客户要求我们的 Paas 平台必须接国产的存储怎么办？

### 面临的问题，如何做集成？

Kubernetes 本身提供了一个强大的 Volume 插件系统，最直接的方式就是向这个 Volume Plugin 增加新的插件。

但是，想必大家也知道 Kubernetes 太复杂了，它有一定的学习曲线，这样做一来成本比较高，再者直接集成第三方代码，可能会对 Kubernetes 平台系统的可靠性和安全性产生隐患。

另外，这种方法它也不方便测试和维护，比方说第三方存储服务如果有变更，我们就需要提交变更代码到 Kubernetes，等待 `Code Review`。
换句话说，我们必须要等到 Kubernetes 发布才能将存储服务的变动上线，这就意味着存储的集成与 Kubernetes 的发布周期捆绑在一起了。

所以直接和 Kubernetes 做集成是非常不方便的。

让我们重新回过来看下，[上面](https://mp.weixin.qq.com/s?__biz=MzU4MjY5NTc4OQ==&mid=2247491322&idx=1&sn=9562268f77f8d65e199f44ede2a664e4&chksm=fdb530f8cac2b9ee1a32d16819a11ba7cb280a370b9c85d33552af0ccbc7e4c5ea3c78d2b5da&scene=21#wechat_redirect)反复提到过的 **In-Tree** 和 **Out-Of-Tree** 这两个概念，我相信从字面意思上大家都已经理解了，再结合下面这张表格，大家心里是否都已经有了答案？

![](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/f63a853c-5285-4f8f-a92b-b12d186276bf.png)

### 解决了什么痛点？

CSI 将三方存储代码与 K8s 代码解耦，不同的存储插件只要实现这些统一的接口，就能对接 K8s，用户无需接触核心的 K8s 代码。

最重要的是，**CSI 规范是现在业界容器编排统一的存储接入标准**。


{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/d4164966-472d-454c-adb7-36197d75340e.png"
    alt="Container Orchestrators(CO)"
    caption="Container Orchestrators(CO)"
    >}}

## 那么，什么是 CSI？

以下是 `ChatGPT` 给出的回答：

> CSI（Container Storage Interface） 是一个规范化的接口，用于容器编排引擎与存储供应商之间的通信。它允许存储供应商编写标准的插件，以便容器编排引擎可以与其集成，从而实现更加灵活和可扩展的存储解决方案。
>
> CSI 驱动器由三个主要组件组成，每个组件都扮演着特定的角色：
>
>  1. **Node Service**: 运行在每个 Kubernetes 节点上，负责在节点上挂载和卸载存储卷，并处理节点级别的存储操作。
>  2. **Controller Service**: 运行在 Kubernetes 控制平面中，负责管理存储卷的生命周期，包括创建、删除和扩容等操作。
>  3. **Identity Service**: 它是 CSI 驱动器的第三个组件，它在 CSI 驱动器注册时提供标识信息，并向 Kubernetes 集群公开驱动器的支持能力。它负责告知 Kubernetes 驱动器的存在，提供驱动器的基本信息和功能支持。
>
>
> CSI 的设计思想是将存储管理和容器编排系统解耦，使得新的存储系统可以通过实现一组标准化的接口来与 Kubernetes 进行集成，而无需修改 Kubernetes 的核心代码。
>
> CSI 驱动器的出现为 Kubernetes 用户带来了更多的存储选择，同时也为存储供应商和开发者提供了更方便的接入点，使得集群的存储管理更加灵活和可扩展。


{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/94982fe3-1d8f-4f81-b256-113167714b34.png"
    alt="Storage in Cloud Native Environment"
    caption="Storage in Cloud Native Environment"
    >}}

CSI 适配工作是由容器编排系统（CO）和存储提供商（SP）共同完成的，CO 通过 gRPC 与 CSI 插件进行通信。相信大家也都观察到了，CSI 在这里**充当了连接的纽带**，上层连接容器编排系统，下层操作三方存储服务。

## CSI 的工作原理，它是如何工作的？

### Typical CSI driver architecture

下面是 CSI 的一个典型架构，虽然 CSI 对于存储提供商来说只要实现三个模块即可，但是整个编排流程可以说是相当复杂的。


{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/ba08a7b0-d8fe-4c33-975b-b85ffeaad0a5.png"
    alt="Kubernetes cluster with CSI"
    caption="Kubernetes cluster with CSI"
    >}}

CSI 的整个运转流程会涉及到两方面的组件：

1. 由 Kubernetes 官方维护的一组标准 external 组件，他们主要负责监听 K8s 里的资源对象，从而向 CSI Driver 发起 gRPC 调用，详见：[Kubernetes CSI Sidecar Containers](https://kubernetes-csi.github.io/docs/sidecar-containers.html "Kubernetes CSI Sidecar Containers")。它们是与 CSI 驱动器一起部署在同一个 Pod 中，用于辅助 CSI Driver 完成一些额外的任务和功能。<br/>
2. 各存储厂商开发的组件（需要实现 Identity Service，Controller Service，Node Service）

我们来看下左边的 `CSI Driver Controller` 部分，它是通过多个 Sidecar 组件配合第三方实现的插件(Controller Service)来完成的。

正如上面提到的，它负责管理存储卷的生命周期，包括创建、删除和扩容等操作。换句话说，我们的存储厂商能够提供什么样的能力，部署 Controller 的时候，就需要提供与之对应的 Sidecar 容器一起部署。

好比说我的 CSI Driver 只提供了 Dynamic provisioning 的能力，其他能力的接口我都没实现，在控制器部署的时候，我只需要将 `external-provisioner` 和我的 Controller Service 部署在一个 Pod 即可，组件的选择完全取决于三方的实现。

### Kubernetes CSI Sidecar Containers

##### #1. external-provisioner

external-provisioner 是一个 Sidecar 容器，用于在 Kubernetes 中动态地**创建**和**删除**外部存储卷。

当一个新的 PVC (PersistentVolumeClaim) 被创建时，external-provisioner 会向外部存储系统发起请求，以创建相应的存储卷，并将其与 PVC 关联，从而满足 Pod 对持久化存储的需求。

external-provisioner 实际上会执行检查，以验证 PVC 中是否存在注解 `volume.kubernetes.io/storage-provisioner`，并且该注解的值是否与 CSI 驱动程序的名称相匹配。整个流程贯穿了 **PV Controller** 这个组件。


{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/a7c0ed00-2c53-4f42-8b85-d88256312f1e.png"
    alt="provision"
    caption="provision"
    >}}

这里涉及到的两个操作分别对应着 **Controller Service** 中的 `CreateVolume` 和 `DeleteVolume` 两个接口的实现，它们的调用者正是 External Provisoner。

这一流程的核心是，external-provisioner 充当了中间人，通过 Kubernetes 的 PVC 和 StorageClass 机制，将 Pod 的持久存储需求传递给外部存储系统。这使得存储卷的创建和管理能够无缝集成到 Kubernetes 集群中，为应用提供了持久性的存储解决方案。

###### 思考

假设每个 PVC 背后对应的 Volume 都需要独立加密，并且加密密钥也各不相同，PVC 的 Spec 中已经没有额外的参数来提供这些信息了，那么我们如何将这些加密密钥传递给 CSI 接口呢？

这里有必要提一下 `CSI Operation Secrets` 这个概念，它允许针对每种不同的 CSI 操作定制不同的 Secret，并且通过 StorageClass 与之配合使用。

让我们来看下面这个 StorageClass 的定义作为例子：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: xfs2-sc-4-per-volume
provisioner: xfs2.csi.basebit.ai
parameters:
  csi.storage.k8s.io/provisioner-secret-name: ${pvc.name}
  csi.storage.k8s.io/provisioner-secret-namespace: ${pvc.namespace}
```

需要关注的是 parameters 字段中的 `csi.storage.k8s.io/provisioner-secret-name` 字段的值，它使用了变量 `${pvc.name}`。怎么理解呢？

具体而言，当我们使用 \${pvc.name} 作为 csi.storage.k8s.io/provisioner-secret-name 参数的值时，每次创建 PVC 后，都可以为它创建一个对应的 `Provisioner Secret`，并将该 PVC 的名称作为 Provisioner Secret 的名称。

这样，每个 PVC 都会拥有一个唯一的 Provisioner Secret，用于身份验证和认证。Secret 的具体定义取决于每个 CSI 驱动器的实现。在针对创建存储卷的场景中，CreateVolumeRequest 可以获取到 Secret 的详细信息。

**这种参数化技术是 Kubernetes 中允许的一种灵活方式，可用于在运行时动态生成配置文件**

> provision、delete、expand、attach 和 detach 等操作通常也需要 CSI 驱动程序在存储后端使用凭证，后面就不再多做赘述了。
>
> 如需了解更多高级用法，请参考文档：[StorageClass Secrets](https://kubernetes-csi.github.io/docs/secrets-and-credentials-storage-class.html "StorageClass Secrets")

##### #2. external-attacher

external-attacher 是一个 Sidecar 容器，其作用是在 Kubernetes 节点上动态地进行外部存储卷的 **挂载（Attach）** 和 **卸载（Detach）** 操作。

它是通过监听 Kubernetes API Server 中的 VolumeAttachment 对象，来触发 **Controller Service** 中的 `ControllerPublishVolume` 和 `ControllerUnpublishVolume` 两个接口调用。

然而，并不是所有情况下都需要使用 attach/detach 操作，尤其是在一些分布式文件系统的 CSI 驱动程序中，这样的操作可能并不适用。因此，可以说 attach/detach 是一项可选的特性。

##### #3. external-resizer

external-resizer 是一个 Sidecar 容器，用于调整外部存储卷的大小。

当 PVC 的存储需求发生变化时，external-resizer 可以根据需求调整外部存储卷的大小，确保存储资源得到最优的利用。

它会调用 **Controller Service** 中的 `ControllerExpandVolume` 接口。

##### #4. external-snapshotter

external-snapshotter 是一个 Sidecar 容器，用于实现外部存储卷的快照功能。

它是通过监听 Kubernetes Snapshot CRD 对象，来触发 **Controller Service** 中的 `CreateSnapshot` 和 `DeleteSnapshot` 两个接口调用。

它负责在外部存储系统上创建、删除和管理快照，以便于实现数据备份、恢复和复制等功能。

###### 思考

这玩意儿有什么用呢？VolumeSnapshot 允许在 Kubernetes 集群中创建卷的快照，这些快照可以用于数据备份和应用程序恢复。当应用程序出现故障或数据损坏时，我们可以使用先前创建的快照来还原应用程序的状态。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-data-my-redis-master-0
spec:
  dataSource:
    name: my-server-snapshot-0
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

##### #5. liveness-probe

liveness-probe 是一个 Sidecar 容器，用于监控 CSI 驱动程序的运行状况，并通过 Liveness Probe 机制向 Kubernetes 报告。

如果驱动器出现故障或停止响应，Kubernetes 将重启相关的 Pod(**Controller Service/Node Service**) 以确保服务的可用性。

> 有关具体的用法和配置示例，可以参考[这里](https://github.com/kubernetes-csi/livenessprobe/blob/master/deployment/kubernetes/livenessprobe-sidecar.yaml "这里") 的说明文档。

###### 思考

实际上 livenessProbe 是通过 CSI 驱动程序在容器上设置的。 `liveness-probe` sidecar 容器的主要作用是提供了 `/healthz` 这个 HTTP 端点。
它背后 checkProbe 最终向 CSI 驱动程序的 **Identity Service** 中的 `Probe` 接口发起调用，用来检测插件是否处于健康状态。

这种架构使得插件的健康状态检查与应用程序分离，通过 CSI 驱动程序的 Probe 接口进行通信。

事实上 Kubernetes 从 v1.23 开始具有内置 gRPC 健康探测，已经不需要这么麻烦了。

##### #6. node-driver-registrar

node-driver-registrar 是一个作为 Sidecar 容器运行的组件，其主要职责是通过直接调用 CSI 驱动程序的 **Node Service** 中的 `NodeGetInfo` 接口，获取驱动程序的信息。

然后，它会利用 kubelet 的插件注册机制，将这些驱动程序的信息注册到相应节点的 kubelet 中。

##### 小结

我们需要记住这张能力关系组合表，它对部署 CSI 驱动程序和排查问题非常的有用。

![](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/f466771f-2228-42b2-89bc-4719b3fffb6e.png)

## 如何实现一个 CSI 插件？

要实现一个 CSI 驱动程序，确实只需要完成一系列接口的实现即可，但仅仅完成这些接口的实现还不足以构建一个稳健、可用的 CSI 驱动程序，构建一个稳健的驱动程序还需要考虑方方面面。

### CSI Plugin components

下面是每一个 CSI 驱动程序要实现的接口清单。

其中 `Identity Service` 负责提供 CSI 驱动程序的身份信息，`Controller Service` 负责 Volume 的管理，`Node Service` 负责将 Volume 挂载到 Pod 中。

![](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/17d73994-aca8-4482-ae60-ce8ce4c1369a.png)

正如前面提到的，一个 CSI 驱动程序能提供什么样的能力，取决于各自存储厂商的实现，三个组件都有对外暴露能力的接口，比如

1. Identity Service 中的 `GetPluginCapabilities` 方法，表示该 CSI 驱动程序主要提供了哪些功能。
2. Controller Service 中的 `ControllerGetCapabilities` 方法，实际上告诉 K8s，CSI 驱动程序具备哪些能力。这些能力可以包括卷的创建、删除、扩容、快照等操作。
3. Node Service 中的 `NodeGetCapabilities` 方法，提供 Node plugin 的能力列表。

### CSI lifecycle

在通常情况下，每个 Volume 都会经历完整的生命周期过程。

![](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/b1cb5cb9-bad6-4b2e-a660-06e729c39dc4.png)

从创建 PersistentVolumeClaim（PVC）开始，接着被 Pod 所使用，这个过程包括三个主要阶段：**Provision -> Attach -> Mount**。

随后，从 Pod 开始被删除，直到 PVC 被删除，整个过程又经历了另外三个关键阶段: **Unmount -> Detach -> Delete**。

然而，存在一种特殊的存储卷，它就是 `Ephemeral Inline Volumes`，它可以通过改变 CSIDriver 的规范中的 volumeLifecycleModes 参数来改变其生命周期。

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: xfs2.csi.basebit.ai
spec:
  ...
  volumeLifecycleModes:
  - Ephemeral
```

`Ephemeral` 模式表示存储卷是临时的，会随着 Pod 的生命周期结束而被释放。对于这种类型的存储卷，Kubelet 在向 CSI 驱动请求卷时，只会调用 `NodePublishVolume`，省略了其他阶段（例如 CreateVolume、NodeStageVolume）的调用。而在 Pod 结束需要释放存储卷时，只会调用 `NodeUnpublishVolume`。

具体的 Pod 规范如下所示：

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: my-csi-app
spec:
  containers:
    - name: my-frontend
      image: busybox
      volumeMounts:
      - mountPath: "/data"
        name: my-csi-inline-vol
      command: [ "sleep", "1000000" ]
  volumes:
    - name: my-csi-inline-vol
      csi:
        driver: xfs2.csi.basebit.ai
        volumeAttributes:
          foo: bar
```

这里的 **volumeAttributes** 用于指定驱动程序需要准备的卷的属性。这些属性在每个驱动程序中都是特定的，没有标准化的实现方法。

##### Ephemeral 使用案例

> [Secrets Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver "Secrets Store CSI Driver")允许用户将 Secret 作为内联卷从外部挂载到一个 Pod 中。当密钥存储在外部管理服务或 Vault 实例中时，这可能很有用。
>
> [Cert-Manager CSI Driver](https://github.com/cert-manager/csi-driver "Cert-Manager CSI Driver") 与 [cert-manager](https://cert-manager.io/ "cert-manager") 协同工作， 无缝地请求和挂载证书密钥对到一个 Pod 中。这使得证书可以在应用 Pod 中自动更新。

通过这个特性，再一次说明了我们并不要求所有的接口都需要实现，取决于插件实现方提供什么样的能力，我们再去实现相应的逻辑即可。

### CSI idempotent

我们应该确保所有的 CSI 操作都是幂等的，这意味着同一操作被多次调用时，结果始终保持一致，不会因为多次调用而导致状态变化或产生额外的副作用。
<br/>这种幂等性是保证系统稳定性和一致性的关键因素。

举个例子，假设我们做一个 DeleteVolume 的操作，如果底层的 Volume 已经不存在了，依然不能报错。无论是第一次执行 DeleteVolume 还是多次重试，操作的最终结果都应该是相同的。

这里不得不提一下 `CSI Sanity Test`，它可用于验证 CSI 驱动程序的基本功能和稳定性。它会模拟不同的错误和异常情况，例如创建已存在的卷、卸载不存在的卷等，以验证驱动程序对这些情况的处理是否正确。

它对开发 CSI 驱动程序非常的有帮助，可以帮助开发人员快速定位和修复常见问题，减少在生产环境中出现意外问题的可能性。

> 官方文档中详细阐述了规范（Container Storage Interface，CSI）的内容，同时还提供了与开发相关的注意事项。
>
> 这些注意事项涵盖了规范中的一些关键要点，以及在开发过程中可能会遇到的挑战和解决方案。
> <br/> 我们可以在 [Container Storage Interface (CSI) Specification](https://github.com/container-storage-interface/spec/blob/master/spec.md "Container Storage Interface (CSI) Specification ") 找到这些详细信息。

## 如何部署 CSI?

标准的 CSI 驱动程序部署架构如下图所示，其中包括一个由 DaemonSet 运行的 CSI Node 组件，以及一个运行在 StatefulSet 内的 CSI Controller 组件。

{{< alert >}}
这两个容器通过本地 Socket (Unix Domain Socket, UDS)进行通信，并使用 gRPC 协议。CSI 插件直接与同一宿主机上的 K8s 组件进行交互，通过本机进程之间的 Unix 域套接字通信，相较于 TCP 套接字，具备更高的通信效率和性能。
{{< /alert >}}

![](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/implementing-csi-plugin/f057e23f-09d9-43e0-b57e-ac6dcd7b9419.png)

在部署 CSI Node 时，需要将宿主机上的 kubelet 目录(/var/lib/kubelet)挂载到驱动程序的容器内，且需将 Mount Propagation 设置为 `Bidirectional`。这样，驱动程序容器内的后续 Mount/Umount 操作能够传播到宿主机上。

请注意，这只是一个高层次的架构概述，具体的实施细节可能会因不同的 CSI 插件和环境而有所变化。

CSI 在集群部署成功后，可以用以下两个命令来做下检查：

**#1. 查看集群内安装的 CSI Driver**

```bash
➜ kubectl get csidrivers
```

**#2. 列出哪些节点具有 CSI**

```bash
➜ kubectl get csinodes
```

## 总结

CSI 是 Kubernetes 存储体系中的核心组件，为存储供应商提供了灵活且可扩展的集成方式，也为 Kubernetes 用户提供了高效稳定的存储解决方案。

通过标准化容器编排器与存储供应商之间的接口，CSI 构建了一种统一的范式，确保所有与 CSI 兼容的存储系统都遵循相同的实现规范。事实上，通过编写一个 CSI 驱动程序，我们不仅为 Kubernetes 存储架构增添了新的维度，还深化了对存储资源管理的理解。

下一期，我将继续与大家分享在实际工作中使用 CSI Driver 遇到的问题和挑战。

## 一些资料

- [CSI Documentation](https://kubernetes-csi.github.io/docs/ "CSI Documentation")
- [CloudNativeCon EU 2018 CSI Jie Yu](https://schd.ws/hosted_files/kccnceu18/fb/CloudNativeCon%20EU%202018%20CSI%20Jie%20Yu.pdf "CloudNativeCon EU 2018 CSI Jie Yu")
