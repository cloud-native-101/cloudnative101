---
keywords:
- cloud native
- cloud native 101
- kubernetes
- webhook
title: "Kubernetes Debugging 101: 如何在本地调试 K8s Webhook"
subtitle: "云原生小技巧: 如何在本地调试 K8s Webhook"
description: 本指南为 Kubernetes 开发者提供本地调试 Webhook 的高效策略，包括使用自签证书、调整 Makefile 和使用隧道工具，助力优化本地开发和测试流程。
date: 2023-11-27T08:18:00+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Kubernetes
- Webhook
- 云原生小技巧
- Kubernetes Debugging 101
---

![cover](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-debugging-101-local-webhook-testing/c9dedf33-c414-400e-9dc7-d9db4102606f.webp)

如果你是一名 Kubernetes Operator 的开发者，你曾经是否面临过这样一个棘手的问题：如何在本地环境中高效地调试 Webhook，尤其是在涉及有效证书回调的情况下。这篇文章旨在提供一种清晰的指南，帮助你克服这一挑战，优化本地开发和测试流程。

## 为什么本地调试 Webhook 如此重要？

当我们初步涉足 Kubernetes Webhook 时，面对的首个挑战通常是 Validation Webhook。
对于这种验证型 Webhook 来说，我们可以通过编写自动化测试来验证其功能。

这不仅确保了我的 Webhook 按预期工作，还允许我在日常开发中临时禁用它，从而加快了整个开发过程。这种方法让我能够巧妙地避免复杂的调试问题，而不对整体功能造成任何影响。

```golang
if os.Getenv("ENABLE_WEBHOOKS") != "false" {
    if err = (&webappv1.Guestbook{}).SetupWebhookWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create webhook", "webhook", "Guestbook")
        os.Exit(1)
    }
}
```

然而，对于 Mutating Webhook 来说，情况就变得有点复杂了。这类 Webhook 通常负责埋点的行为甚至更深层次的集群操作，比如注入 sidecar，这时候单靠自动化测试显然是不够的。我们需要一个更加高效的本地测试和调试方法。

在我们团队中，有同事采用 Kind 来部署和测试服务，这种方法非常值得称赞。它完全符合 K8s 的操作模式，为我们提供了一个接近生产环境的本地测试平台。但是，大家可能也注意到了，这种方式存在一个**效率瓶颈**：每次进行代码更改后，都需要重新构建 Docker 镜像并部署到集群中，这一过程既耗时又影响开发流程的连贯性。

作为开发工程师，我们渴望的是一个极速的内部开发循环，一个不再需要频繁的 docker build、docker push 或繁琐的部署流程，即使这些已经完全自动化。

我们希望能够使用本地熟悉的开发工具，如 VS Code 或者 IntelliJ IDEA 进行本地调试，而~~不是先部署到集群环境，再通过日志来分析错误这种远程调试模式~~。

## 从 Service 到 URL 的魔法变换

在不禁用 Webhook 的情况下，我们在本地启动 controller 后会有如下错误。<br/>这个比较好处理，我们只要使用自签证书，注入到 `WebhookServer` 即可，在前面的文章中我介绍过很多次，这里不再赘述。

```bash
2023-11-26T12:55:17+08:00       INFO    Stopping and waiting for webhooks
...
2023-11-26T12:55:17+08:00       INFO    Wait completed, proceeding to shutdown the manager
2023-11-26T12:55:17+08:00       ERROR   setup   problem running manager 
{"error": "open /var/folders/hn/v2s5bx...00000gn/T/k8s-webhook-server/serving-certs/tls.crt: 
no such file or directory"}
...
```

我们来运行一个示例，想必下面这个错误大家都非常熟悉吧，这个是因为 Webhook 注册的地址'不对'，它是集群内的地址。

```bash
➜ kubectl apply -f ./config/samples
Error from server (InternalError): error when creating "config/samples/webapp_v1_guestbook.yaml": 
Internal error occurred: failed calling webhook "vguestbook.kb.io": failed to call webhook: 
Post "https://testing-webhooks-webhook-service.testing-webhooks-system.svc:443/validate-webapp-foobar-ai-v1-guestbook?timeout=10s": 
no endpoints available for service "testing-webhooks-webhook-service"
```

我们再来看下 ValidatingWebhookConfiguration 的配置。

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: testing-webhooks-validating-webhook-configuration
webhooks:
- admissionReviewVersions:
  - v1
  clientConfig:
    service:
      name: testing-webhooks-webhook-service
      namespace: testing-webhooks-system
      path: /validate-webapp-foobar-ai-v1-guestbook
      port: 443
  failurePolicy: Fail
  matchPolicy: Equivalent
  name: vguestbook.kb.io
  rules:
  - ...
  ...
```

在这个配置中，webhooks 字段定义了一个或多个要注册的 Webhook。每个 Webhook 通过 clientConfig 配置与 Kubernetes API 服务器的连接方式。

正如大家所看到的 `https://testing-webhooks-webhook-service.testing-webhooks-system.svc:443/validate-webapp-foobar-ai-v1-guestbook`，这个默认地址其实就是 K8s 集群内部的地址。
这恰是 K8s 中处理 Webhook 的常规方法，其中 service 字段指向集群内运行的特定服务。

然而，在本地开发环境中，我们只在本地运行了我们的 Operator，直接使用内部服务是不大可能的，因为它要求 Webhook 服务必须部署在 K8s 集群中。

##### 使用 kubectl explain 探索 Webhook 配置

当我们在 K8s 中配置 Webhook 时，了解其配置细节是非常重要的。`kubectl explain` 是一个非常实用的小工具，它可以帮助我们深入理解 K8s 资源的各个属性。

以 `ValidatingWebhookConfiguration.webhooks.clientConfig` 为例：

```bash
➜ kubectl explain ValidatingWebhookConfiguration.webhooks.clientConfig
GROUP:      admissionregistration.k8s.io
KIND:       ValidatingWebhookConfiguration
VERSION:    v1

FIELD: clientConfig <WebhookClientConfig>:

DESCRIPTION:
FIELDS:
     caBundle   <string>
     `caBundle` is a PEM encoded CA bundle which will be used to validate the
     webhook's server certificate. If unspecified, system trust roots on the
     apiserver are used.

     service    <ServiceReference>
     `service` is a reference to the service for this webhook. Either `service`
     or `url` must be specified.

     If the webhook is running within the cluster, then you should use `service`.

     url        <string>
     `url` gives the location of the webhook, in standard URL form
     (`scheme://host:port/path`). Exactly one of `url` or `service` must be
     specified.
```

通过以上提供的详细信息，`clientConfig` 它除了通过定义 `Service` 让 API 服务器连接到 WebhookServer 外，还有另外一种方式，那就是直接通过 **URL 连接**。

为了解决这个问题，我们可以将 Webhook 的配置从服务转变为直接使用 URL。

##### 使用 URL 连接 Webhook

通过将 `clientConfig` 中的 `service` 字段替换为 `url` 字段，我们可以指定 Webhook 服务的外部 URL。这样一来，开发者可以在本地运行 Webhook 服务，并通过公开的 URL 使其可被 Kubernetes API 服务器访问。

例如：

```yaml
webhooks:
- admissionReviewVersions:
  - v1
  clientConfig:
    url: https://testing-webhooks.loca.lt/validate-webapp-foobar-ai-v1-guestbook
```

这种方法使得在本地开发环境中调试 Webhook 变得更加灵活和便捷。开发者可以使用本地服务器或通过隧道（如 [ngrok](https://ngrok.com "ngrok") 或 [localtunnel](https://github.com/localtunnel/localtunnel "localtunnel")）暴露的服务，从而实现在本地环境中的有效调试。

事实上，我最初的首选是 `ngrok`，因为这玩意确实好用，它还有个 `localhost:4040` 非常的实用，但遗憾的是，它的 tls 能力是付费的。幸好，有很多平替工具可以选择，比如 `localtunnel`，用起来也非常的方便。

**步骤 1**: 在我们的 `main.go` 需要接收一个证书路径

```golang
...
var certDir string
flag.StringVar(&certDir, "webhook-cert-dir", "/tmp/k8s-webhook-server/serving-certs", "Admission webhook cert/key dir.")
...

mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:                 scheme,
    Metrics:                metricsserver.Options{BindAddress: metricsAddr},
    HealthProbeBindAddress: probeAddr,
    WebhookServer: webhook.NewServer(webhook.Options{
        CertDir: certDir,
        Port:    9443,
    }),
    LeaderElection:   enableLeaderElection,
    LeaderElectionID: "dcc993a0.foobar.ai",
})

...
```

**步骤 2**：调整 Makefile，并启动我们的程序

修改 Makefile 文件，让其可以接收证书目录：

```makefile
.PHONY: run
run: manifests generate fmt vet ## Run a controller from your host.
 go run ./cmd/main.go --webhook-cert-dir ./config/certs
```

启动程序：

```bash
➜ make run
...
go run ./cmd/main.go --webhook-cert-dir ./config/certs
2023-11-26T11:18:42+08:00       INFO    controller-runtime.builder      Registering a mutating webhook  {"GVK": "webapp.foobar.ai/v1, Kind=Guestbook", "path": "/mutate-webapp-foobar-ai-v1-guestbook"}
2023-11-26T11:18:42+08:00       INFO    controller-runtime.webhook      Registering webhook     {"path": "/mutate-webapp-foobar-ai-v1-guestbook"}
2023-11-26T11:18:42+08:00       INFO    controller-runtime.builder      Registering a validating webhook        {"GVK": "webapp.foobar.ai/v1, Kind=Guestbook", "path": "/validate-webapp-foobar-ai-v1-guestbook"}
2023-11-26T11:18:42+08:00       INFO    controller-runtime.webhook      Registering webhook     {"path": "/validate-webapp-foobar-ai-v1-guestbook"}
...
2023-11-26T11:18:42+08:00       INFO    controller-runtime.webhook      Starting webhook server
...
2023-11-26T11:18:42+08:00       INFO    controller-runtime.webhook      Serving webhook server  {"host": "", "port": 9443}
...
```

**步骤 3**：将本地主机服务器通过隧道公开

```bash
# 安装
npm install -g localtunnel

# 使用 lt 命令启动隧道
lt --port 9443 \
    --local-https \
    --local-ca $(pwd)/certs/ca.crt \
    --local-cert $(pwd)/certs/tls.crt \
    --local-key $(pwd)/certs/tls.key \
    --subdomain testing-webhooks
your url is: https://testing-webhooks.loca.lt
```

**步骤 4**： 修改 ValidatingWebhookConfiguration 配置

我们将默认的 service 服务。

```yaml
webhooks:
- admissionReviewVersions:
  - v1
  clientConfig:
    service:
      name: testing-webhooks-webhook-service
      namespace: testing-webhooks-system
      path: /validate-webapp-foobar-ai-v1-guestbook
      port: 443
...
```

替换换成 url 直连模式

```yaml
webhooks:
- admissionReviewVersions:
  - v1
  clientConfig:
    url: https://testing-webhooks.loca.lt/validate-webapp-foobar-ai-v1-guestbook
...
```

> 以上仅以 **ValidatingWebhookConfiguration** 为例，如果你的 controller 同时使用了 **MutatingWebhookConfiguration**，别忘了，处理方式是一样的。

**步骤 5**：看看实际效果如何

最后，我们再执行同样一个用例，就可以被当前的 ValidatingWebhook 拦截到了。

```bash
➜ kubectl apply -f ./config/samples/webapp_v1_guestbook.yaml
The Guestbook "guestbook-sample" is invalid: metadata.name: Invalid value: "guestbook-sample": 
Guestbook name must be no more than 5 characters for test purposes
```

💡 **小贴士**：对于那些出于安全考虑不愿将本地服务暴露在公网上的小伙伴们，这里有一个安全的替代方案。你可以使用如 `docker.for.mac.host.internal` 这样的特定域名，它允许在不同环境下安全地连接到你的主机。

为了实现这一点，你需要根据你所在的环境，将 `docker.for.mac.host.internal` 替换为能够访问到你本地主机的相应域名。此外，别忘了补充 caBundle 字段，确保使用了正确的 CA 证书的 base64 编码字符串。

示例配置如下所示：

```yaml
webhooks:
- admissionReviewVersions:
  - v1
  clientConfig:
    caBundle: [你的 CA 证书的 base64 编码字符串]
    url: https://docker.for.mac.host.internal:9443/validate-webapp-foobar-ai-v1-guestbook
```

> 当然，在实际开发中，手动更改 Webhook 配置显然不是理想选择。推荐大家根据项目需求，结合这里提供的策略，开发自动化的配置处理流程。这不仅提升效率，还确保了配置的准确性和项目的灵活性。

## Debugging operator on Kubernetes

在以前的旧文中，我分享过 [《Kubernetes 101: Debugging Microservices on Kubernetes》](https://mp.weixin.qq.com/s/GuqCTz5qQ5PBKCI5grsgnQ) 这篇文章，主要介绍微服务在 K8s 环境下，在本地如何进行有效的调试。

{{< article link="/posts/kubernetes-101-debugging-microservices-on-k8s/" >}}

事实上，我们当前 webhook 的调试场景，完全可以利用 [Nocalhost](https://nocalhost.dev "Nocalhost") 这款工具达到同样的目的。

![nocalhost](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-101-debugging-microservices-on-k8s/640.gif)

在 Nocalhost 的开发模式下，我们可以直接在 K8s 集群中构建、测试和调试应用程序的，这意味着我们可以默认使用 `clientConfig.service 模式`，直接通过内部服务来连接，非常的方便。如果你还不太熟悉 Nocalhost，那可得抓紧时间补课了 😉

至于证书，我们自然是选择 cert-manager 来管理 admission webhooks 证书了，毕竟我们已经在集群里运行了，没有比它更方便的了。

> 我在 github 上写了个 chart，感兴趣的可以了解下。

{{< github repo="lqshow/testing-webhooks" >}}

<center>
<img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/emoji/nice.gif" width="40%" alt="Nice" />
</center>

## 写在最后

在本文中，我们深入了解了 Kubernetes Webhook 的本地调试流程，从基础理解到实际操作。通过将传统的基于服务的配置方式转变为直接使用 URL，我们不仅克服了本地调试的局限性，还引入了前所未有的灵活性和效率，大幅优化了整个开发周期。

此外，我们还探讨了如何在更广泛的 Kubernetes 生态中应用 Nocalhost 和 cert-manager。这些工具的整合不仅简化了开发和调试过程，还强化了安全性和可扩展性。通过实际案例，我们展示了如何在云原生环境中灵活地应对挑战，从而提升整体开发效率和质量。

好了，今天的分享就到这里，感谢你的阅读！🙌🏻😁📃 期待我们的下次见面！👋🚀
