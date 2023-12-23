---
keywords:
- cloud native
- cloud native 101
- Kubernetes
- Helm
- Chart
title: "Cloud Native 101: 一文掌握如何高效上云，开启云原生之旅(下)"
subtitle: "一文告诉你怎么将应用搭上云原生这趟便车（下）"
description: 本文详细介绍了 Helm 作为 Kubernetes 包管理工具的优势、核心概念和使用方法，为开发者提供简化云原生应用部署的实用指南。
date: 2022-01-30T17:30:00+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Kubernetes
- helm
---

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/101-efficient-cloud-migration-journey/d0e1819c835701b0bf0aab2377df4982.jpeg"
    alt="cover"
    caption="cover"
    >}}
    
我在前面的 [文章](https://mp.weixin.qq.com/s/Dj5a4BANeuae-QHqWcoCnQ) 里提到过，假如将一个无状态应用对外提供服务，至少需要 5-6 个细粒度的 Kubernetes 资源。

{{< article link="/posts/101-efficient-cloud-migration-journey-part-1/" >}}

我们作为开发者，在 Debug 阶段，重复使用 apply 命令勉强还是可以接受的。但是，如果我们将这样的资源直接交付到运维手里，这样的做法是不建议的。为什么呢？后面我会详细去说。

我们知道在 Linux 的世界里，安装一个软件是非常简单的，比如我在 CentOS 下要使用 Nginx，只需要敲下 `yum install -y nginx` 这个指令，即可一键完成安装。

既然 Kubernetes 已经成为了事实上的云原生分布式操作系统，那么在它的生态里，有没有类似的安装方式呢？好问题，有的。它正是本篇文章要分享的主题 [Helm](https://helm.sh/ "Helm")。

让我们重新回到上面提到的那个例子，如果在 Kubernetes 世界里需要安装一个 Nginx ，我们来看下在 Helm 的加持下，怎么来完成这件事呢？

很简单，只需要 2 条指令即可完成安装。

```bash
# 1. 首先在本地添加一个 chart 仓库
helm repo add bitnami https://charts.bitnami.com/bitnami
# 2. 使用命令运行一个 nginx 实例
helm install my-release bitnami/nginx
```

我们先不管它的安装结果如何，使用这样的命令来安装一个 Kubernetes 应用，使用者基本上不需要去关心你应用的背后到底有哪些资源组成，相比 apply 一大坨资源文件来说是不是方便太多了？

这是 Helm 最直观的好处，就是`对使用者友好`，他们不一定要有很多 Kubernetes 的专业知识，就可以将一个复杂的应用部署起来。

读到这里了，你是否对这个工具产生一点兴趣了呢？

## 那么，什么是 Helm?

从使用者的角度来说，它好比 Ubuntu 上 的 apt，又或者是 CentOs 上的 yum 命令，它可以让你安装 Kubernetes 应用变得非常的容易。

Helm 把各种 Kubernetes 的编排文件通过模板化，参数化的形式抽离，大大简化 Kubernetes 上的云原生应用程序的部署，同时还支持发布和回滚版本控制。

一句话，它就是 Kubernetes 的包管理工具。

> 以下是官方对 Helm 做出的解释
>
> Helm 是查找、分享和使用软件构建 Kubernetes 的最优方式。

## 为什么要使用 Helm?

这里我先来解答下文章开头的疑问，为啥不建议直接交付细粒度的资源。原因其实很简单，all-in-one 这种方式它并不能友好的处理多环境部署问题，无论你用 shell 脚本怎么去做适配，总体感觉就是很难用。<br/>另外一个很原因也很直白，就是没法支持版本。

我们知道 Kubernetes 虽然解决了容器编排这个问题，但它同时也带来了各种复杂性，不可否认，在 Kubernetes 上部署或者说交付一个应用，确实是一件挺难的事情。而且作为一个 YAML 工程师，平时经常和各种 Kubernetes YAML 资源打交道，长期下来难免也会枯燥乏味。

Helm 可以说是正好解决了以上这些问题，它可以将特定于环境或部署的配置提取到单独的文件中，从而实现在 Development、Staging、Pre-production、Production 不同环境中只需要一套 Chart 就能很方便的完成部署。

以下可以说是我从使用 Helm 获得的最大好处

1. 降低了应用程序部署的复杂性
2. 应用程序可在不同的复杂环境中管理
3. 简化持续交付的过程
4. 提高工作效率

## Helm 的核心概念

Helm 有 3 个重要的核心概念

### #1. Chart

一个 Helm 包，包含了运行 Kubernetes 一个应用实例所需要的镜像、依赖和资源定义等，它是描述一组相关 Kubernetes 资源的文件集合。

### #2. Release

在 Kubernetes 集群上运行的 Chart 的一个实例，每次安装都会生成一个 Release，每个实例会有自己的 Release 名称。

### #3. Repository

Repository 它是用来发布和存储 Chart 的仓库，同时可以很方便 `分享`你的 Chart。

所以对某些阶段来说，它并不是必须的，你做好一个 Chart 后，完全可以通过本地化的方式来验证和部署。

但是 Chart 的规模上来后，Repository 这种集中化管理的方式，就会变的非常有必要，尤其是在企业内部。

## 如何调试 Chart?

作为一个 Chart 的开发者，必须要保证自己创建的 Chart 是可用的，Helm 默认提供了一些工具来帮助开发者。

下面是 Debugging Templates 时比较常用的命令，大家可以了解一下。

### #1. helm template

开发者在开发新的 Chart 时，如果不确定自己写的 Chart 是否正确，可以在本地做一下模拟安装，如果书写有错误的话，会有提示，只需根据提示一步步调整模板即可。

```bash
helm template --debug $(pwd)/my-chart
```

另外该命令会在本地显示生成所有的资源清单，可以通过此方法来做检查自己做的 Chart 是否符合预期，并可以开启一些开关，做进一步确认。

```bash
helm template --debug \
  --set foo.enabled=true \
  --set bar.enabled=true \
  $(pwd)/my-chart
```

### #2. helm get values

当需要确认安装到集群实例具体的配置时，通过这个命令能够定位到信息。

比方说，当前实例中有些资源并没有和预期一样被输出，可以通过该命令，来确认一下对应的一些开关是否被开启。

```bash
helm get values <RELEASE_NAME>
```

### #3. helm get manifest

如果你部署的实例有异常或者需要检查部分资源，可以直接通过以下命令，快速检索出该实例所有的资源清单，而不需要针对具体的某个资源做查找。

```bash
helm get manifest <RELEASE_NAME>
```

### #4. helm lint

在将 Chart 推送到仓库之前，需要验证下制作的 Chart 是否正确无误，验证方法如下。

```bash
helm lint $(pwd)/my-chart
```

## 一些实践经验

下面来介绍下我们使用 Helm 的一些经验吧。

### #1. 关于缩进

因为 chart 是 YMAL 编写的，当嵌套层次比较多时，各种缩进将会成为一个问题。

我们其实可以通过辅助模板的方式配置在 \_helpers.tpl 文件中，然后利用 include 方法来引入这个模板，同时结合 nindent 关键字避免做手动间距。

#### \_helpers.tpl

```yaml
{{- define "mychart.projectedVolumeMounts" -}}
- name: migration-files
  mountPath: /data/migrations
- name: projected-volume
  mountPath: /data
  readOnly: true
{{- end -}}
```

#### job.yaml

```yaml
...
spec:
  template:
    ...
    spec:
      ...
      containers:
      - volumeMounts:
        {{- include "mychart.projectedVolumeMounts" . | nindent 8 }}
```

### #2. ConfigMap 变更

因为 Kubernetes 声明式 API 这种模式，如果一个资源对象没有做任何调整，集群是不会做出任何变更的。

比方说我一个服务使用了 ConfigMaps 这个资源，通常情况下 ConfigMaps 作为配置文件注入容器中，有时升级可能只需要在 ConfigMaps 中修改或加入一些新的配置。
但是对引用了 ConfigMaps 对象的控制器资源，比方说我用到了 Deployment，我并没有对 Deployment 的数据做出变更，应用程序还会继续使用旧的配置运行，从而和预期不符。

当然你可以人工介入，通过手动删除服务 Pod 做重启来达到目的，但这个是不建议的，这种方式服务会发生中断，最好能通过滚动更新来完成，且它不自动化。

那么通过 Helm 的话，怎么做呢？

我们可以在 Deployment spec 里加入以下资源完成自动更新，利用 sha256sum 函数，确保在另一个文件更改时更新部署的 annotations 的 spec，从而自动触发滚动更新

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

但是需要注意的是，这种处理方式，只适合引用当前 chart 内的 ConfigMaps。不建议公共的 ConfigMaps 也这样处理，因为一旦被引用的公共模块做了错误的更新，所有引用的 Deployment 或者其他控制器，都会自动做滚动更新，错误的配置会使你所有服务整个宕掉，你懂得。

另外，Secret 的变更处理方法同上

### #3. Database migrations

我相信大家将应用往 Kubernetes 迁移的时候，肯定有为如何将应用依赖的数据库迁移集成到部署中头疼过吧？

目前我们实践下来，最有效的方式就是通过 Job 来做这个迁移的事情, 因为数据库升级显而易见是一次性的工作。

> 我们可以通过 Helm hooks 结合 Job 这个工作负载将 Database Migration 集成到应用的部署中。
> 大家可以读一下下面这篇文章，对该方案做了详细的说明，我这里就不再多做描述了。
>
> [Database migrations on Kubernetes using Helm hooks](https://itnext.io/database-migrations-on-kubernetes-using-helm-hooks-fb80c0d97805 "Database migrations on Kubernetes using Helm hooks")

### #4. [Don't repeat yourself](https://en.wikipedia.org/wiki/Don't_repeat_yourself "Don't repeat yourself")

我们在创建 Chart 的时候，会碰到一些通用的配置模块。
打个比方，有两个使用不同工作负载的 Chart，同时用到了配置完全相同的 ConfigMap 或者 Secret 资源。

因为每个 Chart 定义的模板或者提取的配置变量，作用范围都在自己的 Chart 内，所以这种情况会导致很多重复的东西。

Helm3 引入的 [Library Charts](https://helm.sh/docs/topics/library_charts/ "Library Charts") 可以很好的处理这个问题，避免出现一次调整，到处去处理相同模块的数据。

### #5. 如何在 CI/CD 做集成

很简单，你可以在任意熟悉的集成工具里利用 helm 提供的现成命令来完成这件事情。

> 可以用 helm upgrade 指令结合 --install 参数来完成。
> Helm 会检查当前 release 是否已经安装，如果没有，会执行安装；如果存在，会进行升级
> 另外 `--atomic` 参数也非常关键，它可以保证升级期间操作失败时可以回滚

```bash
helm upgrade -i $(RELEASE_NAME) \
 -f $(VALUES_FILE) \
 --set-string image.repository=$(REPOSITORY) \
 --set-string image.tag=$(APP_VERSION) \
 --namespace $(RELEASE_NAMESPACE) \
 $(CHART_NAME)
```

表格内是对以上变量的说明
| Field | Desc |
| :---------------- | :---------------------------------------------------- |
| RELEASE_NAME | 实例名称 |
| VALUES_FILE | 部署配置文件 |
| REPOSITORY | 应用程序镜像仓库地址 |
| APP_VERSION | 应用版本号（每次 git commit 动态获取） |
| RELEASE_NAMESPACE | 实例所在的命名空间 |
| CHART_NAME | chart 名称，可以是本地目录，也可以是 Chart Repository |

## Helm 有什么缺点？

讲了这么多，我都在提使用 Helm 的各种优点。它在实际应用过程中真是这样的吗？

显然不是的，我们甚至为此付出过相当大的代价。

### #1. 学习曲线

Helm 虽然简化了 Kubernetes 应用的管理，但是创建一个 Chart 还是有点复杂的，它的学习曲线虽然不能说很陡峭，但是也并不容易，至少我个人使用下来还是需要一段时间去适应的。

在我看来，熟练编写一个有味道的 Chart，最快的方式还是多参考一些优秀的案例，将他们的套路直接转换成自己的私货。

### #2. 审核 Chart 的难度

另外一个比较痛苦的就是对 Chart 做 Review。

因为 Chart 是 YMAL 编写的，当嵌套层次比较多时，各种缩进会成为一个问题，所以审核光靠肉眼完全是不行的，一开始会在上面花费一定的时间。

我们甚至出现过，MR 审核通过了，到上线时才暴露出问题的情况。

### #3. 预定义配置选项

不可否认，Helm 它这种配置提取，参数的抽离形式非常好。

但是，必须要说但是。

在我看来，Helm 最大的痛点也是在于此，它需要事先对所有配置选项做好预先定义。什么意思呢，简单来说，就是在 Chart 中没有被事先定义的内容是无法做更改的。

所以，我们会通过结合这些自定义选项，为`应用定制`出各种复杂的模板。

> 好在社区有了 [Kustomize](https://github.com/kubernetes-sigs/kustomize "Kustomize")，它可以和 Helm 做很好的结合。这里分享一篇 Helm 和 Kustomize 结合的文章，大家可以读一下。
>
>
> [Helm Is Not Enough, You Also Need Kustomize](https://itnext.io/helm-is-not-enough-you-also-need-kustomize-82bae896816e "Helm Is Not Enough, You Also Need Kustomize")

## 推荐阅读

- [Helm documentation](https://helm.sh/docs/ "Helm documentation")
- [Application and package management using Helm](https://docs.microsoft.com/en-us/learn/modules/aks-app-package-management-using-helm/ "Application and package management using Helm")
- [Helm 3, the Good, the Bad and the Ugly](https://banzaicloud.com/blog/helm3-the-good-the-bad-and-the-ugly/ "Helm 3, the Good, the Bad and the Ugly")
- [Deploy Kubernetes Helm Charts](https://github.com/roboll/helmfile "Deploy Kubernetes Helm Charts")

## 写在最后

`Helm` 虽然有些许槽点，但是不可否认的是它确实是一个非常优秀的工具，在它的加持下你可以很方便的将应用部署到 Kubernetes 集群，可以说它让你离云原生又近了一步。

那么，应用是否只要上了 Kubernetes 它就是云原生应用了呢？

别着急，我先给你浇盆冷水，这才刚刚搭上车而已，后续我会继续分享云原生应用相关的内容，期待你的关注。
