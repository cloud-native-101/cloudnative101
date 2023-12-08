---
keywords:
- cloud native
- cloud native 101
- kubernetes
- Dnsmasq
- mkcert
- CFSSL
- self-signed certificate
title: "Kubernetes TLS 101: 在本地 K8s 中轻松部署自签 TLS 证书"
subtitle: "云原生小技巧：在本地 K8s 中轻松部署自签 TLS 证书"
description: "本文详细介绍了在本地 Kubernetes 环境中配置和使用自签 TLS/SSL 证书的过程，以实现 HTTPS 的安全访问。文章逐步引导读者通过 Kind 创建本地 Kubernetes 集群，安装和配置 Traefik，以及使用 Dnsmasq 和 CFSSL/mkcert 工具生成和管理自签名证书。适合 Kubernetes 开发者阅读，以提升本地开发环境的安全性和便利性。"
date: 2023-11-03T08:18:22+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- 云原生小技巧
- Kubernetes
- Dnsmasq
- CFSSL
- Kubernetes TLS 101
---

{{< figure
    src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/deploy-self-signed-certificates/fdce5089-26f2-4d19-a771-f4b7692da3e6.png"
    alt="配图有 DALL·E 3 生成"
    caption="配图有 DALL·E 3 生成"
    >}}

随着互联网的飞速发展，安全性日益成为我们关注的焦点。HTTPS 已从一项奢侈的技术逐渐成为现代网络交互的标准。它不仅仅是保护信息的重要工具，更是实现信任和品质的象征。
当你在本地的 K8S 开发环境中遇到需要使用 HTTPS 来进行访问，又该如何为其配置 TLS/SSL 证书呢？🤔

今天，让我们一起揭秘如何在 K8S 环境中轻松自签证书，为你的本地开发环境带来安全性的提升！

## 0. Preparation

### 1. Install Kind

在生成 Kind 的配置文件时，我利用 Kind 的 `extraPortMapping` 配置选项将端口从主机转发到节点上运行的入口控制器。

它的作用是允许本地主机通过端口 **80/443** 向 Ingress 控制器发出请求。

```bash
cat << EOF > cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: local
nodes:
- role: control-plane
  image: kindest/node:v1.25.3
  extraPortMappings:
    - containerPort: 80
      hostPort: 80
      listenAddress: 127.0.0.1
    - containerPort: 443
      hostPort: 443
      listenAddress: 127.0.0.1
EOF
```

使用生成的配置，在本地安装 Kind 集群。

```bash
kind create cluster --config cluster.yaml
```

{{< alert >}}
因为配置了 extraPortMappings 的原因，如果需要在本地部署多套 K8s 集群，必须调整端口，又或者是去除 extraPortMappings 这个配置项。
{{< /alert >}}

### 2. Install Traefik

将 Traefik Labs 的图表仓库添加到 Helm。

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

这里我们给 websecure 设置主机端口为 443，这是为了确保传入的 HTTPS 流量可以被正确地路由到 Traefik。

```bash
helm upgrade -i traefik \
 --set ports.traefik.expose=true \
 --set ports.web.hostPort=80 \
 --set ports.websecure.hostPort=443 \
 --set-string service.type=ClusterIP \
 --namespace traefik \
 --create-namespace \
 traefik/traefik
```

### 3. Install Dnsmasq

虽然 `/etc/hosts` 对于简单的域名到 IP 地址的映射是很有用的，但它是静态的，而且不支持通配符或模式匹配，因此你不能为一个域名的所有子域设置相同的 IP 地址。

[Dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq "Dnsmasq") 它是一个轻量级的 DNS 正向和反向缓存服务器，同时也可以为 DHCP 功能提供服务。它被设计为易于配置和使用，并且它通常用于小型网络环境，特别是那些需要简单的 DHCP 和 DNS 服务的地方。

可以说 Dnsmasq 提供了一个更强大、灵活且集中的解决方案，以下是安装方法。

```bash
brew install dnsmasq
```

### 4. Install CFSSL/mkcert

为本地生成自签名的 SSL/TLS 证书有很多的工具，我在这里就分享两种，每种工具都有其特点和最佳用途，大家可以根据自己的需求和偏好来选择。

**第一种**：[CFSSL](https://github.com/cloudflare/cfssl "CFSSL")，以下是安装方法。

```bash
brew install cfssl
```

**第二种**：[mkcert](https://github.com/FiloSottile/mkcert "mkcert")，以下是安装方法。

```bash
brew install mkcert
```

### 小结

至此，我们已经完成了在本地 K8S 开发环境中准备的基础设施工作。通过 Kind，我们成功地搭建了一个本地的 Kubernetes 集群；通过 Helm 和 Traefik，我们为集群配置了强大的路由和反向代理功能；最后，通过 Dnsmasq，我们提供了一个灵活的本地 DNS 解决方案，替代了传统的 `/etc/hosts` 方法。这些都为我们接下来进行 TLS/SSL 证书配置打下了坚实的基础。

现在，我们将进入下一个阶段，真正探讨如何在 K8S 开发环境中配置自签证书，以实现 HTTPS 的安全访问。带上你的好奇心，和我一起探索这片云原生的奥秘之地！🚀

## 1. 创建自签名证书

首先，我们得创造一个自签证书。这里，我选择使用 `CFSSL` 来完成这一流程。

### 初始化配置

轻轻敲入以下命令，生成一个闪亮的配置文件 `config.json`✨

```bash
cfssl print-defaults config > config.json
```

配置内容，我们可以根据自己的需求稍作调整

```json
{
  "signing": {
    "default": {
      "expiry": "87600h",
      "usages": [
        "signing",
        "key encipherment",
        "server auth",
        "client auth"
      ]
    }
  }
}
```

### 生成证书

将以下内容写入到 `create-selfsign-cert.sh` 脚本

```bash
#!/usr/bin/env bash

cn=$1
if [[ -z "$cn" ]]; then
    read -p "Common name: " cn
fi

extfile=$(mktemp)
cat >"$extfile" <<INNER_EOF
{
  "CN": "$cn",
   "hosts": [
    "$cn"
  ],
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "names": [
    {
      "C": "CN",
      "L": "Shanghai",
      "O": "BaseBit",
      "OU": "XDP",
      "ST": "Shanghai"
    }
  ]
}
INNER_EOF


function finish {
  rm "$extfile"
}
trap finish EXIT

cfssl selfsign -config=config.json selfsign "$extfile" | cfssljson -bare "$cn"

mv ${cn}-key.pem ${cn}.key
mv ${cn}.pem ${cn}.crt
```

接着，执行以下命令生成泛域名证书：

```bash
$(pwd)/create-selfsign-cert.sh "*.kind.cluster"
```

💡 小贴士：执行后，以下三个文件将被创建：

```bash
➜ ls |grep kind
*.kind.cluster.crt         # 自签名证书
*.kind.cluster.csr         # 证书签名请求(CSR)文件
*.kind.cluster.key         # 私钥文件
```

> 关于 CFSSL 的更多魔法 🪄，请前往官网自行探索！

## 2. 创建 Kubernetes TLS Secret

接下来，我们将自签名证书和私钥存储在 Kubernetes 中作为 TLS Secret：

```bash
# 创建一个 TLS Secret
kubectl create secret tls tls-secret \
  --key *.kind.cluster.key \
  --cert *.kind.cluster.crt
```

## 3. 配置 Kubernetes Ingress 使用 TLS Secret

Nice，现在准备工作都完成啦 🎉，接下来，让我们召唤一个服务，试试效果吧！

```bash
# 创建一个 Nginx Deployment
kubectl create deployment nginx-deployment --image=nginx:1.25.3
# 暴露 Deployment 作为一个 Service
kubectl expose deployment nginx-deployment --port=80
```

我们可以引用这个 TLS Secret 在 Kubernetes Ingress 资源中启用 HTTPS，对应域名为 `nginx.kind.cluster`

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
spec:
  tls:                         # 以下 4 行是为了支持 TLS
  - hosts:                     #
    - nginx.kind.cluster       #
    secretName: tls-secret     #
  rules:
  - host: nginx.kind.cluster
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

## 4. 配置 Dnsmasq

因为我们的域名是自定义的，所以还需要在本地画上一道符咒。

将以下信息添加到 Dnsmasq 配置中，它可以从本地 IP 提供通配符[子]域。

```bash
echo 'address=/kind.cluster/127.0.0.1' >> $(brew --prefix)/etc/dnsmasq.conf
```

正如上面提到的，我将 Kind 集群内的端口映射到了主机上，所以这里只需配置 `127.0.0.1` 就好，不用再配置集群 host 的实际 IP。Dnsmasq 也会尝试解析子域名记录，例如 foo.kind.cluster 、 bar.kind.cluster ，这非常的方便。

配置完成，使用 brew 来重启 Dnsmasq

```bash
sudo brew services restart dnsmasq
```

我们再为 `.kind.cluster` 结尾的域名配置了一个专用的 DNS 解析器。

```bash
cat <<EOF | sudo tee /etc/resolver/kind.cluster
nameserver 127.0.0.1
EOF
```

最后，使用 dig 命令确认域名解析正确指向了 127.0.0.1：

```bash
➜ dig kind.cluster @127.0.0.1

; <<>> DiG 9.10.6 <<>> kind.cluster @127.0.0.1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 54402
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;kind.cluster.                  IN      A

;; ANSWER SECTION:
kind.cluster.           0       IN      A       127.0.0.1

;; Query time: 4 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Thu Nov 02 07:31:11 CST 2023
;; MSG SIZE  rcvd: 57
```

🎉 下面我们来做个验证吧。

![](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/deploy-self-signed-certificates/a4962a03-5d18-46da-a8dd-04ac8d518d85.png)

## 5. 信任自签名证书

使用自签名证书的缺点是，当用户访问通过此 Ingress 暴露的服务时，浏览器会显示一个警告，因为该证书不是由受信任的证书颁发机构颁发的，怎么解决呢？

其实很简单，在你的计算机上，将这个自签名证书添加到受信任的根证书存储中，这样你的浏览器就不会每次警告你连接不安全。

下面我们以 MacOS 系统举例

**步骤 1**: 双击 `.crt` 文件，这会打开“钥匙串访问”应用。在“钥匙串访问”中，你会看到证书已经被导入。如果没看到，你也可以手动拖拽.crt 文件到“钥匙串访问”窗口中。

**步骤 2**: 然后右键点击你导入的证书，选择“获取信息”。

![](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/deploy-self-signed-certificates/38919587-ff8f-4cb6-b9c4-3eade8932d7e.png)

**步骤 3**: 展开“信任”部分，在“使用此证书时”选择“始终信任”

![](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/deploy-self-signed-certificates/3eea6937-dcc6-46b0-9554-33c0df7164d3.png)

**步骤 4**: 最后，让我们再来做下验证 🎉

![](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/deploy-self-signed-certificates/8b03942c-c658-482a-b174-0180c8d6ffd4.png)

我们也可以通过 `CURL` 来验证，它也不再报任何的错误，效果如下所示

```bash
➜ curl -v https://nginx.kind.cluster
*   Trying 127.0.0.1:443...
* Connected to nginx.kind.cluster (127.0.0.1) port 443 (#0)
* ALPN: offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-AES128-GCM-SHA256
* ALPN: server accepted h2
* Server certificate:
*  subject: C=CN; ST=Shanghai; L=Shanghai; O=BaseBit; OU=XDP; CN=*.kind.cluster
*  start date: Nov  1 13:54:45 2023 GMT
*  expire date: Oct 29 13:59:45 2033 GMT
*  subjectAltName: host "nginx.kind.cluster" matched cert's "*.kind.cluster"
*  issuer: C=CN; ST=Shanghai; L=Shanghai; O=BaseBit; OU=XDP; CN=*.kind.cluster
*  SSL certificate verify ok.
* using HTTP/2
...
< HTTP/2 200
...
```

通过以上步骤，我们的本地开发环境将能够信任并正确使用自签名证书。

但还有一种更简单的替代方案 — `mkcert`，它可以帮助你在本地开发环境直接创建受信任的证书，不需要繁琐的配置，大大简化了本地环境配置：

```bash
mkcert -install

mkcert '*.kind.cluster'
```

大家不妨试下，这玩意完全可以替代 `CFSSL`，它对于本地开发和测试来说是足够的。

<center>
<img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/emoji/nice-2.gif" width="30%" />
</center>

## 写在后面

在这篇文章中，我们采用了相对传统的方法来创建自签证书，可能对某些场景来说并不算是真正的“云原生”。

在接下来的文章中，我会为大家带来 [Cert-Manager](https://cert-manager.io/ "Cert-Manager") 的详细介绍，它是一种真正云原生的自签证书方法，可以自动化证书的请求和续订，旨在更好地融合并适应现代的 Kubernetes 生态系统。🚀
