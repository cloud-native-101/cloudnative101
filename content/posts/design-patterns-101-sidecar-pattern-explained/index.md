---
keywords:
- cloud native
- cloud native 101
- kubernetes
- docker
- sidecar container
title: "Kubernetes Patterns 101: 边车模式详解，解锁容器设计的新视角"
subtitle: "一文告诉你什么是 Kubernetes 容器设计模式之边车模式"
description: 本文深入探讨了 Kubernetes 中的边车模式，这是一种用于增强主应用功能的设计模式。文章探讨了其在不同系统架构中的实现，特别强调了在微服务治理和 Service Mesh 领域的应用。
date: 2021-11-02T08:18:00+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Kubernetes
- Sidecar
- Docker
- Kubernetes Patterns 101
---


![cover](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/sidecar-pattern/ced36491-5dd9-41f4-9d6d-df5a8deacfe2.png)

## Overview

顾名思义，边车用大白话来讲就是加装在摩托车旁边，用来达到拓展现有功能的能力，可以让其坐上更多的人或物。边车很像软件工程里的代理模式，对服务进行包装，使其不改变原来的功能，拓展原来的服务。

现在微服务盛行，技术栈五花八门，其中最让人头疼的就是服务治理了，而 `Sidecar` 模式的出现，正好为服务治理提供了一种解决方案。

本篇文章的重点并不是服务治理，微服务的服务治理太难了，仅仅通过一两篇文章是讲不完了，后续我会专门开个系列来慢慢展开。

今天主要是带大家来开车的，一辆带边的三轮摩托车。。。

## Architecture

在讲 `Sidecar` 模式之前，不妨先来回顾一下系统架构的演进历程，从一步步的演进历程来看，或许我们对 `Sidecar` 模式的理解会更加的深刻。

下面主要以 `日志收集` 和 `服务健康检查` 这两个附加能力，来举例说明在不同时期 `Sidecar` 模式的实现方式。

### 1. VM

> 我们早期是从物理服务器，通过虚拟化技术演进为虚拟机的，下面这个图片可以说是 `远古时代` 的架构。

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/sidecar-pattern/ce9768bd-c4c6-4636-bf93-b99b4870dbaf.png"
    alt="远古时代"
    caption="远古时代"
    >}}

在`远古时代`的架构里，所有的服务都是以进程的方式逐一部署在 `VM` 里的，另外两个 `Logs Progess` 和 `Health Check Progress` 就相当于 `Sidecar` 的能力。

大家可以想象一下，异构系统架构在当时的条件下，运维工程师面对众多不同技术栈的 `VM B`、`VM C`、 `VM D` 以及没有尽头的 `VM XXX...`，他们需要做大量重复的工作，可以想象这种部署方式对于运维工程师来说是多么的奔溃。

### 2. Docker

> 通过容器化技术演进为目前的 `container`，从而进入`容器时代`。

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/sidecar-pattern/0b7e5001-9fee-46c2-accd-d6dea9a66855.png"
    alt="容器时代"
    caption="容器时代"
    >}}

我们从上面的架构图可以看出，虽然以容器的部署方式解放了运维工程师的工作量，但是这种容器设计模式其实是不被允许的，它将所有 `Sidecar` 的能力和主应用程序打包在了一起，变成了一个富容器。容器应该是解决某个问题的功能单元，让容器保持单一用途才是正确的方式。

其实在这个`容器时代`，我们经历了 `2` 个阶段：

#### 第一阶段

所有服务治理相关的功能，比如 `限流熔断`、`流量控制`、`服务限流` 等等，都是和应用程序紧耦合在一起的，也就是说，我们的业务逻辑混合了各式各样非业务功能的代码，当时造成了开发人员很大的心智负担。

特别是在当业务逻辑功能没问题，因为加这些非功能性代码出了 `bug`，我们排查 `bug` 的过程往往是奔溃的。

令人更加奔溃的是，我们还是公司还是采用了不同的技术栈，`Golang`、`TypeScript` 和 `Java` 都有涉及。

#### 第二阶段

我们吸取了 `第一阶段` 的教训，做了 `2` 件大事。

1. 一个是为了简化开发将这些非业务功能性代码做了剥离，将提供的能力代码拆分为独立的类库，有部分采用了开源的成熟框架，使其和业务代码做集成。
2. 另外一件事是重构，将`部分服务`统一往 `Golang` 上迁移。

#### 问题

系统在线上跑了几个月，顺利运行，好像一切很美好的样子对吧？但是，我们仔细想想其中还是存在很多问题的：

1. 代码是侵入式的
2. 不同类库和不同的开源框架，需要考虑团队的学习成本
3. 技术栈难统一，新项目还好，可以直接统一用一种语言来实现。运行已久，且比较核心的项目其实很难去动它，代价非常高，大家都懂的(没人敢动)
4. 迭代升级困难，比如有一个类库出了问题，涉及到的每个服务镜像全部都需要重新去做构建，这个工作量还是需要去衡量的

### 3. Kubernetes

> 容器编排大战后，由 `Kubernetes` 胜出，衍生出 `Pod` 的概念

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/sidecar-pattern/292f96c2-3de2-4dd7-91a4-159daf66bdc5.png"
    alt="微服务时代"
    caption="微服务时代"
    >}}

我们从上面的架构图可以看出，`Kubernetes` 的 `Pod` 在容器的基础上，做了更高一层的抽象。一个 `Pod` 內可以有多个容器存在，它们共享了同一个 `Network Namespace`，并且可以声明共享同一个 `Volume`

左边是主应用程序所在的容器，右边两个是 `服务治理` 相关的辅助容器，也就是我们本篇文章的主题，可以称之为 `Sidecar` 容器。

我们可以在一个 `Pod` 中，启动一个或多个辅助容器，来完成一些独立于主容器之外的工作。这种模式的好处是显而易见的，每个容器都有它自己单一的用途，真正做到了职责分离，可以说是基本上解决了上述 `容器时代架构` 遗留下的大部分问题。

像上面提到的富容器打包模式，这种一个容器下跑多个功能不相干的进程时，在 `Kubernetes` 的世界里，我们自然而然地想到它们是不是应该被描述成一个 `Pod` 里的多个容器才是最合理的。

现代应用在 `Kubernetes` 里面的形态就是 `Pod` = `App Container` + `Sidecar Container`

## Sidecar Pattern

看完上面系统架构的演进历程，我相信大家心里面对 `Sidecar Pattern` 都有了自己的认识了吧，这种设计模式其实出现的很早，只是每个阶段对它实现的方式都不一样而已。

但是本质都是一样的，主应用和附加能力的应用它们拥有共同的生命周期，要么在同一个`VM`里，要么在同一个`容器`內，或者是在同一个`Pod`內。

`Sidecar Pattern` 可以说是现代云计算非常重要的设计模式，它通过职责分离与容器的隔离特性，能够降低容器的复杂度，同时能扩展并增强已有容器的功能。

> `Kubernetes` 的 `Init Container Pattern` 虽然也是关注点分离，但是它主要是做一些依赖准备性的工作，在应用容器启动前会成功退出。有需要的话可以点击文中链接回顾下 `Init Container` 这种容器设计模式。
>
> [一文告诉你什么是 Kubernetes 容器设计模式之初始化容器](https://mp.weixin.qq.com/s/ZXNnv83UpcFgAoA1Bok9cQ)

{{< article link="/posts/design-patterns-101-init-container-explained/" >}}

### 优势

1. 不仅对原来的应用代码零侵入，而且不限制原来应用的语言，特别适合异构微服务的场景
2. 开发只需专注业务代码实现，开发成本更低
3. 减少业务开发人员的心智负担，使其沉浸在业务中，只需关注业务能力的实现
4. 特别是对`老系统`，`Sidecar Pattern` 的价值更大

## Use cases

我们用 `Pod` 內的容器会共享 `Volume` 这个特性来举个例子，这个其实和 [一文告诉你什么是 Kubernetes 容器设计模式之初始化容器](https://mp.weixin.qq.com/s/ZXNnv83UpcFgAoA1Bok9cQ) 文中的`Scenario 03` 提的例子非常像，我稍微对其做了一下调整，下面是这个例子的一个架构图。


{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/sidecar-pattern/91e7af10-b245-46a7-ae61-8525ab5c8f60.png"
    alt="HTTP 前端部署"
    caption="HTTP 前端部署"
    >}}

我们再来看下这个 `Pod` 的 `YAML` 结构，如下所示：

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-example
spec:
  containers:
  - name: application-container
    image: nginx:1.15.2
    imagePullPolicy: Always
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-files
      mountPath: /usr/share/nginx/html

  - name: sidecar-container
    image: lqshow/busybox-curl:1.28
    # 你可以想象成你的前端应用的静态文件全部打包在 /project/dist 目录下
    command: ['/bin/sh', '-c', "echo 'Hello, World!' > /project/dist/index.html && sleep 3600"]
    volumeMounts:
    - name: shared-files
      mountPath: /project/dist
    workingDir: /project/dist

  volumes:
  # 通过同一个卷来共享数据，用来共享 Sidecar 静态文件
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

再验证访问结果，是否有达到我们的预期

```bash
➜ curl localhost:3000
Hello, World!
```

从这个例子我们可以看出，它对容器的 `可重用性` 利用的非常好，构建 `Web 服务` 就该这样子。应用开发者根本没必要将 `Nginx` 打包装进自己的容器里。有句老话说的好，让专业的人做专业的事，其实做通用性容器也是这个道理。

## Service Mesh

`Sidecar Pattern` 的使用越来越普遍，尤其是在 `Service Mesh` 领域， 它可以说是将 `Sidecar Pattern` 玩出了极致，且非常符合云原生的理念。

接上之前提的，系统架构的演进历程，`Service Mesh` 可以说是后微服务时代了。且随着系统复杂性的增强，可以说 `Service Mesh` 重新定义了微服务治理。

<center>
{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/sidecar-pattern/565b92d2-3ddb-4e2c-aa5a-a6da3f07af8c.png"
    alt="后微服务时代"
    caption="后微服务时代"
    >}}
</center>

> 本篇文章不对 `Service Mesh` 做阐述
>
> 具体详见：[Pattern: Service Mesh](https://philcalcado.com/2017/08/03/pattern_service_mesh.html "Pattern: Service Mesh")

## Wrapping Up

读到这里了，相信大家已经非常清楚 `Sidecar Pattern` 都能解决什么问题了吧，我来做个小结吧。

`Sidecar Pattern` 真正做到了让 `控制`与`逻辑` 的分离，沿用上面例子的话，它让专业的人去做专业的事成为可能，而且使开发一个`高内聚`、`低耦合`的软件变的更加容易。

后微服务时代，可以说是颠覆了传统的开发和运维模式，不仅对开发工程师要求高了，对运维工程师也是一个不小的挑战。
