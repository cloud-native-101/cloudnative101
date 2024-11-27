---
title: "Kubernetes Storage 101: 引入动态挂载卷，实现工作负载运行时的存储灵活性(下)"
description: "一种基于 Kubernetes Operator 模式和 CSI Driver 的动态存储挂载方案，通过自定义资源实现工作负载运行时存储的灵活添加和移除，体现云原生存储管理的创新理念。"
date: 2024-11-27T14:44:45+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Kubernetes Storage 101
- Kubernetes
---

![cover](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-storage-101-dynamic-volume-mounting-runtime-flexibility/a179d8b5-4c28-47ac-bf45-4ece2e7f3e09.webp)

在上一篇文章[《Kubernetes Storage 101: 引入动态挂载卷，实现工作负载运行时的存储灵活性(上)》](https://mp.weixin.qq.com/s/kDPEl65SK_6X-LRLi6tn1A) 中，我们探索了如何在 Kubernetes Pod 中为运行中的实例动态的添加或移除存储资源。然而，这种方法仅适用于 Pod 类型的工作负载，缺乏对 Kubernetes 中其他类型工作负载的支持。此外，它通过 `gRPC` 服务来暴露功能，这与 Kubernetes 的声明式管理模式相悖。

{{< article link="/posts/kubernetes-storage-101-dynamic-volume-mounting-runtime-flexibility-part-1/" >}}

因此，本篇文章将深入探讨如何利用 Kubernetes 的 Operator 模式，设计一个通用且符合云原生理念的动态存储挂载方案，突破传统方法的局限。

## 从声明式管理模式说起

在云原生的世界里，Kubernetes 的声明式管理就像是一个智能的"状态协调器"。

举个简单的例子，如果我们想扩容一个 Deployment，只需修改其 YAML 配置中的 `spec.replicas` 字段，然后 `kubectl apply`，Kubernetes 就会自动将当前状态调整到我们期望的状态。

同样地，在实际应用中，我们经常需要为工作负载添加额外的配置，例如为 Pod 增加 Sidecar 容器，或者追加新的 Volume。这些操作都可以通过修改资源的声明式配置来完成，既直观又高效。

我们的动态存储挂载方案将延续这一理念。

### 引入自定义资源定义（CRD）

通过自定义资源，我们可以扩展 Kubernetes 的存储管理能力。

例如，我们可以创建一个名为 `DynamicVolumeClaim` 的 CRD，专门用于动态挂载存储卷。下面是一个示例，展示了如何将 S3 存储桶动态挂载到具有特定标签（例如 app=nginx）的 Pod 中：


```yaml
apiVersion: dynamicvolume.csi.cloudnative101.net/v1alpha1
kind: DynamicVolumeClaim
metadata:
  name: dynamicvolume-s3-sample
spec:
  # 目标选择器：精确匹配需要挂载存储的工作负载
  selector:
    matchLabels:
      app: nginx

  # 存储提供者配置
  s3Config:
    bucket: my-s3-bucket
    endpoint: http://my-s3-service.example.com

  # 挂载配置
  mountpoint: /data
  readOnly: true
```

在这个示例中：

**selector**: 指定了目标 Pod，可以通过标签选择器精准地匹配需要挂载存储的 Pod。例如 `app: nginx` 会选择所有带有 `app=nginx` 标签的 Pod。

**s3Config**: 提供了连接到 S3 存储桶所需的信息，包括存储桶名称（`bucket`）和 S3 服务的访问端点（`endpoint`）。

**mountpoint**: 指定了在 Pod 内部挂载 S3 存储桶的路径。

**readOnly**: 标识挂载的存储是否为只读。


通过这种方式，我们无需修改 Pod 的定义，就可以动态地为指定的工作负载挂载存储卷，完美契合了 Kubernetes 的声明式管理模式。

## 走向通用化的设计

在 Kubernetes 的生态系统中，存在多种 built-in 的工作负载类型，如 Deployment、StatefulSet、DaemonSet、Job 以及通过 CRD 扩展的资源等等，**它们最终都会生成 Pod**。而我们的目标是实现一种通用的动态存储挂载方案，适用于所有类型的工作负载。

### 聚焦 Pod 层面

既然所有的工作负载最终都以 Pod 的形式运行，那么我们可以将 `DynamicVolumeClaim` 的作用对象定位在 Pod 层面。通过在 CRD 中使用选择器，我们可以精准地为匹配条件的 Pod 动态挂载存储卷，而无需关心它们是由哪种工作负载类型生成的。

#### 思考

让我们停下来想想：如果我们只关注 Pod 层面，是否意味着所有类型的工作负载都能变得更加轻松？这样的设计思路是否能够兼顾通用性和性能？如何在多样化的工作负载中实现一致的存储挂载管理？

### 与现有资源的协作

为了实现与 Kubernetes 现有资源的良好协作，`DynamicVolumeClaim` 应该遵循 Kubernetes 的 API 约定和资源模型。这包括：

**兼容性**：确保 `DynamicVolumeClaim` 能够与现有的存储资源（如 PersistentVolume 和 PersistentVolumeClaim）以及存储类（StorageClass）等概念兼容。

**扩展性**：通过支持自定义参数和配置，使得 `DynamicVolumeClaim` 能够适应不同的存储后端和需求，例如支持不同的云存储服务、文件系统选项等。

**安全性和隔离**：考虑到多租户环境的需求，`DynamicVolumeClaim` 应该提供足够的安全机制和隔离策略，确保存储资源的访问控制和数据保护。

通过以上设计考虑，`DynamicVolumeClaim` 可以成为一个通用的、灵活的存储管理解决方案，适用于 Kubernetes 生态系统中的各种工作负载和场景。

### 与 Kubernetes 原生能力融合

为了使我们的方案更加云原生，我们需要充分利用 Kubernetes 提供的特性。例如：

**Mount Propagation**：利用挂载传播特性，实现容器之间以及容器与宿主机之间的挂载共享。

**Operator 模式**：通过编写自定义的 Controller，监听 `DynamicVolumeClaim` 资源的变化，自动执行挂载和卸载操作。

## 如何设计挂载点

挂载点是存储资源与容器交互的关键接口。通过预设的挂载点，我们可以确保存储的变化能够及时、准确地传播到需要的容器中，实现动态挂载的目的。


挂载点的设计是实现动态挂载的关键。在业界，常见的做法有两种：

**Sidecar 模式**：在同一个 Pod 内，通过共享 Volume 的方式，让主容器和 Sidecar 容器共享同一个存储卷。这样，Sidecar 容器可以负责挂载和管理存储资源，主容器则直接使用。
    
1. 优点：Pod 内共享存储，管理简单。
2. 缺点：额外的容器资源开销
   
**HostPath 共享**：通过在同一个节点上的第三方 Pod，利用 HostPath 目录作为挂载点，通过 Mount Propagation 实现挂载的传播。
    
1. 优点：跨 Pod 高效共享
2. 缺点：需要精确的权限控制

动态挂载卷的工作原理其实也非常简单，归根结底也是通过一个预设的挂载点来传播存储，我们选择了基于 Mount Propagation 的方案，兼顾了性能和灵活性。

思考一下：选择 Sidecar 模式还是 HostPath 共享模式，哪种方式更适合你的场景？不同的应用场景可能会有不同的答案。

## 技术架构：通用动态挂载方案

为了让大家更清晰地理解，我们来看一下具体的实现。

### 动态卷 CSI Driver

我们可以开发一个名为 **Dynamic Volume CSI Driver** 的组件，利用 [Container Storage Interface (CSI)](https://kubernetes-csi.github.io/docs/) 的 Inline Volume 特性，将动态挂载功能与 Kubernetes 紧密集成，为工作负载提供通用的动态存储挂载能力。

> 我为什么使用 CSI Driver 来做动态挂载呢？设计灵感来源于 [Secrets Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver) 这个项目，感兴趣的读者可以阅读一下。


### 工作原理

如下图所示，方案主要由 **CSI Node Service（DaemonSet）** 和 **Dynamic Mount Pod（DMP）** 组成。


![architecuure](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-storage-101-dynamic-volume-mounting-runtime-flexibility/architecture-1.jpg)

##### CSI Node Service

主要负责处理用户的动态挂载请求，以及管理 Dynamic Mount Pod（DMP）的生命周期。它实现了 CSI 的 Ephemeral Inline Volume 能力，同时包含三个控制器来完成控制面的任务。

###### **控制器组件**：

**Storage Provider Class Controller**：管理存储提供者的配置，防止资源被随意删除。

**External Volume Controller**：监听 `DynamicVolumeClaim` 资源的创建和删除，根据不同的 `Storage Provider` 创建或销毁 DMP，以及恢复异常关闭的 Dynamic Mount Pod。

**Dynamic Mount Pod Controller**：监听当前节点所在的 DMP，当所有挂载都不存在时，释放 DMP 及相关资源。

##### Dynamic Mount Pod

- 实际执行挂载操作的 Pod。
- 通过 Informer 监听 Pod 的变化（DynamicVolumeClaim CR 的增加或者删除行为，它会同步反映到 DMP 的 annotation 中）
- 在 `/xvols` 目录下执行存储的挂载或卸载操作。

##### 思考

你可能会问：为什么不直接将挂载点设置为宿主机目录？虽然这样实现起来简单，但风险巨大，尤其在节点故障时，爆炸半径会很大，影响整个节点下的用户实例。这也是我们必须谨慎考虑的方面。

### Diagram  

通用动态挂载卷的核心功能充分利用了 Kubernetes 提供的 Mount Propagation（挂载传播）能力，在 DMP 内实现对存储卷的动态挂载。下面的图示详细展示了用户 Pod 内实现动态挂载卷的过程，并阐述了与之相关的 Mount Propagation 组件。


![architecuure](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-storage-101-dynamic-volume-mounting-runtime-flexibility/diagram.jpg)

**步骤 1**: 用户创建 Pod。在 Pod 的 annotation 中，用户需要指定预先创建好的 StorageProviderClass（如 `s3`）和 volumeMountPathRoot（如 `/user-defined-root-path`）。

**步骤 2**: 在 Pod 初始化阶段，Mutating Webhook 将一个 `Ephemeral Inline Volume` 注入到用户的 Pod。

**步骤 3**: CSI Node Service 利用 K8s CSI Inline Volume 特性，将 Pod 内的 `/user-defined-root-path` 目录与宿主机的某一特定目录进行映射，路径如下：`/var/lib/kubelet/pods/<POD-ID>/volumes/kubernetes.io~csi/<PV-NAME>/mount`。

**步骤 4**: 在 NodePublishVolume 阶段结束时，CSI Node Service 会为该 Pod 在当前宿主机上分配一个专属目录。此目录路径如：`/var/lib/dynamic/xvols/<storage-provider-class>/<uuid>`。接下来，该目录下的工作负载 UID 目录与前述的映射路径进行绑定，确保任何对此目录的修改都会同步至 `/user-defined-root-path`。

**步骤 5**: Kubelet 启动用户 Pod。

**步骤 6**: 当用户发送动态挂载请求（即创建一个 DynamicVolumeClaim（CR））时，CSI Node Service 将在对应节点上生成一个 `DMP`。这个 `DMP` 容器的 `/xvols` 目录被挂载到了步骤 4 分配的专属目录 `/var/lib/dynamic/xvols/<storage-privider-class>/<uuid>/<podUID>`，并确保它们之间的传播策略设置为 `Bidirectional`。

**步骤 7**: DMP 的 Informer 负责监听请求变动，并在其 `/xvols` 目录中执行外部存储的挂载或卸载。这样，`/xvols` 下的所有变更都会实时同步到用户 Pod 的 `/user-defined-root-path` 目录。

这个流程使用户能够在其 Pod 内实现动态挂载卷，充分利用了 Kubernetes 提供的 Mount Propagation 能力，实现了存储的高度灵活性和可访问性。

### Concepts

我们的 Operator 提供了 2 个 CRD 来管理动态挂载卷。

#### 1. StorageProviderClass

> 定义了与外部挂载存储的连接详细信息


`StorageProviderClass` 是 Dynamic Volume CSI 驱动程序中**集群级别**的资源，用于向 CSI 驱动程序提供定义了与外部挂载存储的连接详细信息的能力。

下面是 StorageProviderClass 资源的示例：

```yaml
apiVersion: v1
metadata:
  name: provider-s3-secret
  namespace: default
kind: Secret
type: Opaque
stringData:
  access-key-id: cldevaccesskey
  secret-access-key: cldevsecretkey
---
apiVersion: dynamicvolume.csi.cloudnative101.net/v1alpha1
kind: StorageProviderClass
metadata:
  name: s3-sample
spec:
  provider: s3
  parameters:
    # 安全地引用 Secret
    dynamicvolume.csi.cloudnative101.net/storage-provider-secret-name: provider-s3-secret
    dynamicvolume.csi.cloudnative101.net/storage-provider-secret-namespace: default

    # 资源配置（可选）
    dynamicvolume.csi.cloudnative101.net/mount-cpu-request: 10m
    dynamicvolume.csi.cloudnative101.net/mount-memory-request: 50Mi
    dynamicvolume.csi.cloudnative101.net/mount-cpu-limit: 10m
    dynamicvolume.csi.cloudnative101.net/mount-memory-limit: 50Mi
```

Mutating Webhook 将自动为引用 StorageProviderClass 的 Pod 卷进行配置：

```yaml
volumes:
- csi:
    driver: dynamicvolume.csi.cloudnative101.net
    readOnly: true
    volumeAttributes:
      mountPodDispatchPolicy: independent
      storageProviderClass: s3-sample
  name: dynamic-volume-root-path
```

⚠️ 在 CRD 里，不建议直接将敏感信息定义在 spec 中，通常的做法是在 CRD 的 `spec` 中引用一个外部定义的 Secret，以确保安全性和可维护性。

#### 2. DynamicVolumeClaim

>  定义了目标工作负载需要挂载的外部存储卷信息以及挂载后的情况

`DynamicVolumeClaim` 是 Dynamic Volume CSI 驱动程序中**命名空间级别**的资源，用于向 CSI 驱动程序提供定义了目标工作负载需要挂载的外部存储卷信息以及挂载后的情况，同时用来跟踪匹配的 `Dynamic Mount Pod`。

下面是 DynamicVolumeClaim 资源的示例：

```yaml
apiVersion: dynamicvolume.csi.cloudnative101.net/v1alpha1
kind: DynamicVolumeClaim
metadata:
  name: externalvolume-s3-sample
  namespace: dynamic-volume-test
spec:
  # 挂载点配置
  mountpoint: /data
  # S3 存储特定配置
  s3Config:
    bucket: ssat202410181
    endpoint: https://play.min.io
  # 目标工作负载选择器
  selector:
    matchLabels:
      app: app-4-s3
  # 存储提供者
  storageProvider: s3
```

💡 **小贴士**：大家可以通过下面的终端操作录屏，获取详细步骤的直观演示：

<script src="https://asciinema.org/a/692437.js" id="asciicast-692437" async="true"></script>


以上是动态挂载项目的一个粗略介绍，项目深究还有诸多细节值得探讨💡 

🔍 **如何赋予工作负载动态挂载卷的能力？**

思考方向：
* 是通过 namespace 级别全局注入？
* 还是直接在工作负载上进行配置？

🧐 **关键技术点：**
* Dynamic Volume CSI 驱动程序的调度策略
* 用户 Pod 意外重启的处理机制
* DMP 意外重启的应对方案

欢迎对动态存储挂载感兴趣的技术爱好者 🤝 一起交流探讨！


## 写在最后

本方案通过 Operator 模式和 CSI Driver，实现了 Kubernetes 存储管理的新范式。它不仅提供了动态、灵活的存储挂载能力，还秉承了云原生的设计理念。希望这一思路能为大家在云原生环境下的存储管理提供新的启发。
