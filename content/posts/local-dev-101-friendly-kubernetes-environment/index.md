---
keywords:
- cloud native
- cloud native 101
- krew
- ksniff
- kubectx
title: "Kubernetes Local Dev 101: 一文打造本地友好的 K8s 工作环境"
subtitle: "一文告诉你怎么打造对本地友好的 Kubernetes 工作环境"
description: 本文详细介绍了一系列实用工具，如 kubectl、krew 等，以提高 Kubernetes 管理的效率和便利性，适合开发者阅读，以优化他们的工作流程。
date: 2021-12-27T08:18:00+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Kubernetes
---


![cover](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/local-dev-101/2b0c7a18-192e-4464-bec8-bdd545048bf8.png)

作为一个 Kubernetes YAML 工程师，熟练掌握各种资源非常的重要，但是在 Kubernetes 的世界里，概念太多了... 而且它又同时支持通过 [CRD](https://mp.weixin.qq.com/s/6Ud5MxDSaozwoDeXzOIUOA) 来自造概念，扩展已有的资源。

{{< article link="/posts/kubernetes-crd-101-what-is-crd/" >}}

每年又有那么多优秀的创新不断涌现，挡都挡不住。

<center>
    <img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/emoji/so_hard.gif" width="30%" alt="学不过来,怎么办？" />
</center>

但是那又如何，2021 这么糟糕的一年你都熬过来了，再想想那未知的 2022 年吧，这个世界上没有"最"，只有"更"。。。这么一想你是不是要更加(有)焦(动)虑(力)了。。。

所以，吃饭的家伙你不能丢了。

正所谓攻欲善其事，必先利其器，这里我给大家分享一下如何打造一个对本地友好的 Kubernetes 工作环境，相信我，这点真的非常重要，让我带你搭上 2021 年最后这趟便车吧。

<center>
    <img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/local-dev-101/db014925-93b5-4583-9f71-91f9c981842e.jpg" width="30%" alt="都卷起来吧" />
</center>

## 为什么需要这样一个环境？

其实很简单，无非就是以下这么几个原因

1. 找点有意思的工具，提升一下日常工作效率，这点是最重要的
2. 不论你是因为什么原因换了机器，还是在外做实施支持工作的时候，没了熟悉的环境，想必有为此烦恼过吧
3. 给小白用户提供一些帮助，他们平时不怎么操作 Kubernetes 集群，但是又想拥有一套可上手的 Kubernetes 工作环境？你大可直接分享给他们。
4. 有时候方便大家，其实就是方便自己

## 一些常用的工具

### #1. kubectl

kubectl 是 Kubernetes 命令行工具，是用来管理控制 Kubernetes 集群，这个必须要装，没什么好说的。

> 官网有很详细的用法，详情请移步官方文档
> [kubectl controls the Kubernetes cluster manager](https://kubernetes.io/docs/reference/kubectl/kubectl/ "kubectl controls the Kubernetes cluster manager")

### #2. auto-completion

命令自动补全功能是真心强烈建议安装的，它可以说是你在日常实操 kubectl 中最有用的工具也不为过，太实用了...你可以使用 Tab 键，在它的帮助下可以自动完成 kubectl 命令中的任意部分。

可以说我现在是完全依赖它过活，因为 Kubernetes 资源太多了，记不住啊...

> 启用 kubectl 自动补全功能 ，可以参考以下文档
>
> Some optional configuration for bash auto-completion on Linux.
> [bash auto-completion on Linux](https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-bash-linux/ "bash auto-completion on Linux")
>
> Some optional configuration for bash auto-completion on macOS.
> [bash auto-completion on macOS](https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-bash-mac/ "bash auto-completion on macOS")
>
> Some optional configuration for zsh auto-completion.
> [zsh auto-completion](https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-zsh/ "zsh auto-completion")

这里有必要提一句，我本地习惯用 zsh，zsh-autosuggestion 这个 plugin 我也强烈推荐，真的是谁用谁知道。

### #3. krew

krew 是 kubectl 插件的管理工具。

你会在里面找到非常有意思，且能提高工作效率的插件，大家可以自行去探索。

> 详情参见
> [Krew is the plugin manager for kubectl command-line tool](https://krew.sigs.k8s.io/ "Krew is the plugin manager for kubectl command-line tool")

### #4. kubectx

在日常工作中，我们经常需要访问不同的 Kubernetes 集群，通过原生的 kubectl 一般有如下几种做法

##### 1. 通过指定单独配置文件访问

指定访问一个集群，你可以这么做

```bash
k get pod --kubeconfig=dev-config

# 或者
KUBECONFIG=dev-config k get pod
```

##### 2. 通过指定 context 来访问

首先你需要知道你的工作环境中，有多少个可操作的 Kubernetes 集群

```bash
➜ k config get-contexts
CURRENT   NAME                              CLUSTER                         AUTHINFO                              NAMESPACE
          staging-context                   cluster.local-staging           kubernetes-admin-staging              traefik
*         kind-local-k8s                    kind-local-k8s                  kind-local-k8s                        kube-node-lease
          testing-context                   cluster.local-testing           kubernetes-admin-testing              rook-ceph
```

然后需要查看某一个集群的资源，通过指定具体的 context 即可

```bash
➜ k get pod --context staging-context

➜ k get pod --context testing-context
```

更换当前上下文

```bash
➜ k config use-context staging-context
Switched to context "staging-context".
```

更换上下文的 namespace

```bash
➜ k config set-context --current --namespace=kafka
Context "staging-context" modified.
```

大家可以看到以上的操作是多么的繁琐，好在有 `kubectx` 这个工具，它可以快速的帮助我们在不同集群和命名空间之间进行切换。

```bash
# 更换上下文
➜ kubectx staging-xdp
Switched to context "staging-xdp".

# 更换上下文的 namespace
➜ kubens kafka
Context "staging-xdp" modified.
Active namespace is "kafka".
```

> 详情参见
> [Faster way to switch between clusters and namespaces in kubectl](https://github.com/ahmetb/kubectx "Faster way to switch between clusters and namespaces in kubectl")

这里建议同时安装一下 [fzf](https://github.com/junegunn/fzf "fzf") ，通过在交互模式下直接使用游标选择上下文，就不用使用复制粘贴这种"笨"方法了，既获得了更好的体验又提升了效率。

> 另外如果需要了解 `KUBECONFIG` 的话，下面这篇文章值得一读
>
> [Mastering the KUBECONFIG file](https://ahmet.im/blog/mastering-kubeconfig/ "Mastering the KUBECONFIG file")

### #5. kube-ps1

有时候手里的 Kubernetes 集群太多了，经常会忘记自己当前切换到了哪个集群里，所以我每次都会通过
`kubectl cluster-info` 命令来确认一下，虽然很烦，但这其实是非常有必要的，因为如果你一旦 [误操作集群](https://mp.weixin.qq.com/s?__biz=MzU4MjY5NTc4OQ==&mid=2247484711&idx=1&sn=7fa7d252b68490321c935e63c1c0602f&chksm=fdb52b25cac2a2338e11fb9ff7897d99d666159512bd38df01cb94fe675da568b081c59de88d) 问题就大了，谁操作谁知道。

```bash
➜ k cluster-info
Kubernetes master is running at https://xxx.xx.x.xxx:6443
```

你也可以通过获取当前上下文的指令，来确认当前所在的集群是哪个

```bash
➜ k config current-context
staging-context
```

那么有没这样一个工具，不论我处于哪个集群的上下文，或者在哪个命名空间下，我都能快速的获取到信息呢？

有的，`kube-ps1` 这个工具它可以帮到我们，如下所示

```bash
➜ kubectx staging-context
Switched to context "staging-context".
(base)
~/.kube via 🅒 base at ☸️  staging-context (traefik)

➜ kubectx testing-context
Switched to context "testing-context".
(base)
~/.kube via 🅒 base at ☸️  testing-context (rook-ceph)
```

> 详情参见
> [Kubernetes prompt info for bash and zsh](https://github.com/jonmosco/kube-ps1 "Kubernetes prompt info for bash and zsh")

### #6. kubectl-images

确认集群中某一个服务的镜像版本，这个操作也是比较常见的，一般我们有如下几种做法

##### 1. 通过 get 结合 jq 获取

Pod 內只有一个 container 情况

```bash
➜ k get pod \
  -l run=hello-world \
  -o json|jq '.items[].spec.containers[]|{name: .name, image: .image}'
{
  "name": "hello-world",
  "image": "datawire/hello-world"
}
```

Pod 里有多个 containers，比如包含了另外一个 sidecar 容器

```bash
➜ k get pod \
  -l name=hello-world \
  -o json|jq '.items[].spec.containers[]|{name: .name, image: .image}'
{
  "name": "hello-world",
  "image": "lqshow/busybox-curl:1.28"
}
{
  "name": "linkerd-proxy",
  "image": "linkerd/proxy:stable-2.10.2"
}
```

##### 2. 通过 get 结合 JSONPath 获取

```bash
➜ k get pod \
  -l name=hello-world \
  -o custom-columns='PodName:metadata.name, ContainerName:spec.containers[*].name, IMAGES:spec.containers[*].image'
PodName        ContainerName               IMAGES
hello-world   hello-world,linkerd-proxy   lqshow/busybox-curl:1.28,linkerd/proxy:stable-2.10.2
```

##### 3. 通过 describe 获取

```bash
➜ k describe pod -l run=hello-world|grep -i image
    Image:          datawire/hello-world
```

以上3种方法完全可以拿到你想要的信息，如果有查看其他服务镜像版本的需求，我复制粘贴，调整下 Pod 名称或者 Pod Label 就行，是不是也可以用？

你说的没错，但是前提你需要清楚的知道查找对象的全名称，或者 Label 才行，那么有没一个只需要通过`关键字`就能定位到的工具呢？当然有(通过 `kubectl-images` 插件)，而且它还会帮你找出 init-container，是不是更加的清晰，更快？

```bash
➜ k images hello
 namespaces, 1 pods, 3 containers and 3 different images
+-------------+---------------------+------------------------------------------------+
|   PodName   |    ContainerName    |      ContainerImage                            |
+-------------+---------------------+------------------------------------------------+
| hello-world | hello-world         | lqshow/busybox-curl:1.28                       |
+             +---------------------+------------------------------------------------+
|             | linkerd-proxy       | linkerd/proxy:stable-2.10.2                    |
+             +---------------------+------------------------------------------------+
|             | (init) linkerd-init | linkerd/proxy-init:v1.3.11                     |
+-------------+---------------------+------------------------------------------------+
```

另外如果你不指定关键字，它会帮你列出 namesapce 下的所有镜像

```bash
➜ k images
 namespaces, 2 pods, 2 containers and 2 different images
+----------------------------------+-----------------+-------------------------------+
|             PodName              |  ContainerName  |        ContainerImage         |
+----------------------------------+-----------------+-------------------------------+
| hello-world                      | hello-world     | datawire/hello-world          |
+----------------------------------+-----------------+-------------------------------+
| traffic-manager-5cb99c9fd6-x6n8x | traffic-manager | docker.io/datawire/tel2:2.4.9 |
+----------------------------------+-----------------+-------------------------------+
```

> 详情参见
>
> [Show container images used in the cluster.](https://github.com/chenjiandongx/kubectl-images "kubectl-images")

### #7. stern

查找日志不论是作为一位集群管理员还是开发工程师，我相信它都是你平时 Debugging 问题时做的最多的操作吧。

通常情况下，使用 `kubectl logs` 命令来处理就行，但是它存在以下几个问题

1. Pod 名称的不断变化，Debugging 会陷入循环复制粘贴的困境，很尴尬
2. 即使你通过 Pod label 去过滤，记住每个服务对应的 label 也是一件很痛苦又没劲的事情
3. Pod 多副本的情况，虽然能够通过 label 匹配的到，但是可观测还是差了些

`Stern` 就能够很好的解决以上提的这些问题，大家不妨一试。

> 使用命令很简单，详情参见
> [Multi pod and container log tailing for Kubernetes](https://github.com/wercker/stern "Multi pod and container log tailing for Kubernetes")

### #8. kubectl-neat

`neat` 也是一个非常有用的工具，它可以对 Kubernetes 资源的 YAML/JSON 做清理。

有时我们需要将集群內某个资源的 YAML 导出修改后再运行，你会发现里面有一大堆你不想要的字段，比如 managedFields、creationTimestamp、uid 以及 status 等等，有了 `neat` 后，它会帮你自动删除这些无用字段。

以前我都是手动一个个删，一两次还好，但是身为 YAML 工程师，就没有一两次的说法，如果你整天做这种操作，想想都要奔溃了吧

下面是最基本的用法，其他用法移步官网

```bash
kubectl get cm nginx-config -oyaml | kubectl neat -o yaml
```

> 详情参见
> [Clean up Kubernetes yaml and json output to make it readable](https://github.com/itaysk/kubectl-neat "Clean up Kubernetes yaml and json output to make it readable")

### #9. ksniff

Kubernetes 世界 Pod 里网络问题抓包排查工具。

> 详情参见
> [Kubectl plugin to ease sniffing on kubernetes pods using tcpdump and wireshark](https://github.com/eldadru/ksniff "Kubectl plugin to ease sniffing on kubernetes pods using tcpdump and wireshark")

### #10. kubectl-iexec

开发工程师在 Debugging 期间，难免需要进入 Pod 內，通常我们有以下几种做法

##### 1. 单Pod单容器情况

```bash
➜ k exec -it hello-world-<HASH-ID> -- sh
/data #
```

##### 2. 单Pod多容器情况

```bash
➜ k exec -it hello-world-<HASH-ID> -c hello-world -- sh
/data #
```

##### 3. 通过 label 定位 Pod 名称

结合 `-o name` 获取名称

```bash
➜ kubectl exec \
  -it \
  $(kubectl get pod -l app=traffic-manager -o name |sed 's/pods\///') \
  -- sh
/ $
```

结合 `JSONPath` 获取名称

```bash
➜ kubectl exec \
  -it \
  $(kubectl get pod -l app=traffic-manager -o jsonpath='{.items[0].metadata.name}') \
  -- sh
/ $
```

结合 `grep/awk` 获取名称

```bash
➜ kubectl exec -it $(kubectl get pod |grep traffic|awk '{print $1}'|xargs -I{}) -- sh
/ $
```

其实不管单Pod，还是多Pod，我们都需要先定位到 Pod 的名字后才能通过 `kubectl exec` 命令进入到容器里面。

所以，不论是在流程和书写命令上都复杂了些，对小白用户存在一定的心智负担。

使用 `kubectl-iexec` 就能够很好的解决以上问题，只需要输入关键字就能完成定位，以下是使用过程中的一些截图

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/local-dev-101/f77e9791-fc35-40e1-a511-58de2bef7654.png"
    alt="1.选择进入的 pod"
    caption="1.选择进入的 pod"
    >}}


{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/local-dev-101/cacb127b-03ab-4a36-b0c0-18f0f36b5878.png"
    alt="2.选择指定的 container"
    caption="2.选择指定的 container"
    >}}

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/local-dev-101/7b195927-1352-4002-91d5-126631c36bf5.png"
    alt="3.通过交互式进入到指定 container"
    caption="3.通过交互式进入到指定 container"
    >}}

> 详情参见
> [Kubectl plugin to interactively exec into a pod](https://github.com/gabeduke/kubectl-iexec "Kubectl plugin to interactively exec into a pod")

### #11. telepresence

telepresence 我在前面文章也有分享过，这里就不做阐述了。

## 拿来主义，真香

大家可以很轻松的在自己的本地环境安装这些工具来使用，也可以利用 Dockerfile 将这些工具打包成一个 Kubetools 镜像来运行。

个人建议使用第二种容器的方式，`Build Once, Run Anywhere`。你懂得，后续即便你要不断的扩充工具库，也是非常的简单。

我将上面提到的这些工具，做成了一个 Docker 镜像，大家可以尝试运行使用。

```bash
docker run -it \
  --rm \
  --name kubetools \
  -v $KUBECONFIG:/root/.kube/config \
  ghcr.io/lqshow/dockerfiles/kubetools:0.0.1 \
  zsh
```

> Dockerfile 详见：<https://github.com/lqshow/dockerfiles/kubetools>

{{< github repo="lqshow/dockerfiles" >}}

## 写在最后

本篇分享其实带有强烈的个人偏好，仅罗列了我个人平时常用的工具而已，我觉得挺好的。

但是，我觉得好，并不表示你们觉得好。

所以没关系，大家完全可以按照 [Kubetools - A Curated List of Kubernetes Tools](https://collabnix.github.io/kubetools/ "Kubetools - A Curated List of Kubernetes Tools") 提供的工具库，打造一个完全属于你自己的 `Kubetools`。

<center>
    <img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/emoji/nice-5.gif" width="30%" alt="nice" />
</center>

希望这篇分享对大家都有所帮助。
