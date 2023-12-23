---
keywords:
- cloud native
- cloud native 101
- Kubernetes Shared Memory
- IPC
title: "Kubernetes Shared Memory 101: 一文解析如何提升 K8s 共享内存限制"
subtitle: "一文告诉你怎么提高 Kubernetes Shared Memory 的限制？"
description: 本文详细介绍了在 Kubernetes 中处理共享内存限制的方法，包括配置 emptyDir 挂载和调整 sizeLimit，适合面临内存限制问题的 Kubernetes 用户阅读。
date: 2021-11-21T08:18:00+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Kubernetes
- IPC
---


![cover](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/ipc/e0a33570-b6b1-4bca-a5f4-2ef68f64ccbb.png)

最近有客户反馈用户在平台上跑 `AI` 训练模型时，经常提示内存不够用，刚开始我们以为是用户的应用资源分配不足导致，后来排查下来是因为用户用了`Shared Memory`，是共享内存被写满了。

## 什么是 `Shared Memory`

那么什么是 `Shared Memory`？又是什么情况下需要使用到 `Shared Memory` 呢？

共享内存是 `Unix` 下的多进程之间的通信方式之一，它使得多个进程可以访问同一块内存空间，不同进程可以及时看到对方进程中对共享内存中数据的更新。

由于多个进程共享一段内存，这种方式需要依靠某种同步操作，如互斥锁和信号量等来达到进程间的同步及互斥，可以说 `Shared Memory` 是最有用的进程间通信方式。

> 以上对 `Shared Memory` 的解说摘自 `进程间通信 IPC (InterProcess Communication)` 这篇文章

本文的重点并不是为了要解释什么是 `Shared Memory`，但是稍微了解一下 `IPC` 还是非常有必要的，下面我们来一起定位下问题的原因。

## Docker

我们先来看下 `Containerized Application` 在 `Docker` 容器內的情况

```bash
➜ docker run -it --rm  lqshow/busybox-curl:1.28 sh
/data # df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
shm                      64.0M         0     64.0M   0% /dev/shm
```

从以上结果可以看出，其实 `Docker` 默认分配的共享内存大小就是 `64MB`，官方文档就有提到过。

> Size of /dev/shm. The format is <number><unit>. number must be greater than 0. Unit is optional and can be b (bytes), k (kilobytes), m (megabytes), or g (gigabytes). If you omit the unit, the system uses bytes. **If you omit the size entirely, the system uses 64m**.
>
> 参考：[Docker run reference](https://docs.docker.com/engine/reference/run/ "Docker run reference")

文档中也提到了怎么解决该问题，可以通过设置参数 `--shm-size` 来调整默认的共享内存大小，根据官方的建议我尝试调整后，确实也是起了作用。

```bash
➜ docker run -it --rm --shm-size 256M lqshow/busybox-curl:1.28 sh
/data # df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
shm                     256.0M         0    256.0M   0% /dev/shm
```

## Kubernetes

但是，客户的应用实际上是在 `Kubernetes` 集群內跑的，我们先在集群內尝试执行一下以下脚本，来看下 `Pod` 內的实际运行情况

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
  labels:
    name: hello-world
spec:
  containers:
  - image: lqshow/busybox-curl:1.28
    name: hello-world
    command: ['sh', '-c']
    args:
      - while true; do
          echo hello-world;
          sleep 1;
        done
EOF
```

我们在 `Kubernetes` 创建的 `Pod` 內，发现其共享内存默认也是 `64MB`，和 `Docker` 默认的共享内存的大小值是一样的。

```bash
➜ kubectl exec -it hello-world -- df -h /dev/shm

Filesystem                Size      Used Available Use% Mounted on
shm                      64.0M         0     64.0M   0% /dev/shm
```

我们尝试用 `dd` 命令去写 `100M` 的内容， 系统无法完整的写入，提示 `No space left on device` 错误。

```bash
➜ kubectl exec -it hello-world sh

/data # dd if=/dev/zero of=/dev/shm/output bs=1M count=100
dd: writing '/dev/shm/output': No space left on device
65+0 records in
64+0 records out
67108864 bytes (64.0MB) copied, 0.060007 seconds, 1.0GB/s
```

再次做下确认，发现共享内存其实已经被 `dd` 写满 `64MB` 了。

```bash
/data # df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
shm                      64.0M     64.0M         0 100% /dev/shm
```

从上面的测试可以看出来，`Pod` 里确实也无法使用超过 `64MB` 的 `Shared Memory`，因此导致了客户的 `AI` 训练模型没办法进行下去。

## 解决方案

我们知道 `Kubernetes` 的资源模型中，可以将 `Pod` 中的 `Memory` 资源在 `limits` 中做配置（`spec.containers[].resources.limits.memory`），但是它只对 `Cgroups` 中的 `memory.limit_in_bytes` 起作用，并不会作用到 `Shared Memory` 中。

既然 `Docker` 能通过设置参数 `--shm-size` 来调整默认的共享内存大小，那么在 `Kubernetes` 世界中有没有解决方案呢？

> 官方文档有提到，可以通过将 `emptyDir` 挂载到 `/dev/shm`，并将介质类型设置为 `Memory` 来解决这个问题。
>
> However, if you set the emptyDir.medium field to "Memory", Kubernetes mounts a tmpfs (RAM-backed filesystem) for you instead. While tmpfs is very fast, be aware that unlike disks, tmpfs is cleared on node reboot and any files you write count against your container's memory limit.
>
> 参考：[Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir "Volumes")

通过文档的描述，虽然有不少问题，我们可以一一做下尝试

### #1. 只设置介质类型

我们将以下脚本放在 Kubernetes 集群內执行

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
  labels:
    name: hello-world
spec:
  volumes:
  - name: dshm
    emptyDir:
      # 只设置介质类型
      medium: Memory
  containers:
  - image: lqshow/busybox-curl:1.28
    name: hello-world
    command: ['sh', '-c']
    args:
      - while true; do
          echo hello-world;
          sleep 1;
        done
    volumeMounts:
    - mountPath: /dev/shm
      name: dshm
EOF
```

先看下 `Pod` 內情况

```bash
➜ kubectl exec -it hello-world -c hello-world -- df -h /dev/shm

Filesystem                Size      Used Available Use% Mounted on
tmpfs                    14.7G         0     14.7G   0% /dev/shm
```

再来看这个 `Pod` 所在节点的内存情况

```bash
➜ ssh 138 'free -h'
              total        used        free      shared  buff/cache   available
Mem:            29G         23G        2.6G        1.4G        3.6G         20G
Swap:            0B          0B          0B
```

通过实际运行的情况，我们观察到虽然未设置 `sizeLimit`，但是显示分配了 `HOST` 主机上将近一半的内存。

相信大家也发现了，`HOST` 主机上提示可用内存其实只有 `2.6G` 了，但是 `Pod` 內提示
`Available` 的还有 `14.7G`，显然是有问题的，这个问题我们先放一边，继续做下一个测试。

### #2. 设置介质类型，同时配置 `sizeLimit`

这次我们换成 `Deployment` 资源，将以下脚本放在 `Kubernetes` 集群內执行

```bash
cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      volumes:
      - name: dshm
        emptyDir:
          # 设置介质类型，且配置 sizeLimit
          medium: Memory
          sizeLimit: 256Mi
      containers:
      - name: hello-world
        image: lqshow/busybox-curl:1.28
        command: ['sh', '-c']
        args:
        - while true; do
            echo hello-world;
            sleep 1;
          done
        volumeMounts:
        - mountPath: /dev/shm
          name: dshm
EOF
```

我们通过实际运行的情况，发现即便设置了 `sizelimt` 限制，但是实际显示仍然分配了 `HOST` 主机上将近一半的内存，和第一个测试结果是一模一样的。

```bash
➜ kubectl exec -it $(kubectl get pod -l app=hello-world --no-headers|grep -v "Evicted"|awk '{print $1}') -c hello-world -- df -h /dev/shm

Filesystem                Size      Used Available Use% Mounted on
tmpfs                    14.7G         0     14.7G   0% /dev/shm
```

那么问题来了，这个 `sizeLimit` 配置到底有没生效呢？同样，我们还是用 `dd` 命令尝试去做一些验证性工作。

```bash
➜ kubectl exec -it $(kubectl get pod -l app=hello-world --no-headers|grep -v "Evicted"|awk '{print $1}') -c hello-world sh

# 1. 先尝试直接写满 256 M
/data # dd if=/dev/zero of=/dev/shm/output bs=1M count=256
256+0 records in
256+0 records out
268435456 bytes (256.0MB) copied, 0.520999 seconds, 491.4MB/s
/data #
/data # df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
tmpfs                    14.7G    256.0M     14.4G   2% /dev/shm
/data #

# 2. 再尝试写入 100 M，并没有看到出现 `No space left on device` 的提示
/data # dd if=/dev/zero of=/dev/shm/output2 bs=1M count=100
100+0 records in
100+0 records out
104857600 bytes (100.0MB) copied, 0.227946 seconds, 438.7MB/s
/data #

# 3. 发现成功的写入了 100 M
/data # df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
tmpfs                    14.7G    356.0M     14.4G   2% /dev/shm

# 4. 在一定时间间隔內，Pod 会因为异常被驱逐
/data # command terminated with exit code 137
```

以上的验证很明显了， `sizeLimit` 实际上是起了作用的，`Pod` 最后是以 `137 CODE` 退出了

> `137 code` 基本上都是因为 `OOM` 引起的退出
>
> 参考：[How to troubleshoot Kubernetes OOM and CPU Throttle](https://sysdig.com/blog/troubleshoot-kubernetes-oom/ "How to troubleshoot Kubernetes OOM and CPU Throttle")

以下是被驱逐 `Pod` 的 `Event` 信息，只保留了重要的提示

```bash
...
Events:
  Type     Reason     Age        From               Message
  ----     ------     ----       ----               -------
  ...
  Warning  Evicted    20m        kubelet, kind-dev4  Usage of EmptyDir volume "dshm" exceeds the limit "256Mi".
  ...
```

从上面 `Event` 里可以看出，当前 `Pod` 是被 `kubelet` 驱逐了。

> 当宿主机资源紧张的情况下，`kubelet` 会主动地结束 `Pod` 以回收短缺的资源。具体驱逐 `Pod` 的逻辑可以参见 `github` 代码
>
> [kubelet/eviction/eviction_manager.go#emptyDirLimitEviction](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/eviction/eviction_manager.go#L479 "kubelet/eviction/eviction_manager.go#emptyDirLimitEviction")

### #3. 升级 `Kubernetes` 集群版本

从官方文档给出的提示可以看出，`SizeMemoryBackedVolumes` 这个特性在我们的集群內并没有生效，原因如下

> Note: If the `SizeMemoryBackedVolumes` feature gate is enabled, you can specify a size for memory backed volumes. If no size is specified, memory backed volumes are sized to 50% of the memory on a Linux host.

我们来看下`SizeMemoryBackedVolumes` 这个 `Feature Gates` 的情况

| **Feature**             | **Default** | **Stage** | **Since** | **Until** |
| ----------------------- | ----------- | --------- | --------- | --------- |
| SizeMemoryBackedVolumes | false       | Alpha     | 1.20      | 1.21      |
| SizeMemoryBackedVolumes | true        | Beta      | 1.22      |           |

从以上表格清单可以看出，`SizeMemoryBackedVolumes` 是从 `1.20` 版本才开始支持，而我们客户的集群版本只有 `v1.19.2`

```bash
# 客户的集群环境版本如下
➜ kubectl version --short
Client Version: v1.18.3
Server Version: v1.19.2
```

我们尝试用 `kind` 工具建一个 `1.22.2` 的集群做下测试。

```bash
kind create cluster --name local-k8s --image kindest/node:v1.22.2
```

以下是用 `kind` 安装后，本地 `Kubernetes` 环境的版本情况

```bash
☸️  kind-local-k8s
➜ kubectl version --short
Client Version: v1.18.3
Server Version: v1.22.2
```

将上面的脚本重新运行在 `v1.22.2` 集群內，发现配置确实起了作用

```bash
➜ kubectl exec -it $(kubectl get pod -l app=hello-world -o name |sed 's/pods\///') -- df -h /dev/shm

Filesystem                Size      Used Available Use% Mounted on
tmpfs                   256.0M         0    256.0M   0% /dev/shm
```

而且操作也符合预期

```bash
➜ kubectl exec -it $(kubectl get pod -l app=hello-world -o name |sed 's/pods\///') sh

/data # dd if=/dev/zero of=/dev/shm/output bs=1M count=256
256+0 records in
256+0 records out
268435456 bytes (256.0MB) copied, 0.203033 seconds, 1.2GB/s
/data #
/data # df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
tmpfs                   256.0M    256.0M         0 100% /dev/shm
/data #
/data # dd if=/dev/zero of=/dev/shm/output2 bs=1M count=100
dd: writing '/dev/shm/output2': No space left on device
1+0 records in
0+0 records out
0 bytes (0B) copied, 0.001265 seconds, 0B/s
/data #
```

### #4. 设置介质类型，并配置 sizeLimit，同时加上 `Pod` 的资源模型

我们对 `Pod` 的 `Memory` 资源加上 `limits` 限制

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
  labels:
    name: hello-world
spec:
  volumes:
  - name: dshm
    emptyDir:
      medium: Memory
      sizeLimit: 256Mi
  containers:
  - image: lqshow/busybox-curl:1.28
    name: hello-world
    command: ['sh', '-c']
    args:
      - while true; do
          echo hello-world;
          sleep 1;
        done
    volumeMounts:
    - mountPath: /dev/shm
      name: dshm
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
EOF
```

文章前面也提过`Pod`內的资源模型: `spec.containers[].resources.limits.memory`，主要针对 `Cgroups` 的 `memory.limit_in_bytes` 起作用。

因此以上的配置，对于内存来说，当我们指定了 `limits.memory=128Mi` 之后，相当于将 `Cgroups` 的 `memory.limit_in_bytes` 设置为 `128 * 1024 * 1024 = 134217728` ，如下所见

```bash
➜ kubectl exec -it hello-world -- cat /sys/fs/cgroup/memory/memory.limit_in_bytes

134217728
```

因为受 `limits` 的限制，即便 `sizeLimit = 256Mi`，实际起作用的是 `128Mi`，这也确实合理。

```bash
➜ kubectl exec -it hello-world -- df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
tmpfs                   128.0M         0    128.0M   0% /dev/shm
```

## 总结

综合以上的几个测试，相信大家以后都能解决 `Kubernetes` 平台中 `Shared Memory` 碰到的问题了吧

1. 理想状态下，将客户的 `Kubernetes` 平台升至 `v1.22.2` 或以上稳定版本
2. 将 `emptyDir` 挂载到 `/dev/shm`，并将介质类型设置为 `Memory` ，同时配置上 `sizeLimit`
3. 设置 `Pod` 的 `QoS`
