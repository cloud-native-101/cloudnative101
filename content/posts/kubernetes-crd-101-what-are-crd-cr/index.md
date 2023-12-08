---
keywords:
- cloud native
- cloud native 101
- kubernetes
- crd
title: "Kubernetes CRD 101:  什么是 CRD、CR，这些到底在说啥"
subtitle: "深入解析 Kubernetes 自定义资源"
description: 本文详细介绍了 Kubernetes 中的自定义资源（CR）和自定义资源定义（CRD），通过实际例子阐述了它们的创建、使用和重要性。文章探讨了 CRD 的关键字段、自定义资源的验证方法，以及如何通过 RESTful API 管理自定义资源。适合 Kubernetes 开发者和管理员阅读，帮助他们更好地理解和利用 Kubernetes 的扩展能力。
slug: kubernetes-crd-101-what-is-crd
date: 2021-11-08T08:18:00+08:00
weight: 99
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- Kubernetes
- Kubernetes CRD 101
---

![CRD](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/kubernetes-crd-101-what-are-crd-cr/cover.jpg)

`Kubernetes` 中的 `API` 对象都叫做资源（`Resource`），就是 `Yaml` 里 `Kind` 字段所描述的东西。

我相信大家对 `Kubernetes` 中的 `API` 对象都有所了解，每个 `API` 对象都有它自己的能力，分的非常细，概念也很多。当平台出现了问题，对一些新手来说，他们往往会不知所措，不知道从什么地方开始排查，非常的迷茫。

比如最近发生在我司的一件趣事。

我司内网 **开发环境** 的服务，对集群外提供访问是通过 `Ingress` 这个资源，**测试环境** 为了和 **生产环境** 保持一致，上了 `Service Mesh(Istio)`，用的是 `VirtualService` 对外提供服务。

`QA同学`日常在使用 `Kubernetes` 的过程中，经常接触到的都是 `Deployment`、`Service`、`Ingress` 这些原生内置的资源对象，那天 **测试环境** 因为某些原因出了问题，导致服务不能够正常访问，她以为是服务的 `Ingress` 资源没安装引起，闹出了不少笑话。

那么问题来了，`VirtualService` 到底是个什么资源呢？它又是怎么集成到 `Kubernetes` 集群內的呢?

这正是本篇文章要分享的主题。

## 一个例子

在回答上面这个问题之前，我们通过下面这个命令再来看一个 `Kubernetes` 集群內的资源。

> 乍一看，你是不是完全不知道 `dagnoderunner-sample` 有什么作用？

```bash
➜ kubectl get dnr -owide

NAME                   AGE
dagnoderunner-sample   86s
```

再来看下这个资源的 `Spec` 情况，我们发现 `Kind` 里描述的是一个叫 `DagNodeRunner` 的资源对象。

它的 `Spec` 也很简单，只有 `foo` 和 `baz` 两个字段，其他啥都没有。

`apiVersion` 字段的内容也不是我们平常所熟悉的，`apps/v1`、`apiextensions.k8s.io/v1` 和 `v1`，而是 `cloudnative101.net/v1beta1`。

```bash
➜ kubectl get DagNodeRunner dagnoderunner-sample -oyaml|kubectl neat

apiVersion: cloudnative101.net/v1beta1
kind: DagNodeRunner
metadata:
  name: dagnoderunner-sample
  namespace: default
spec:
  foo: bar
  baz: true
```

那么 `DagNodeRunner` 到底是个什么东西呢？它其实就是本篇文章分享的其中一个主题：`CR`。

## 什么是 CR

`CR` 的全称是 `Custom Resource`。

顾名思义，它是 `Kubernetes` 世界里的一种自定义资源，包括前面提到的 `VirtualService` 也是一种自定义资源，也就是说除了常见的 `Deployment` 之类的内置资源以外，`Kubernetes` 可以允许用户自定义资源。

那么前面举例的 `DagNodeRunner` 这个 `CR` 它有什么作用呢？说出来你可能不信，其实我也不知道有啥用。但是，这并不是重点，大家有没有好奇这个自定义资源是怎么生成的？

我们先以瓢画葫，来写个自定义资源的 `Spec` 吧，如下所示

```bash
cat <<EOF | kubectl create -f -
apiVersion: batch.cloudnative101.net/v1
kind: CronJob
metadata:
  name: cronjob-sample
  namespace: default
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 60
EOF
```

我们将以上脚本放在 `Kubernetes` 集群內执行，很遗憾，执行失败，并没有成功生成上面自定义的资源 `CronJob`。

```bash
error: unable to recognize "STDIN": no matches for kind "CronJob" in version "batch.cloudnative101.net/v1"
```

系统给出的提示已经非常明显了，上面的资源，集群并不能识别出来，所以导致创建自定义资源失败，很合理，不然啥都能创，岂不是要乱套了，可以想象的到 `Kubernetes` 的扩展能力会变的非常难管理。

那么 `dagnoderunner-sample` 这个自定义资源实例，到底是怎么建出来的呢，其实这个和 `gRPC` 对外提供服务前要先定义 `proto` 协议一个道理。

在 `Kubernetes` 世界里，用户的自定义资源想要在集群內运行，那么集群內，必须先要有这个自定义资源的定义，这里要引入本篇文章分享的另外一个主题，也就是说，要先有 `CRD`。

## 什么是 CRD

`CRD` 的全称是 `Custom Resource Definition`。

顾名思义，它指的就是自定义资源的定义，简单来说，它是描述我们定义的资源长什么样子。

`CustomResourceDefinition` 它是 `Kubernetes` 内置的原生的一个资源类型，它允许用户在 `Kubernetes` 中添加一个跟 `Deployment` 或者 `Ingress` 类似的、新的 `API` 资源类型，即：自定义资源。

我们可以通过 `kubectl get crd` 命令查看集群内定义的 CRD 资源。

```bash
➜ kubectl get crd

NAME                                 CREATED AT
dagnoderunners.cloudnative101.net   2021-11-06T04:09:55Z
```

我们通过命令再来看看这个 CRD 长什么样子

```bash
➜ kubectl get crd dagnoderunners.cloudnative101.net -oyaml|kubectl neat

apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: dagnoderunners.cloudnative101.net
spec:
  conversion:
    strategy: None
  group: cloudnative101.net
  names:
    categories:
    - cloudnative-labs
    kind: DagNodeRunner
    listKind: DagNodeRunnerList
    plural: dagnoderunners
    shortNames:
    - dnr
    singular: dagnoderunner
  scope: Namespaced
  versions:
  - name: v1beta1
    schema:
      openAPIV3Schema:
        description: DagNodeRunner is the Schema for the dagnoderunners API
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: DagNodeRunnerSpec defines the desired state of DagNodeRunner
            properties:
              baz:
                type: boolean
              foo:
                minimum: 1
                type: string
            required:
            - foo
            type: object
        type: object
    served: true
    storage: true
```

分析完以上 `Spec`，我们现在已经非常清楚上面例子提到的 `dagnoderunner-sample` 它为什么能够跑在集群內，而 `CronJob` 这个资源不能顺利生成的原因了。

只有我们在 `Kubernetes` 集群內执行了 `CRD` (可以理解为向集群注册了一种新的资源) 后，它才会允许用户将自定义的 `API` 资源类型添加到集群中，这样就可以像使用其他 `Kubernetes` 内置资源类型一样，使用 `kubectl` 来创建和访问它了。

### CRD 关键字段说明

| field                                   | desc                                                                     |
| --------------------------------------- | ------------------------------------------------------------------------ |
| metadata.name                           | CRD 的名字，名称与 spec 中的字段匹配：<spec.names.plural>.<spec.group>   |
| spec.scope                              | CRD 可以是命名空间范围的，也可以是集群范围的，值为 Namespaced 或 Cluster |
| spec.group                              | REST API 使用的组名称：`/apis/<group>/<version>`                         |
| spec.names.kind                         | CamelCased 格式的单数类型                                                |
| spec.names.plural                       | REST API 使用的复数名称：`/apis/<group>/<version>/<plural>`              |
| spec.names.singular                     | kubectl 中使用的资源单数名称                                             |
| spec.names.shortNames[*]                | kubectl 中使用的资源简称列表                                             |
| spec.names.categories[*]                | 自定义资源所属的分组资源的列表                                           |
| spec.versions[*].name                   | REST API 使用的版本号：`/apis/<group>/<version>`                         |
| spec.versions[*].served                 | 每个版本都可以通过 served 标志来独立启用或禁止                           |
| spec.versions[*].storage                | 其中一个且只有一个版本必需被标记为存储版本                               |
| spec.versions[*].schema.openAPIV3Schema | 用于验证自定义对象的 schema                                              |

**API Group**

它是相关 API 功能的集合，每个 Group 拥有一或多个 Versions，用于接口的演进。

**Kinds**

每个 `GV` 都包含多个 API 对象，称为 `Kinds`，在不同的 Versions 之间同一个 Kind 定义可能不同。

在 `Kubernetes` 世界里，同一种 API 对象可以有多个版本，这正是 `Kubernetes` 进行 API 版本化管理的重要手段。

### 自定义资源的验证

我们再来做另外一个测试，将下面的脚本放在 `Kubernetes` 集群內执行。

```bash
cat <<EOF | kubectl create -f -
apiVersion: cloudnative101.net/v1beta1
kind: DagNodeRunner
metadata:
  name: dagnoderunner-sample
spec:
  foo: bar
  baz: test
EOF
```

发现并不能执行成功，因为 `baz` 在 Spec 里定义的是 `boolean` 类型，这里我们用了 `string`，当然会失败。

```bash
error: error validating "STDIN": error validating data: ValidationError(DagNodeRunner.spec.baz): invalid type for io.cloudnative-labs.v1beta1.DagNodeRunner.spec.baz: got "string", expected "boolean"; if you choose to ignore these errors, turn validation off with --validate=false
```

`CRD` 它可以通过在 `spec.versions[*].schema.openAPIV3Schema` 定义规则，来验证用户提交的自定义实例是否符合资源的描述，只有符合标准了，`CR` 实例才可以在集群內运行。

### 自定义资源的 RESTful API

我们可以通过启动 `proxy server`，来看下新的 API 对象的定义

```bash
# starts a proxy to the Kubernetes API server
kubectl proxy --port=8080
```

#### # 1. 查看自定义资源的定义

```bash
➜ curl http://localhost:8080/apis/cloudnative101.net/v1beta1
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "cloudnative101.net/v1beta1",
  "resources": [
    {
      "name": "dagnoderunners",
      "singularName": "dagnoderunner",
      "namespaced": true,
      "kind": "DagNodeRunner",
      "verbs": [
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "create",
        "update",
        "watch"
      ],
      "shortNames": [
        "dnr"
      ],
      "storageVersionHash": "dlMUrzpeS6U="
    }
  ]
}
```

#### # 2. 查看自定义资源 `DagNodeRunner` 实例列表

```bash
➜ curl http://localhost:8080/apis/cloudnative101.net/v1beta1/dagnoderunners|jq

{
  "apiVersion": "cloudnative101.net/v1beta1",
  "items": [
    {
      "apiVersion": "cloudnative101.net/v1beta1",
      "kind": "DagNodeRunner",
      "metadata": {
        "creationTimestamp": "2021-11-06T13:30:50Z",
        "generation": 1,
        "managedFields": [
          {
            "apiVersion": "cloudnative101.net/v1beta1",
            "fieldsType": "FieldsV1",
            "fieldsV1": {
              "f:spec": {
                ".": {},
                "f:baz": {},
                "f:foo": {}
              }
            },
            "manager": "kubectl",
            "operation": "Update",
            "time": "2021-11-06T13:30:50Z"
          }
        ],
        "name": "dagnoderunner-sample",
        "namespace": "default",
        "resourceVersion": "259910",
        "uid": "eaafdd0f-d6e6-489c-a49f-e6ffaa6264c9"
      },
      "spec": {
        "baz": true,
        "foo": "bar"
      }
    }
  ],
  "kind": "DagNodeRunnerList",
  "metadata": {
    "continue": "",
    "resourceVersion": "266515"
  }
}
```

## 总结

### CRD

`CRD` 是 `Kubernetes` 的一种内置原生的一个资源类型，是 `Custom Resource Definition` 的缩写，它是自定义资源的定义，用来描述自定义资源长什么样子。

它是用来向 `Kubernetes` 集群注册一种新资源，用于拓展 `Kubernetes` 集群的能力。

有了 `CRD`，我们可以自定义底层基础设施的抽象，能够基于公司的产品自造概念(换句话来说可以根据业务需求来定制我们的资源类型)，利用 `Kubernetes` 已有的资源和能力，通过乐高积木的模式，定义出更高层次的抽象。

### CR

`CR` 就更好理解了，它实际上就是 `CRD` 的一个实例，是符合 `CRD` 中字段格式定义的一个资源描述。

### CRDs + Controllers

我们都知道 `Kubernetes` 的扩展能力很强大，但是光有 `CRD` 并没有啥用，它背后还需要有控制器(`Custom Controller`)的支撑，才能体现出 `CRD` 的价值，`Custom Controller` 会去监听 `CR` 的 `CRUD` 事件来实现自定义业务逻辑。

在 `Kubernetes` 的世界里，可以说是 `CRDs + Controllers = Everything`。

后面我会给大家分享一下，利用 `Kubebuilder` 如何从零开始开发 `CRDs`、`Controllers` 和 `Admission Webhooks` 的例子，我们下一篇分享见。

希望这篇文章对你有所帮助，比心。。。

## References

- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/ "Custom Resources")
- [Extend the Kubernetes API with CustomResourceDefinitions](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/ "Extend the Kubernetes API with CustomResourceDefinitions")
