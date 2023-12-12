---
keywords:
- cloud native
- cloud native 101
- kubernetes
- rbac
- oidc
- kubernetes security
title: "Kubernetes RBAC 101: 如何通过 OIDC 强化集群安全"
subtitle: "深入解析 Kubernetes 安全管理"
description: 本文详细介绍了如何通过整合 OpenID Connect (OIDC) 来增强 Kubernetes 集群的安全性。文章探讨了 Kubernetes 的认证与授权机制、RBAC 的核心概念，以及 OIDC 在 Kubernetes 中的配置和应用。适合 Kubernetes 管理员和开发人员阅读，帮助他们在云原生环境中实现更安全、高效的权限管理和访问控制。
date: 2023-11-13T08:18:00+08:00
weight: 99
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Kubernetes
- Kubernetes RBAC 101
---

![cover](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-rbac-oidc-security-guide/bf42248e-a862-467f-96fb-410db965d594.png)

🔐 **Kubernetes 集群管理中的安全难题**

在云原生的浪潮中，Kubernetes 已成为部署应用程序的不二法门。然而，随着它的普及，安全隐患也日益凸显。许多 Kubernetes 管理员和开发人员在处理复杂的权限和访问控制时，常常感到困惑和无助。究竟如何既简化管理又确保集群安全，这成了一个棘手的问题。

🌟 **引入 OpenID Connect (OIDC) —— 安全的革新**

本文将和大家一起探讨 Kubernetes 的角色基础访问控制（RBAC），并详述如何通过整合 OIDC 来强化集群安全。我们将逐步展开 OIDC 的配置和使用方法，以及如何通过它在 Kubernetes 中实现高效的身份验证和授权管理。

🚀 **准备好启程，探索 Kubernetes 安全的新领域吗？**

跟随我的步伐，了解如何将 OIDC 和 Kubernetes 的力量结合起来，使你的集群不仅高效易管，还坚不可摧！

## Kubernetes 安全的基石

### Kubernetes 中的认证与授权

**认证(Authentication，简称 AuthN)：你是谁？**

在 Kubernetes 的世界里，首先要确认的是“你是谁”。这是通过认证过程完成的，Kubernetes 验证访问集群的实体（人或进程）的身份。认证可以通过多种方式进行，包括证书、令牌、基本的用户名和密码，以及更复杂的系统如 OpenID Connect (OIDC)。

**授权(Authorization，简称 AuthZ)：你能做什么？**

一旦身份确定，下一步是明确你能在 Kubernetes 中执行哪些操作。这是通过授权过程完成的，其中 Kubernetes 根据一系列预定义的规则决定一个已认证的用户或进程可以执行哪些操作。

简而言之，AuthN 解决认证问题，AuthZ 解决授权问题。

### RBAC 的重要性

##### 1. **权限管理的复杂性**

随着 Kubernetes 在企业中的广泛应用，精确控制访问权限变得至关重要。这里，RBAC 发挥着关键作用，它允许管理员基于角色，而非单个用户身份分配权限，极大简化了权限管理。

##### 2. **RBAC 的核心组件**

1. **Role 和 ClusterRole：** 定义了一组权限。Role 通常用于命名空间级别的权限，而 ClusterRole 用于集群级别的权限。
2. **RoleBinding 和 ClusterRoleBinding：** 用于将角色（Role/ClusterRole）与用户、组或服务账户绑定。RoleBinding 在特定命名空间中有效，而 ClusterRoleBinding 在整个集群中有效。

### 为什么要重视 RBAC？

##### 1. **安全与合规**

在 Kubernetes 集群中实施严格的 RBAC 策略，对于保障安全和符合监管要求至关重要。限制关键资源的访问，可以最大程度地降低潜在安全风险。

##### 2. **最小权限原则**

RBAC 支持最小权限原则，确保用户和服务只获得完成任务所需的最小权限。

##### 3. **透明性和易管理性**

RBAC 的使用提升了权限管理的透明度和便捷性，使得权限的审计和调整更加容易。

RBAC 配置错误可能导致攻击者获得更高权限，并完全掌控整个集群。

## OpenID Connect (OIDC) 简介

### OpenID Connect (OIDC) 的基本概念

OpenID Connect（OIDC）是一种开放标准的认证协议，它建立在 OAuth 2.0 框架之上。OIDC 提供了一种安全且灵活的方式来认证和授权应用程序和系统中的用户。它在 OAuth 2.0 的基础上增加了与身份相关的特性，使其成为 Kubernetes 集群中认证用户的理想选择。

### 使用 OIDC 进行 Kubernetes 认证的好处

##### 1. **集中式身份管理**

OIDC 允许我们利用现有的身份提供者（`Identity Provider, IdP`）基础设施进行用户认证。这意味着可以集中管理用户身份，减少管理开销，并确保在多个应用程序和平台之间保持一致性。

##### 2. **身份验证与授权的分离**

OIDC 的引入在 Kubernetes 中实现了身份验证与授权的明确分离。**AuthN**（验证用户身份） 由 OIDC 提供，而 **AuthZ**（决定用户可以执行的操作）由 Kubernetes 的 RBAC 系统处理。这种分离增强了安全性，同时提供了更大的灵活性来管理用户访问权限。

##### 3. **单点登录（SSO）**

通过将 Kubernetes 与 OIDC 提供者集成，用户可以享受无缝的单点登录体验。一旦认证通过，用户可以访问多个 Kubernetes 集群和应用程序，而无需重新输入凭证，从而简化了用户体验。

##### 4. **增强的安全性**

OIDC 使用 JSON Web Tokens (JWT) 作为身份令牌，提供了一种安全有效的方式来传输和验证用户的身份信息。这确保了 OIDC 提供者与 Kubernetes 之间的通信安全，减少了未经授权的访问或数据泄露的风险。

##### 5. 细粒度的访问控制

结合 Kubernetes 的 RBAC，OIDC 可以实现细粒度的访问控制。管理员可以定义详细的策略，控制不同用户基于他们的**身份**和**组成员**资格访问 Kubernetes 资源的能力。

## 使用 OIDC 设置 Kubernetes 身份认证

在 Kubernetes 中集成 OIDC 需要几个关键步骤，涉及到配置 Kubernetes API 服务器、设置身份提供者，以及定义 OIDC 认证参数。以下是这些步骤的详细介绍：

### 1. 配置 Kubernetes API 服务器以使用 OIDC

Kubernetes API 服务器需要用特定的 OIDC 参数启动，以便支持 OIDC 认证。这通常涉及到编辑 API 服务器的启动配置文件。<br/>
在 API 服务器的配置文件中，需要添加一系列以 `--oidc-` 开头的参数。这些参数告诉 Kubernetes 如何与 OIDC 提供者通信和验证用户身份。

### 2. 设置身份提供者

首先我们需要选择一个身份提供者。

OIDC 是一个开放标准，Kubernetes 可以与多种兼容 OIDC 的身份提供者（如 Google Identity, Okta, Keycloak, **Casdoor** 或其他支持 OIDC 的身份提供者）集成，为用户提供灵活的选择，每个提供者都有其特定的配置步骤。

在所选的 OIDC 提供者中，创建一个新的客户端。客户端是我们的 Kubernetes 集群在身份提供者中的代表。

### 3. 定义 OIDC 认证参数

下面是一些关键参数

- `--oidc-issuer-url`：这是身份提供者的 URL，Kubernetes 用它来发现 OIDC 配置。
- `--oidc-client-id`：这是我们在身份提供者中为 Kubernetes 集群创建的客户端 ID。
- `--oidc-client-secret`（可选）：与客户端 ID 相关的密钥，用于保证身份验证请求的安全。
- `--oidc-username-claim`：指定 JWT 中哪个字段应该用作用户名。
- `--oidc-groups-claim`：指定 JWT 中的哪个字段代表用户的组成员资格，这对于后续的 RBAC 配置至关重要。
- `--oidc-username-prefix`：为从 OIDC 身份提供者获得的用户名添加前缀。这有助于避免用户名冲突。
- `--oidc-ca-file`：身份提供者的 TLS 证书的路径。这是用于验证身份提供者 SSL 证书的根证书。

### 4. 进行测试和验证

配置完成后，进行测试以确保 Kubernetes 可以通过 OIDC 正确认证用户，同时确认用户是否能够根据其 OIDC 凭证获得适当的访问权限。

## OIDC 集成实践

在理解 OIDC 设置流程后，是时候实际操作了。接下来的例子中，我将使用 [Kind](https://kind.sigs.k8s.io "Kind") 部署集群，并选择 [Casdoor](https://casdoor.org "Casdoor") 作为 OIDC 的身份提供者。

### 1. 在 IdP 注册应用程序

我们需要在 Casdoor 上先建立一个 `Application:` **kind-oidc**，获取 `Client ID` 和 `Client secret`。

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-rbac-oidc-security-guide/0c07b29e-e92c-441f-a3c0-032f50799f46.png"
    alt="注册应用程序"
    caption="注册应用程序"
    >}}

另外我还创建了一个用户: **oidc-user**，方便后续做验证。

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-rbac-oidc-security-guide/5b3c23cd-8788-4040-ae5c-2dd149714a55.png"
    alt="创建用户"
    caption="创建用户"
    >}}

### 2. 使用 Kind 在本地安装新集群

我在配置里直接注入了 OIDC 认证的相关参数，这里需要注意的是 IdP 的证书文件的注入，我是利用 `extraMounts` 这个配置选项，将我本地主机上的证书文件挂载到集群中的节点上。

💡 小贴士：如需了解本地自签证书的生成方法，请参考[先前的分享](https://mp.weixin.qq.com/s/f3vFf_GkURscjOwy2vXP1Q) 中的 **“OrbStack + Kind/创建 Kubernetes TLS Secret”** 一节。

```bash
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: oidc
nodes:
- role: control-plane
  extraMounts:
  - hostPath: $(pwd)/_wildcard.local-control-plane.orb.local.pem
    containerPath: /etc/kubernetes/pki/casdoor_ca.crt
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        # 身份提供者的 URL
        oidc-issuer-url: https://casdoor.local-control-plane.orb.local
        # 在 casdoor 中创建的 Kubernetes 客户端的客户端 ID
        oidc-client-id: 17aeccaf48f83a7da13c
        # 指定 JWT 中的 name 字段用作用户名
        oidc-username-claim: name
        # 添加到用户名申领之前的前缀，用来避免与现有用户名发生冲突
        oidc-username-prefix: "oidc:"
        # 指定 JWT 中的 groups 字段代表用户的组成员资格
        oidc-groups-claim: groups
        # casdoor 实例的证书
        oidc-ca-file: /etc/kubernetes/pki/casdoor_ca.crt
EOF
```

使用生成的配置，在本地安装 K8s 集群。

```bash
kind create cluster --config kind-config.yaml
```

### 3. 为用户生成 kubeconfig 文件

配置 kubectl 凭据，设置 OIDC 用户。

**步骤 1**： 获取当前集群的 kubeconfig 文件

```bash
# 将 kind 的 kubecfong 导出到本地
kind get kubeconfig --name oidc > $(pwd)/kubeconfig
```

**步骤 2**：配置 `kubectl` 的凭据，设置一个名为 `oidc-user` 的用户，并使用 OIDC 作为身份认证提供者。

> 使用以下命令，将获取到的 `client-id`、`client-secret`、`id-token` 和 `refresh-token` 配置到 `kubectl`。这样，`kubectl` 就可以使用这些令牌与 K8s 集群通信了。
>
> 如果我们在使用 `kubectl config set-credentials` 命令时指定了 `--kubeconfig` 参数，那么应该查看该参数指定的文件路径。如果没有指定，那么它默认更新 `~/.kube/config` 文件。

```bash
# Sets a user entry in kubeconfig
kubectl config set-credentials oidc-user --auth-provider=oidc \
  --auth-provider-arg=idp-issuer-url=https://casdoor.local-control-plane.orb.local \
  --auth-provider-arg=client-id=<client-id> \
  --auth-provider-arg=client-secret=<client-secret> \
  --auth-provider-arg=id-token=<id-token> \
  --auth-provider-arg=refresh-token=<refresh-token> \
  --kubeconfig=$(pwd)/kubeconfig
```

命令的各个部分的作用如下：

- `--auth-provider=oidc`: 指定身份认证提供者为 OIDC。
- `--auth-provider-arg=idp-issuer-url=<https://casdoor.local-control-plane.orb.local>`: 设置 OIDC 身份提供者的 URL。这是 OIDC 服务发现的基础 URL，客户端用它来发现 OIDC 的配置信息，如授权端点等。
- `--auth-provider-arg=client-id=<client-id>`: 指定注册到 OIDC 身份提供者的客户端 ID。在这里，你需要替换 `<client-id>` 为实际的客户端 ID。
- `--auth-provider-arg=client-secret=<client-secret>`: 提供与客户端 ID 匹配的客户端密钥。这个密钥应该是保密的，并且在 OIDC 身份提供者那里注册。替换 `<client-secret>` 为实际的客户端密钥。
- `--auth-provider-arg=refresh-token=<refresh-token>`: 如果提供，这个刷新令牌将被用来在当前的 ID 令牌过期时获取新的令牌。替换 `<refresh-token>` 为实际的刷新令牌。
- `--auth-provider-arg=id-token=<id-token>`: 当前的 OIDC ID 令牌。这通常是一个 JWT（JSON Web Token），包含用户的认证信息。替换 `<id-token>` 为实际的 ID 令牌。

我们可以通过 OIDC/OAuth2 向提供者获取访问令牌，以下是 Golang 实现的代码。
在文章的结尾我会介绍一个小工具，可以非常方便的获取到令牌。

```go
package main

import (
 "context"
 "fmt"
 "log"
 "time"

 "github.com/coreos/go-oidc/v3/oidc"
 "golang.org/x/oauth2"
)

const (
 providerURL  = "https://casdoor.local-control-plane.orb.local"
 clientID     = "17aeccaf48f83a7da13c"
 clientSecret = "320ce2d43be8d7a79111f1d3d98faf5a6f0f0b2f"
 userName     = "oidc-user"
 password     = "123456"
)

type Token struct {
 oauth2.Token
 RawIDToken string `json:"id_token,omitempty"`
}

func main() {
 ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
 defer cancel()

 provider, err := oidc.NewProvider(ctx, providerURL)
 if err != nil {
  log.Fatalf("Failed to get OIDC provider: %v", err)
 }

 conf := &oauth2.Config{
  ClientID:     clientID,
  ClientSecret: clientSecret,
  Scopes:       []string{"openid", "offline_access", "profile"},
  Endpoint:     provider.Endpoint(),
 }

 oauthToken, err := conf.PasswordCredentialsToken(ctx, userName, password)
 if err != nil {
  log.Fatalf("Error retrieving OAuth token: %v", err)
  return
 }

 rawIDToken, ok := oauthToken.Extra("id_token").(string)
 if !ok {
  log.Fatalf("No ID token found: %v", err)
  return
 }

 token := Token{Token: *oauthToken, RawIDToken: rawIDToken}

 fmt.Printf("Token: %+v\n", token)
}
```

{{< alert >}}
请注意，`id-token` 和 `refresh-token` 是敏感信息，我们应该确保 `kubeconfig` 文件的安全，不要无意中将其暴露在不安全的环境中。
{{< /alert >}}

**步骤 3**: 修改 `kubeconfig` 文件中的上下文，将此上下文关联的用户，指定使用 `步骤 2` 创建的用户。

```bash
# Set a context entry in kubeconfig.
kubectl config set-context kind-oidc \
 --user=oidc-user \
 --kubeconfig=$(pwd)/kubeconfig
```

这个命令的目的是在 `kubectl` 的配置中设置一个新的用户凭据，这样你就可以使用 OIDC 进行身份验证来与 Kubernetes 集群交互，可以实现更安全和集中的用户管理。

Nice，到此我们的 kubecofnig 文件已经准备就绪了，我们现在尝试用它来访问下试试？

```bash
➜ kubectl --kubeconfig kubeconfig get pod
Error from server (Forbidden): pods is forbidden: User "oidc:oidc-user" \
cannot list resource "pods" in API group "" in the namespace "default"
```

提示没权限，非常合理！因为我们还没做 AuthZ，所以当前用户无法对 K8s 执行任何操作。

### 4. 配置 Kubernetes 集群角色

我们为 OIDC 的用户，设置基于角色的访问控制（RBAC）的配置。先创建一个角色，定义一组基本的权限。

```bash
kubectl apply -f - <<EOF
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: role
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
EOF
```

接下来重点来了，将这个 `role` 角色的权限授予用户 `oidc:oidc-user`，特别需要注意的是 user 必须为 `oidc:oidc-user`。

```bash
kubectl create rolebinding oidc-user-binding --role=role --user=oidc:oidc-user
```

还记得吗？这个是因为我们在 API Server 那里对添加到用户名申领之前的前缀做了如下配置 `--oidc-username-prefix=oidc:`。

这里与[上次分享](https://mp.weixin.qq.com/s/jqdesmiF-qkhgdHdX7pc5w) 的从 JumpServer `捞出`集群里的 kubeconfig 方式是不一样的，那种是 ServiceAccount 的认证机制。

🎉 让我们再次验证下吧。

```bash
➜ kubectl --kubeconfig kubeconfig get pod
No resources found in default namespace.
```

### 思考

问题来了，如果注册的用户多了，如何有效管理大量用户的权限呢？

有小伙伴说，我可以给每一个用户都创建一个 Role，然后再使用 Rolebinding 进行绑定。这方法不是说不能用，但是还有没更好的方案呢？

我们利用 OIDC 集成的群组（Group）概念可以简化 Kubernetes 中的角色管理。在 OIDC 提供者中将用户分配到不同群组，群组成员资格信息将包含在 OIDC 令牌声明中，从而实现更细粒度的权限控制。

<section style="display: flex;"><section style="text-align: center;padding-right: 5px;width: 50%;"><p style="font-size: 16px;padding-top: 8px;padding-bottom: 8px;margin: 0 0 20px;padding: 0;line-height: 1.8em;color: #3a3a3a;"><strong style="font-weight: bold;color: white;">创建群组</strong></p><figure style="margin: 0;margin-top: 10px;margin-bottom: 10px;display: flex;flex-direction: column;justify-content: center;align-items: center;"><img data-ratio="1.1277777777777778" data-src="`https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-rbac-oidc-security-guide/999fb482-2959-460c-b233-c227b7a52f26.png" data-type="png" data-w="1080" style="margin: 0px auto 15px; max-width: 100%; border-radius: 5px; display: block; width: 100% !important; height: auto !important; visibility: visible !important;" data-index="4" src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-rbac-oidc-security-guide/999fb482-2959-460c-b233-c227b7a52f26.png" class="" _width="100%" crossorigin="anonymous" alt="Image" data-fail="0"></figure></section><section style="text-align: center;padding-left: 5px;width: 50%;"><p style="font-size: 16px;padding-top: 8px;padding-bottom: 8px;margin: 0 0 20px;padding: 0;line-height: 1.8em;color: #3a3a3a;"><strong style="font-weight: bold;color: white;">分配用户到群组</strong></p><figure style="margin: 0;margin-top: 10px;margin-bottom: 10px;display: flex;flex-direction: column;justify-content: center;align-items: center;"><img class="rich_pages wxw-img" data-ratio="0.9277777777777778" data-src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-rbac-oidc-security-guide/8be124f5-b8c5-4b17-abfa-e5713472b4e9.png" data-type="png" data-w="1080" style="margin: 0px auto 15px; max-width: 100%; border-radius: 5px; display: block; width: 100% !important; height: auto !important; visibility: visible !important;" data-index="5" src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-rbac-oidc-security-guide/8be124f5-b8c5-4b17-abfa-e5713472b4e9.png" _width="100%" crossorigin="anonymous" alt="Image" data-fail="0"></figure></section></section>

打个比方，以我们的组织为例，我们可以赋予 Team Leader 比普通成员更高的权限。

首先，让我们看看一个 OIDC 用户 JWT 的示例，其中包括用户所属的组信息：

```json
{
  "owner": "built-in",
  "name": "oidc-user",
  ...
  "groups": [
    "built-in/team-leader"
  ],
  "iss": "https://casdoor.local-control-plane.orb.local",
  "sub": "9c56d387-8ef1-4815-a2c1-441275c8c2cf",
  ...
}
```

在此 JWT 中，`groups` 字段表示用户所属的组。在我们的例子中，`oidc-user` 被分配到 `built-in/team-leader` 组。

接下来，我们定义一个专门针对 Team Leader 的 `ClusterRole`，以赋予他们对特定 Kubernetes 资源的操作权限。例如，我们可以允许他们获取、列出、观察、创建、更新、补丁和删除 pods 和 services：

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: oidc-team-leader
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF
```

然后，我们创建一个 `ClusterRoleBinding`，将上述定义的角色与：**built-in/team-leader** 用户组绑定：

```bash
kubectl create clusterrolebinding oidc-team-leader-binding \
  --clusterrole=team-leader \
  --group=built-in/team-leader
```

这一步骤确保了所有属于 `built-in/team-leader` 组的用户都将自动获得 `oidc-team-leader` 角色的权限。这种权限的分配方式既高效又安全，因为它依赖于集中式的身份验证机制，并利用 Kubernetes 的 RBAC 功能来精确控制访问权限。

补充说明：JWT 的 `groups` 字段支持数组格式，这意味着可以将单个用户添加到多个组中。结合 Kubernetes 的授权机制，这为灵活且精确的权限管理提供了强大的支持。

<center>
<img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/emoji/nice-1.gif" width="40%" alt="Nice" />
</center>

> 💡 在实际操作中，这些步骤应该都是自动化的流程。
> 比如使用身份提供商提供的 SDK 或命令行工具来帮助自动化获取令牌的过程，以及 kubectl 凭证的生成和 RBAC 的配置。

## kubelogin

[kubelogin](https://github.com/int128/kubelogin "kubelogin") 是一个用于 Kubernetes 的插件，它简化了与 OpenID Connect (OIDC) 提供者集成的认证过程。Kubernetes 通常使用 kubeconfig 文件进行认证，这种方法对于使用 OIDC 的场景来说可能比较复杂。kubelogin 旨在解决这个问题，提供更为便捷的认证方式。

这是它的一些主要功能：

1. **自动获取和刷新令牌**：它可以自动获取 ID 令牌和刷新令牌，无需用户手动介入。
2. **集成登录流程**：它与 OIDC 提供者的登录流程紧密集成，提供了一个更平滑的登录体验。
3. **kubeconfig 管理**：`kubelogin` 可以更新 kubeconfig 文件，使其包含必要的认证信息。

下面是一个使用谷歌身份平台进行 Kubernetes 身份验证的例子:

![kubelogin](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-rbac-oidc-security-guide/kubelogin.gif)

Cool，那么怎么配置呢？

非常简单，只需将原来配置 kubectl 凭证的命令换成以下命令即可，其他流程都保持不变。

```bash
# 1. Export kind of kubecfong locally
kind get kubeconfig --name oidc > $(pwd)/kubeconfig_oidc-login

# 2. Sets a user entry in kubeconfig
kubectl config set-credentials oidc-user \
  --exec-api-version=client.authentication.k8s.io/v1beta1 \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url=https://casdoor.local-control-plane.orb.local \
  --exec-arg=--oidc-client-id=17aeccaf48f83a7da13c \
  --exec-arg=--oidc-client-secret=320ce2d43be8d7a79111f1d3d98faf5a6f0f0b2f \
  --kubeconfig=$(pwd)/kubeconfig_oidc-login

# 3. Set a context entry in kubeconfig.
kubectl config set-context kind-oidc \
  --user=oidc-user \
  --kubeconfig=$(pwd)/kubeconfig_oidc-login
```

配置完毕，在第一次使用，会跳转到 IdP 登录界面，提示让我们登录，登录成功后就可以正常使用了。

```bash
kubectl --kubeconfig kubeconfig_oidc-login get pod
```

💡 小贴士：登录后的令牌信息会被缓存在 `~/.kube/cache/oidc-login` 目录下，文件内容包含两个字段 `id_token` 和 `refresh_token`，这里就不做展示了。

```bash
.kube/cache/oidc-login via 🅒 base at ☸️  kind-oidc
➜ ll
total 16
drwx------@  3 linqiong  staff    96B Nov 11 21:50 .
drwxr-x---  51 linqiong  staff   1.6K Nov 11 21:49 ..
-rw-------@  1 linqiong  staff   5.8K Nov 11 21:50 26bd83170f515b3931109eb03d1373...

```

## 写在后面

通过本文的介绍，我们不难发现，OIDC 在 Kubernetes 的安全管理中扮演着关键角色。它不仅提供了一种高效且安全的方式来处理身份验证问题，还与 Kubernetes 的 RBAC 系统完美结合，为集群安全提供了强有力的保障。

随着技术的不断发展和完善，我们有理由相信，Kubernetes 的安全管理将变得更加简单和高效。让我们一起期待这个方向的更多突破和创新！🚀

## References

- [Kubernetes Authentication with OIDC: Simplifying Identity Management](https://medium.com/@extio/kubernetes-authentication-with-oidc-simplifying-identity-management-c56ede8f2dec)
