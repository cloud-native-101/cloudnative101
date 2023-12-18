---
keywords:
- cloud native
- cloud native 101
- kubernetes
- kubeconfig
- JumpServer
title: "Cloud Native Tips 101: 掌握 JumpServer 的登录方法"
description: "本文详细介绍了 JumpServer 的多种登录方式，包括 SSH Config File 和 Kubeconfig 文件生成，帮助读者高效管理 Kubernetes 集群，提升工作效率。"
date: 2023-10-28T08:18:00+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Cloud Native Tips 101
- Jumpserver
---

![cover](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/mastering-jumpserver-login/705dc5a8-4a56-4651-9394-f6f0569a8dec.png)

JumpServer 是一个开源的堡垒机系统，能够帮我们安全、集中地管理那些遍布各个角落的服务器。
虽然很多小伙伴都选择用 Web 浏览器来打开它，但话说回来，它真的只能这么用吗？

当然不！它还有好多酷炫的使用方法等着我们去探索，比如 SSH Config File、生成 Kubeconfig 又或者使用 VSCode Remote-SSH 插件。今天咱们就来探索一下这几种登录方式吧。

## SSH Config File

还记得我之前写的那篇博文: [一文告诉你什么是使用 SSH 的正确姿势！](https://mp.weixin.qq.com/s/-3TWHrmJrJflgW3BGPDhbg)吗？没看过的小伙伴，速度去补课！

😉 好了，话不多说，我们直接进入主题。

### 1. 生成 RSA 密钥对

首先，我们需要生成一个 RSA 密钥对，这可以使用 `ssh-keygen` 命令来完成：

```bash
➜ ssh-keygen -t rsa -C "lqshow@jumpserver.com" -f ~/.ssh/jumpserver-rsa -b 4096
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /Users/linqiong/.ssh/jumpserver-rsa
Your public key has been saved in /Users/linqiong/.ssh/jumpserver-rsa.pub
The key fingerprint is:
SHA256:JSnBLCesW8Xseit7jruRN23IbpG5iXFjg9idbB08jFk lqshow@jumpserver.com
The key's randomart image is:
+---[RSA 4096]----+
|   . =.          |
|    + B.E.       |
|   . *.*o .      |
|  . . +.=o       |
|   = = =So       |
|  o =o#o.        |
|    oB=Bo        |
|    o=*o         |
|    =Oo          |
+----[SHA256]-----+
```

生成后的密钥对将保存在指定的路径。我们可以查看它们：

```bash
➜ pwd
/Users/linqiong/.ssh
(base)
~/.ssh via 🅒 base at ☸️  kind-dvcd (minio)
➜ ls -lha |grep jump
-rw-------    1 linqiong  staff   3.3K Oct 27 19:36 jumpserver-rsa
-rw-r--r--    1 linqiong  staff   747B Oct 27 19:36 jumpserver-rsa.pub
```

### 2. 上传公钥

接下来，我们需要将生成的公钥上传到 JumpServer。复制、粘贴，你懂的！😜

```bash
➜ cat jumpserver-rsa.pub|pbcopy
```

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/mastering-jumpserver-login/a68fca14-dae8-4111-81a1-ef380b3d0a58.png"
    alt="上传公钥到 JumpServer"
    caption="上传公钥到 JumpServer"
    >}}

### 3. 配置 SSH Config

要使得从本地通过 SSH 直接连接到 JumpServer，我们需要在 `~/.ssh/config` 文件中进行配置：

```bash
Host jumpserver                        # 目标服务器别名
   HostName jumpserver.basebit.me      # 主机地址
   User qiong.lin                      # 用户名
   Port 2222                           # 指定端口
   IdentityFile ~/.ssh/jumpserver-rsa  # 私钥文件路径
   PreferredAuthentications publickey  # 权限认证方式
   PubkeyAcceptedAlgorithms +ssh-rsa   # 接受的公钥算法
   HostkeyAlgorithms +ssh-rsa          # 主机使用的密钥算法
```

### 4. 验证登录

看到这里，你是不是已经迫不及待想试试了？一切就绪，让我们登录吧！

```bash
➜ ssh jumpserver
```

下面是成功登录后的情况

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/mastering-jumpserver-login/9b40eea1-982c-47f2-8d21-5dd65fef1444.png"
    alt="验证登录"
    caption="验证登录"
    >}}

此外，如果你经常使用 K8s，你可以很方便地从 JumpServer 的命令行界面进入 Kubernetes 集群并进行操作！

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/mastering-jumpserver-login/b57e52bd-6bc9-47ad-b7d3-1602d2b031a3.png"
    alt="进入 Kubernetes 集群选择页"
    caption="进入 Kubernetes 集群选择页"
    >}}

看，这样我们就进入了一个集群！🎉

```bash
[K8S]> 6
开始连接Kubernetes https://172.18.18.81:6443  0.5
Welcome to JumpServer kubectl, try kubectl --help.
# kubectl get pod
NAME                                READY   STATUS               RESTARTS   AGE
demo-deployment-1-df6c9cb5c-5xshs   1/1     Running              0          4d2h
demo-deployment-1-df6c9cb5c-674t5   1/1     Running              0          4d2h
demo-deployment-7c555b66df-2htzw    1/1     Running              0          44d
xvm-4022425b-vm-v1-impl             1/1     Running              0          44d
#
```

然后你就可以使用 kubectl 命令来管理你的 Kubernetes 集群了。

<center>
<img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/emoji/nice-3.gif" width="40%" alt="Nice" />
</center>

通过 SSH 配置文件的方式，我们可以更加轻松地从本地终端访问和操作 JumpServer，而无需再通过 Web 浏览器登录。这无疑为我们的日常工作带来了更多的便利。🥳

## Kubeconfig File

另外还有一种方法，就是直接使用 KUBECONFIG 在本地登录。关于这个，我在旧文：[内部群炸了锅，同事把平台给删了](https://mp.weixin.qq.com/s/qq0Gs-P5HHU_sweRw37nHA) 也详细介绍了集群权限精细化管理的流程。这里我就不再赘述了。

那么问题来了，我们如何从 JumpServer 上`拿到` KUBECONFIG 文件呢？🤔

我们先来看下 Kubeconfig 的标准 Spec：

```yaml
apiVersion: v1
kind: Config
current-context:  <cluster-name>
contexts:
- context:
    cluster:  <cluster-name>
    user:  <cluster-name-user>
  name:  <cluster-name>
clusters:
- cluster:
    certificate-authority-data: <ca-data-here>
    server: https://your-k8s-cluster.com
  name: <cluster-name>
users:
- name:  <cluster-name-user>
  user:
    token: <secret-token-here>
```

💡小贴士： 从上面的结构中，你会发现有 5 个关键点我们需要特别关注。

1. certificate-authority-data: 集群的 CA 证书
2. server: 集群端点
3. name: 集群名称
4. user: 服务帐户的名称
5. token: 服务帐户的秘密令牌。

也就是说，只要从集群里拿到了这几点信息，就可以构建一个 Kubeconfig 了，我们来直接看下流程吧。

### 1. 创建 ServiceAccount

> ServiceAccount 是 Kubernetes API 管理的默认用户类型。

```bash
kubectl -n default create serviceaccount jumpserver-cluster-admin
```

### 2. 创建 Secret

> 以前创建一个 sa，会默认创建一个以 sa 为前缀的 secret，但是从 Kubernetes 版本 1.24 开始，必须使用注释 kubernetes.io/service-account.name 和类型 kubernetes.io/service-account-token 单独创建服务帐户的密钥。
>
> 所以，当前步骤咱们根据集群版本，按需创建 Secret 即可

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: jumpserver-cluster-admin-secret
  namespace: default
  annotations:
    kubernetes.io/service-account.name: jumpserver-cluster-admin
type: kubernetes.io/service-account-token
EOF
```

### 3. 创建 ClusterRoleBinding

> 假设我的 Jumperser 账户有管理员权限，这里我偷懒就不创建具体的 Role 了，我直接建立一个 绑定到 `cluster-admin` 的 ClusterRoleBinding

```bash
cat << EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jumpserver-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jumpserver-cluster-admin
  namespace: default
EOF
```

### 4. 创建 Kubeconfig

待以上关键资源创建完后，我们再通过执行以下脚本，来获取集群里的信息

```bash
export CLUSTER_NAME=$(kubectl config current-context)

export SECRET=$(kubectl -n default get secrets | \
               grep ^jumpserver-cluster-admin | \
               cut -f1 -d ' ')

export CLUSTER_ENDPOINT=$(kubectl config view --minify -o \
                          jsonpath='{.clusters[0].cluster.server}')

export CLUSTER_CA_CERT=$(kubectl -n default get secret ${SECRET} -o yaml | \
                         awk '/ca.crt:/{print $2}')

export SA_SECRET_TOKEN=$(kubectl -n default get secret ${SECRET} -o \
                        go-template='{{.data.token}}' | \
                        base64 -d)
```

最后，执行以下命令，所有变量都会被替换，生成一个可用的 Kubeconfig 文件

```bash
cat << EOF > jumpserver-cluster-admin-config
apiVersion: v1
kind: Config
current-context: ${CLUSTER_NAME}-context
contexts:
- name: ${CLUSTER_NAME}-context
  context:
    cluster: ${CLUSTER_NAME}
    user: jumpserver-cluster-admin
clusters:
- name: ${CLUSTER_NAME}
  cluster:
    certificate-authority-data: ${CLUSTER_CA_CERT}
    server: ${CLUSTER_ENDPOINT}
users:
- name: jumpserver-cluster-admin
  user:
    token: ${SA_SECRET_TOKEN}
EOF
```

### 5. 验证

再次使用  kubectl 命令来验证下：

```bash
➜ kubectl --kubeconfig $(pwd)/jumpserver-cluster-admin-config get node
NAME          STATUS   ROLES                  AGE    VERSION
kea-app       Ready    <none>                 185d   v1.23.5
kea-master1   Ready    control-plane,master   307d   v1.23.5
kea-monitor   Ready    <none>                 307d   v1.23.5
kea-util      Ready    <none>                 307d   v1.23.5
kea-xdp1      Ready    <none>                 307d   v1.23.5
kea-xdp2      Ready    <none>                 307d   v1.23.5
kea-xdp3      Ready    <none>                 307d   v1.23.5
```

🥳 恭喜！你已经成功地从 JumpServer `捞出了` Kubernetes 集群的 kubeconfig 文件，并且验证了其有效性。现在可以在本地结合 [kubectx](https://github.com/ahmetb/kubectx "kubectx") 快速的切换集群，愉快的玩耍了。

<center>
<img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/emoji/nice.gif" width="40%" alt="Nice" />
</center>

## 总结

总结一下，JumpServer 提供了丰富的连接方式，以满足不同工程师的个性化需求。

而我们今天提到的 SSH Config 和 Kubeconfig 两种方式，无疑为我们提供了更为高效、简洁的登录方式，使得在工作中无需频繁切换工具和界面。当然，你可以根据自己的实际情况选择最合适的登录方式。

好了，今天的分享就到这里，感谢你的阅读！如果觉得有用，别忘了点赞、转发、评论，我们下期再见！👋🚀
