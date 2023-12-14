---
keywords:
- cloud native
- cloud native 101
- kubernetes
- init container
title: "Kubernetes Patterns 101: 初识初始化容器，探索容器设计的精髓"
subtitle: "一文告诉你什么是 Kubernetes 容器设计模式之初始化容器"
description: 探索 Kubernetes 的初始化容器设计模式：深入了解它们的工作原理、生命周期和与普通容器的区别。通过实际案例，本文为开发者和架构师揭示其在云原生应用中的重要应用和价值。
date: 2021-10-23T16:30:00+08:00
lastmod: 2021-11-02T20:07:04+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Kubernetes
- Init Container
- Kubernetes Patterns 101
---

![cover](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/sidecar-pattern/ced36491-5dd9-41f4-9d6d-df5a8deacfe2.png)

# Overview

在这个微服务架构盛行的时代，开发人员设计应用程序都会遵守 `高内聚`、`低耦合` 的原则，同样 `Kubernetes` 世界里也有它的设计模式，本篇文章主要对初始化容器展开讨论。

我们知道在 `Kubernetes` 世界里，`Pod` 才是 `Kubernetes` 项目中的原子调度单位，而不是 `Container`， `Container` 只是 `Pod` 众多属性里的一个普通的字段。

但凡和调度、存储、网络，以及安全相关的属性，基本上都是 `Pod` 级别的，另外 `Pod` 在 `Kubernetes` 项目里还有一个更重要的意义，它是实现 `容器设计模式` 的核心机制。

你可以简单的理解为 `容器设计模式` 主要是为了分离应用程序中的关注点，换句话说是为了职责分离。我们通常会把不同功能的应用分别放在不同的 `Container` 中，从而遵守一个容器一个服务的原则（`单一职责原则`）。

## Pod Template

从 `Pod` 的 `Spec` 中可以发现，`Spec` 定义里面其实有两个和 `Container` 相关的字段，分别是 `initContainers` 和 `containers`，如下所示 `Pod` 內的容器分为两种：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: container-example
spec:
  initContainers:
  - name: init-container-1
    image: lqshow/busybox-curl:1.28
    ...
    - name: init-container-2
    image: lqshow/busybox-curl:1.28
    ...
  containers:
  - name: container-1
    image: lqshow/busybox-curl:1.28
    ...
  - name: container-2
    image: lqshow/busybox-curl:1.28
    ...
```

那么这两种 `Container` 有什么区别呢？顾名思义，从命名上也比较容易的看出来，`spec.initContainers` 定义的容器，会比 `spec.containers` 先启动。

但它是一种比较特殊的容器，两者拥有的生命周期是不同的，本篇文章主要关注 `Init Containers`。

## Init Containers

首先如上所述，`Init Containers` 里的容器是在 `Pod` 内的应用容器启动之前运行的，但是和 `spec.containers` 不同的是，`spec.initContainers` 定义的容器会按照顺序逐一执行，只有等到 `Init Containers` 全部执行完后，主应用容器才开始启动。

我们可以将 `Init Container Pattern` 理解为面向对象编程语言中，`构造函数` 的概念。

顺便提一句， `spec.containers` 里的应用程序容器稍有不同，尽管 K8s 按照 spec.containers[] 数组中的顺序创建和启动容器，但是数组內容器之间是平等无序的。什么意思呢？我们知道容器的启动完成并不等同于容器已经准备好对外提供服务了，会存在 Pod 中的多个容器并行运行的情况，这就意味着默认情况下我们不能依赖于一个容器在另一个容器之前启动。

![执行顺序](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/init-container/3dbc9037-fc8d-44b8-8625-ee03d3bb5814.png)

我们先来看一个完整的 `Pod` 的 `YMAL` 结构，如下所示：

> 以下是参考官方例子做的调整：[Kubernetes documentation(Init Containers)](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/ "Kubernetes documentation(Init Containers)")

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-containers-pattern-example
spec:
  initContainers:
  - name: first-container
    image: lqshow/busybox-curl:1.28
    command: ['sh', '-c', "echo waiting for first-container; sleep 2;"]
  - name: second-container
    image: lqshow/busybox-curl:1.28
    command: ['sh', '-c', "echo waiting for second-container; sleep 5;"]
  containers:
  - name: app-container
    image: lqshow/busybox-curl:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
EOF
```

将脚本放在 `Kubernetes` 集群內执行，观察 `Pod` 的启动情况，从 `STATUS` 这一列可以看出它是按顺序逐一先启动 `2` 个 `Init Container`，最后才将主 `Container` 拉起，`Pod` 最终处于 `Running` 状态。

```bash
➜ kubectl get pod -w|grep example

NAME                                         READY   STATUS             RESTARTS   AGE
init-containers-pattern-example              0/1     Init:0/2           0          1s
init-containers-pattern-example              0/1     Init:1/2           0          4s
init-containers-pattern-example              0/1     Init:1/2           0          5s
init-containers-pattern-example              0/1     PodInitializing    0          9s
init-containers-pattern-example              1/1     Running            0          10s
```

我们通过查看 `Pod` 的详细信息，确实一切也符合预期。如果 `Init Container` 有多个应用会依次启动，只有一个运行成功了，才会启动下一个，等所有 `Init Container` 都运行结束了，主应用才会启动。

- `first-container` 执行 `2` 秒后结束
- `second-container` 执行 `5` 秒后结束
- `app-container` 在上述两个 `Init Container` 结束后 (`Fri, 22 Oct 2021 23:10:03 +0800`) 才开始启动

```bash
➜ kubectl describe pod init-containers-pattern-example

Name:         init-containers-pattern-example
Namespace:    default
Start Time:   Fri, 22 Oct 2021 23:09:53 +0800
[...]
Status:       Running
Init Containers:
  first-container:
    [...]
    Command:
      sh
      -c
      echo waiting for first-container; sleep 2;
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Fri, 22 Oct 2021 23:09:55 +0800
      Finished:     Fri, 22 Oct 2021 23:09:57 +0800
    Ready:          True
    Restart Count:  0
    [...]
  second-container:
    [...]
    Command:
      sh
      -c
      echo waiting for second-container; sleep 5;
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Fri, 22 Oct 2021 23:09:57 +0800
      Finished:     Fri, 22 Oct 2021 23:10:02 +0800
    Ready:          True
    [...]
Containers:
  app-container:
    [...]
    Command:
      sh
      -c
      echo The app is running! && sleep 3600
    State:          Running
      Started:      Fri, 22 Oct 2021 23:10:03 +0800
    Ready:          True
    [...]
[...]
Events:
  Type    Reason     Age        From                Message
  ----    ------     ----       ----                -------
  Normal  Scheduled  <unknown>                      Successfully assigned default/init-containers-pattern-example to kind-dev-1
  Normal  Pulled     17s        kubelet, kind-dev-1  Container image "lqshow/busybox-curl:1.28" already present on machine
  Normal  Created    17s        kubelet, kind-dev-1  Created container first-container
  Normal  Started    16s        kubelet, kind-dev-1  Started container first-container
  Normal  Pulled     14s        kubelet, kind-dev-1  Container image "lqshow/busybox-curl:1.28" already present on machine
  Normal  Created    14s        kubelet, kind-dev-1  Created container second-container
  Normal  Started    14s        kubelet, kind-dev-1  Started container second-container
  Normal  Pulled     8s         kubelet, kind-dev-1  Container image "lqshow/busybox-curl:1.28" already present on machine
  Normal  Created    8s         kubelet, kind-dev-1  Created container app-container
  Normal  Started    8s         kubelet, kind-dev-1  Started container app-container
```

### 总结

1. `Init Container` 会在主应用启动之前先启动，如果存在多个 `container` 会按顺序依次执行
2. 每个 `Init Container` 成功终止退出后，下一个 `Init Container` 才能够运行
3. 当所有 `Init Container` 都运行完成时，主应用容器才会启动。

## Use cases

在了解了 `Init Container` 的作用后，这里重点介绍一下几个真实世界中利用 `Init Container` 特性的场景案例

### 设置共享卷权限

> 主容器在启动前需要具备一些先决条件，这时就需要执行一些预设脚本

比如通过 `InitContainer` 设置目录权限，它在主容器之前启动，确保文件夹是可写的

```yaml
initContainers:
  - command:
    - sh
    - -c
    - chmod -R 777 /tmp/workspace
    image: busybox:1.31.1
    name: volume-mount-hack
    volumeMounts:
    - mountPath: /tmp/workspace
      name: workspace
volumes:
  - hostPath:
      path: /workspace/c8879f79-fb99-49c2-a484-ca22fafb37e5
      type: DirectoryOrCreate
    name: workspace
```

### 等待依赖服务就绪后启动应用容器

> 检查应用依赖的其他模块是否已经 `Ready`，用来阻塞应用的启动，直到所有外部依赖关系都被满足

如果你的应用程序初始化依赖与很多模块，`Init Container` 在这里是非常合适的。
比如下面这个例子的先决条件是必须先连接上 `mysql`，主应用才能运行。

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: until-the-dependencies-are-ready-pod
  labels:
    app: until-the-dependencies-are-ready
spec:
  initContainers:
  - name: init-mysql
    image: lqshow/busybox-curl:1.28
    command: ['sh', '-c', 'until nslookup mysql.default; do echo waiting for mysql; sleep 2; done;']
  containers:
  - name: app-container
    image: lqshow/busybox-curl:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
EOF
```

### 利用共享卷实现高效的前端静态资源部署

> 每个 `Pod` 內的 `Volume` 是共享的，因此 `Init Container` 里的数据也可以被主容器使用到。我们可以利用在同一个 `Pod` 中的容器数据共享这个机制，可以对应用的部署做一些优化。

通常前端产品的应用都是会被打包成纯静态文件，然后由 `HTTP Server` 去渲染页面。按照传统思维，前端应用的 `Owner` 在做镜像的时候，一般会将 `HTTP Server(比如 nginx)` 和静态文件打包在一起做交付。

其实大可不必这样，一来这样做的镜像基本都是在 `百兆以上`（即所谓的`富容器`），我们可以利用 `Docker` 的二次构建流程只保留静态文件，最后的镜像可以缩减到 `10M` 左右。

> 通过多阶段构建生成 `mini` 镜像，以下是二次构建的伪代码，仅做参考

```dockerfile
FROM node:latest as builder

WORKDIR /data/project

# Install app dependencies
COPY package.json ./
RUN npm install

# Bundle app source
COPY ./ ./
RUN npm run build

FROM alpine:3.14.2

WORKDIR /project/dist
COPY --from=builder /data/project/dist  ./
```

在 `Kubernetes` 世界里，根据 `Init Container` 的容器设计模式，我们可以把 `Nginx` 作为主应用容器，前端项目只提供静态文件，作为 `Init Container` 的输入镜像，而 `Init Container` 只作一件事，就是把静态文件拷贝到一个共享卷中，供主应用容器使用。

这正是 `Kubernetes` 容器编排的魅力所在。

**这样部署有以下 3 个好处**

1. 初始容器可以利用 `Docker` 多阶段构建来生产出更小的镜像，因为只提供静态文件，使部署更快。
2. 将 `Nginx` 作为主容器独立出来，后续升级配置可以统一管控。
3. 解决了 `App` 中静态文件 和 `Nginx` 之间的耦合关系，做到了每个容器职责分离。

> 由于篇幅限制，下面不以真实世界的前端应用做镜像，只展示一个简单数据共享的例子

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-example
spec:
  initContainers:
  - name: app-container
    image: lqshow/busybox-curl:1.28
    # 你可以想象成你的前端应用的静态文件全部打包在 /var/www/html 目录下
    command: ['/bin/sh', '-c', "echo 'Hello, World!' > /var/www/html/index.html"]
    volumeMounts:
    - name: shared-files
      mountPath: /var/www/html

  containers:
  - name: nginx-container
    image: nginx:1.15.2
    imagePullPolicy: Always
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-files
      mountPath: /usr/share/nginx/html

  volumes:
  # 通过同一个卷来共享数据，用于共享 App 静态文件
  - name: shared-files
    emptyDir: {}
EOF
```

将脚本放在 `Kubernetes` 集群內执行，并通过 `port-forward` 来验证结果。

```bash
➜ kubectl port-forward pod/nginx-example 3000:80
Forwarding from 127.0.0.1:3000 -> 80
Forwarding from [::1]:3000 -> 80
```

查看访问结果

```bash
➜ curl localhost:3000
Hello, World!
```

### 错误示例

> 这里举一个 `错误` 使用 `Init Container` 特性的例子

我们的应用程序升级更新时，如果应用程序有涉及到数据库结构的调整，那么我们怎么将数据库结构的变更集成到部署里面呢？有同学想到了使用 `Init Container` 做 `Database Migration`。

这里其实会存在几个问题：

1. 在同时创建多个 `Pod` 情况下，会同时运行多个 `Init Container`，这需要用户的 `Migration` 脚本写的足够健壮，能够规避各种异常情况，同时需要保证幂等性。
2. 如果主容器失败，会导致 `Pod` 重启，此时也会导致所有的 `Init Container` 都需要重新执行。
3. 开发者在 `Debug` 期间，难免会因为一些原因手动去对 `Pod` 做 `Delete` 操作，引发 `Pod` 重启。

所以说此时的场景使用 `Init Container` 的部署模式，并不是一个好的建议，那么如何应对这种场景呢？有的，那就是通过 `Job`, 我们可以通过 `Helm hooks` 结合 `Job` 的方式来将 `Database Migration` 集成到应用的部署中，后续会专门开一个章节详细介绍下该流程。

## Summary

1. `Init Container` 可以理解为面向对象编程语言中，构造函数的概念
2. `Init Container` 会延迟主应用程序的启动
3. `Init Container` 可以很好的贯彻 `单一职责原则`，做到关注点分离，让不同角色更专注于领域知识和能力
4. 每个 `Init Container` 成功终止退出后，下一个 `Init Container` 才能够运行（如果任意一个失败了，主容器不会启动）
5. `Init Container` 失败时将会重新启动（需结合 `restartPolicy` 重启策略），因此需保证代码的幂等性
6. `Init Container` 设计原则，启动脚本尽量短小精悍。如果启动时间很长，考虑将其分解为多个步骤，放入到不同的 `init` 中方便排错。

## Reference

- [Configure Pod Initialization](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/#creating-a-pod-that-has-an-init-container "Configure Pod Initialization")
- [Kubernetes Production Patterns](https://github.com/gravitational/workshop/blob/master/k8sprod.md "Kubernetes Production Patterns")
