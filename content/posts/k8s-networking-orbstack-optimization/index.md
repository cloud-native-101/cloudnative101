---
keywords:
- cloud native
- cloud native 101
- kubernetes
- networking
- orbstack
title: "Kubernetes Networking 101: OrbStack - 本地 K8s 环境的域名映射优化，开发者的新宠"
subtitle: "云原生小技巧：OrbStack — 本地 K8s 环境的域名映射优化，开发者的新宠"
description: 探索 OrbStack：一款针对本地 Kubernetes 环境的域名映射优化工具。本文深入分析了其独特功能，如为容器赋予个性化域名和支持 mDNS 域名解析，以及与 Kind 集群的协作，旨在提高 Kubernetes 开发者的效率和本地开发体验。
date: 2023-11-06T08:18:00+08:00
lastmod: 2023-12-09T11:07:04+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- 云原生小技巧
- Kubernetes
- Orbstack
- Kubernetes Networking 101
---

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/bd62c4df-aecf-4188-a003-94f691a01726.png"
    alt="Source: OrbStack.dev"
    caption="Source: OrbStack.dev"
    >}}


接上回的篇章，在 [《云原生小技巧 #3：在本地 K8s 中轻松部署自签 TLS 证书》](https://mp.weixin.qq.com/s/SHJeqf9SOUnytqzIsyHpeg) 中，我向大家展示了如何使用 [Dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq "Dnsmasq") 来搞定本地 DNS 的问题。那是一种相对传统的做法，简单易行。但是，对于我们这些追求效率的开发者来说，能够即插即用显然更加令人兴奋。

{{< article link="/posts/kubernetes-tls-101-deploy-self-signed-certificates/" >}}

今天，我要介绍的这个新伙伴: [OrbStack](https://orbstack.dev/ "OrbStack")，它的 Slogan 是: `Say goodbye to slow, clunky containers and VMs`。不过，说实话，我最喜欢的还是它的 **Local domain names** 的能力，因为它是零配置的。

## Container domain names

`OrbStack` 对待容器的态度可谓是亲(强)密(大)无间，它为每个容器赋予了一个独一无二的域名。

举个例子，假设我在本地启动了一个名为 `getting-started` 的容器，并将容器内的 **80** 端口映射到了本地的 **3000** 端口

```bash
docker run -d -p 3000:80 --name getting-started docker/getting-started
```

下面是我本地容器运行的情况

![本地运行情况](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/392e5edb-165c-4efd-9fc7-a5ee95352fcd.png)

在以往，我需要通过 **localhost + port** 的方式来访问这个容器。

![localhost 访问](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/aae5cbe2-c549-4f96-a309-42bb0b5f6373.png)

现在呢？只需通过 `OrbStack` 分配的域名，我就可以畅通无阻地访问它，而且不需要指定端口，非常的丝滑。

![域名访问](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/21feef92-d170-45c7-bba6-ba4582d9f545.png)

### mDNS

通过一系列的命令和检查，我们可以看到 `getting-started.orb.local` 这个域名确实被解析到了容器的 IP 地址：`192.168.215.3`。

```bash
➜ ping getting-started.orb.local
PING getting-started.orb.local (192.168.215.3): 56 data bytes
64 bytes from 192.168.215.3: icmp_seq=0 ttl=63 time=1.714 ms
64 bytes from 192.168.215.3: icmp_seq=1 ttl=63 time=0.472 ms
64 bytes from 192.168.215.3: icmp_seq=2 ttl=63 time=1.204 ms

➜ docker inspect getting-started \
  -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
192.168.215.3
```

我本机的 `/etc/hosts` 文件内容也没发生过变化，那么它是怎么做到的呢？我们先来看下系统的 DNS 配置信息。

```bash
➜ scutil --dns
DNS configuration

resolver #1
  ...

resolver #2
  domain   : local
  options  : mdns
  timeout  : 5
  flags    : Request A records
  reach    : 0x00000000 (Not Reachable)
  order    : 300000

resolver #3
  ...

```

在 `scutil --dns` 命令的输出中，`resolver #2` 部分的 `options` 字段包含 `mdns`，这表示该解析器配置用于处理 `.local` 域名的多播 DNS 查询。

> mDNS 即多播 DNS（[Multicast DNS](https://en.wikipedia.org/wiki/Multicast_DNS "Multicast DNS")）它是一种在本地网络上无需传统 DNS 服务器即可解析主机名的协议。这是 Bonjour（Apple 的实现）用来在本地网络上发现服务和主机名的一种机制。

```bash
# 获取本地 getting-started.orb.local 域名的地址
➜ dns-sd -G v4v6 getting-started.orb.local
DATE: ---Sat 04 Nov 2023---
 9:52:21.350  ...STARTING...
Timestamp     A/R  Flags         IF  Hostname                               Address                                      TTL
 9:52:21.351  Add  40000003      18  getting-started.orb.local.             FD07:B51A:CC66:0000:A617:DB5E:C0A8:D703%<0>  300
 9:52:21.352  Add  40000002      18  getting-started.orb.local.             192.168.215.3                                300

# 再查看特定主机的解析信息
➜ dns-sd -Q getting-started.orb.local
DATE: ---Sat 04 Nov 2023---
 9:55:31.664  ...STARTING...
Timestamp     A/R  Flags         IF  Name                          Type   Class  Rdata
 9:55:31.668  Add  40000002      18  getting-started.orb.local.    Addr   IN     192.168.215.3
```

Cool...有了这个能力就非常赞了，我可以轻松地将我的本地 Mysql 连接调整成这个样子。

![mysql](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/62335551-49ec-49cf-a1ba-7c38385cefbc.png)

### 自定义域名

`OrbStack` 允许用户自定义容器的域名，在启动容器时通过标签的方式方便的注入。

```bash
docker run --rm -l dev.orbstack.domains=foobar.local docker/getting-started
```

> 正如上面提到的 OrbStack 是通过 mDNS 来实现域名到 IP 的解析，所以它只对 `.local` 这个 TLD 有效，在做自定义域名的时候需要注意下。

![TLD](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/deaf9a38-ee38-48dc-b83c-6543619537e7.png)

### Domain names

通过访问 `http://orb.local` 我们可以看到所有正在运行的容器链接。

![DOMAIN NAMES](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/cc6d7d27-4dc5-4063-a20e-6a9ea6c64513.png)

甚至可以在它的客户端上查看容器列表，单击信息图标获取。

![ADDRESS](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/7369f29e-dd8a-4619-a64d-124ba339255e.png)

## OrbStack + Kind

接下来，我们利用 `Local domain names` 的能力，重新部署下自签 TLS 证书的流程，看下和[上次的分享](https://mp.weixin.qq.com/s/SHJeqf9SOUnytqzIsyHpeg) 有什么区别？

### 1. 获取集群的域名

通过 UI，获取到 Kind 集群的域名：`local-control-plane.orb.local`

![cluster domain](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/46761709-fe93-4fdf-9587-8ae9c78dba69.png)

### 2. 创建 K8s TLS Secret

然后，我们利用 [mkcert](https://github.com/FiloSottile/mkcert "mkcert") 创建了一个通配符证书

```bash
➜ mkcert '*.local-control-plane.orb.local'

Created a new certificate valid for the following names 📜
 - "*.local-control-plane.orb.local"

Reminder: X.509 wildcards only go one level deep, so this won't match a.b.local-control-plane.orb.local ℹ️

The certificate is at "./_wildcard.local-control-plane.orb.local.pem" and the key at "./_wildcard.local-control-plane.orb.local-key.pem" ✅

It will expire on 4 February 2026 🗓

```

并将其作为 K8s TLS Secret 添加到我们的集群中。

```bash
kubectl create secret tls tls-secret \
  --key=_wildcard.local-control-plane.orb.local-key.pem \
  --cert=_wildcard.local-control-plane.orb.local.pem
```

### 3. 配置 K8s Ingress 使用 TLS Secret

```bash
# 创建一个 Nginx Deployment
kubectl create deployment nginx-deployment --image=nginx:1.25.3
# 暴露 Deployment 作为一个 Service
kubectl expose deployment nginx-deployment --port=80
```

最后，我们在 K8s Ingress 资源中引用了这个 TLS Secret，以启用 HTTPS，对应的域名为： `nginx.local-control-plane.orb.local`。

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
spec:
  tls:                                          # 以下 4 行是为了支持 TLS
  - hosts:                                      #
    - nginx.local-control-plane.orb.local       #
    secretName: tls-secret                      #
  rules:
  - host: nginx.local-control-plane.orb.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-deployment
            port:
              number: 80
EOF
```

完成这些步骤后，我们就可以愉快地验证一下了，中间我们不需要对 DNS 做任何的配置。 🎉

![cert](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/14ca044c-d083-4294-be1c-0614794e549d.png)

## HTTPS for containers

{{< alert "tips" >}}
小贴士：`OrbStack` 在其即将到来的稳定版中将默认启用 HTTPS 支持，这意味着我们将不再需要手动创建、安装或信任自签名证书，为本地开发者带来前所未有的便捷。
{{< /alert >}}

![canary](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/3d80ea35-bbd5-49fd-a2bc-a9b017b0c2cf.png)

对于那些迫不及待想要体验最新功能的小伙伴们，可以通过以下步骤来抢先体验：进入设置，选择更新通道为 `Canary(faster)`，然后在 OrbStack 菜单中选择检查更新。

<section style="display: flex;"><section style="text-align: center;padding-right: 5px;width: 70%;"><p style="font-size: 16px;padding-top: 8px;padding-bottom: 8px;margin: 0 0 20px;padding: 0;line-height: 1.8em;color: #3a3a3a;"><strong style="font-weight: bold;color: #ffffff;">调整更新通道</strong></p><figure style="margin: 0;margin-top: 10px;margin-bottom: 10px;display: flex;flex-direction: column;justify-content: center;align-items: center;"><img class="rich_pages wxw-img" data-backh="214" data-backw="368" data-ratio="0.5814814814814815" data-src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/9f87abdc-b06d-4b62-8e7a-4e54a696be2c.png?wx_fmt=png" data-type="png" data-w="1080" style="margin: 0px auto 15px; max-width: 100%; border-radius: 5px; display: block; width: 100% !important; height: auto !important; visibility: visible !important;" data-index="12" src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/9f87abdc-b06d-4b62-8e7a-4e54a696be2c.png" _width="100%" crossorigin="anonymous" alt="Image" data-fail="0"></figure></section><section style="text-align: center;padding-left: 5px;width: 30;"><p style="font-size: 16px;padding-top: 8px;padding-bottom: 8px;margin: 0 0 20px;padding: 0;line-height: 1.8em;color: #3a3a3a;"><strong style="font-weight: bold;color: #ffffff;">检查更新<br><br></strong></p><figure style="margin: 0;margin-top: 10px;margin-bottom: 10px;display: flex;flex-direction: column;justify-content: center;align-items: center;"><img class="rich_pages wxw-img" data-ratio="0.5122699386503068" data-src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/f4dde638-669d-452b-ba73-5736de5c5397.png" data-type="png" data-w="652" style="margin: 0px auto 15px; max-width: 100%; border-radius: 5px; display: block; width: 166px !important; height: auto !important; visibility: visible !important;" data-index="13" src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/f4dde638-669d-452b-ba73-5736de5c5397.png" _width="166px" crossorigin="anonymous" alt="Image" data-fail="0"></figure></section></section>

升级完后，容器里已有的服务就可以直接通过 `https://getting-started.orb.local/` 访问了。

![img](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/dabad70d-60e7-4229-acb0-57fbb8528f2b.png)

## OrbStack 的原生 K8s 支持

事实上 OrbStack 提供了一个轻量级的单节点 K8s 集群，它对于开发环境来说是优化的。

![k8s](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/k8s-networking-orbstack-optimization/98c63b9f-830d-4646-8767-2b18c2f076f7.png)

在本地开发，如果没有 `multi-node clusters` 需求的话，我们可以不用 Kind 自建集群，直接用它就好。

除了以上提到的域名能力之外，我们还可以通过 **Pod 的 IP** 又或者 **Service 的 IP** 直接访问，这对于我们平时开发或者测试来说非常的方便，不需要再做 `port-forward` 了。

{{< alert "tips-2" >}}
在 Orbstack 提供的默认 K8s 集群下，安装 traefik 时我们使用 `LoadBalancer` 类型，它会直接提供泛域名:  `*.k8s.orb.local`，但是证书还是需要自己签。

```bash
helm upgrade -i traefik \
    --set ports.traefik.expose=true \
    --set-string service.type=LoadBalancer \
    --namespace traefik \
    --create-namespace \
    traefik/traefik
```

{{< /alert >}}

> 大家可以直接看 [Using Kubernetes](https://docs.orbstack.dev/kubernetes/ "Using Kubernetes")，这里不再赘述。

<center>
<img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/emoji/nice-4.gif" width="40%" alt="Nice" />
</center>

## 写在后面

从传统的 DNS 解决方案，到现代的 `OrbStack`，我们见证了本地开发环境的巨大变革。通过 `OrbStack`，我们不仅提升了工作效率，还享受到了前所未有的便捷。无论是容器的即时访问，还是 Kind 集群的无缝连接，`OrbStack` 都展现了其强大的能力。

在云原生的世界里，每一次技术的进步都是为了让开发者的生活变得更加简单。而今天，我们又向这个目标迈进了一大步。我希望你们能够尝试 `OrbStack`，并且享受它带来的便利。

在下一篇文章中，我将探索更多云原生技术的奥秘。敬请期待，我们下次见！🚀
