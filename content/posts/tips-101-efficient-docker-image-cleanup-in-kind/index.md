---
keywords:
- cloud native
- cloud native 101
- kubernetes
- Kind
title: "Cloud Native Tips 101: 高效清理 Kind 集群中的 Docker 镜像"
description: "本文提供了一种简便的方法来清理 Kind 集群开发中产生的冗余 Docker 镜像，帮助开发者有效释放磁盘空间，优化本地开发环境。"
date: 2023-10-25T08:18:00+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Cloud Native Tips 101
- Kind
---

![cover](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/efficient-docker-image-cleanup-in-kind/40a02f5f-10e3-485e-b05e-fd928e8459d0.png)

随着云原生技术的兴起，Kubernetes 在日常开发中的应用越来越广泛。为了满足本地开发和测试的需求，kind 作为一个允许在 Docker 容器内本地运行 Kubernetes 集群的工具，受到了许多开发者的喜爱。

但随之而来的一个小问题是，kind 在开发过程中可能会产生大量不必要的镜像，占据大量的磁盘空间。今天，我们就来讲解如何高效地清理这些镜像。

## 背景

在 `kind` 集群的日常开发中，为了加载特定的 Docker 镜像到集群中，我们经常会使用如下的命令：

```bash
kind load docker-image --name dvcd rancher/k3s:v1.25.14-k3s1
```

使用这个命令的副作用是，每次导入镜像时，`kind` 都会在容器内产生一个 `import-日期` 格式的镜像，如：

```bash
docker.io/library/import-2023-10-19      <none>     2d201b25ca02a       133MB
```

当我们频繁导入镜像时，这些 `import` 镜像会迅速堆积，从而占据大量的磁盘空间。

## 解决方案

针对上述问题，我们可以采取以下三个步骤进行清理：

### 1. 使用 `ctr` 清理 `import` 镜像

假设我们的容器名称为 `dvcd-worker`，我们可以执行以下命令删除所有名字中含有 “import”的镜像：

```bash
CONTAINER_NAME=dvcd-worker
KEYWORD="import"
docker exec -it ${CONTAINER_NAME} sh -c "\
ctr -n k8s.io images ls | \
grep $KEYWORD | \
awk '{print \$1}' | \
xargs -I{} ctr -n k8s.io image rm {}"
```

该命令的核心部分是通过 `ctr` 列出所有的镜像，然后使用 `grep` 筛选出名字中包含 "import" 的镜像，接着通过 `awk` 获取每个镜像的 ID，并使用 `xargs` 将它们传递给 `ctr` 进行删除。

{{< alert "tips-2" >}}
当然，如果你需要批量清理其他的镜像，只需修改关键字即可。
{{< /alert >}}

### 2. 重启容器

清理完毕后，为了确保资源被正确释放，我们需要重启相关的 Docker 容器：

```bash
docker restart ${CONTAINER_NAME}
```

### 3. 清理 `<none>` 镜像

清理 `import` 镜像后，我们还需要进一步清理被标记为 `<none>` 的镜像，命令如下：

```bash
docker exec -it ${CONTAINER_NAME} sh -c "\
crictl images | \
awk '/<none>/ {print \$3}' | \
xargs -I{} crictl rmi {}"
```

此命令使用 `crictl` 工具列出所有镜像，使用 `awk` 进行筛选并获取 `<none>` 镜像的 ID，然后使用 `xargs` 将它们传递给 `crictl` 进行删除。

## 总结

通过上述方法，我们可以轻松清理 `kind` 中积累的 Docker 镜像，有效释放磁盘空间。在后续的“云原生小技巧”系列中，我们将继续为大家带来更多实用的内容，敬请期待！
