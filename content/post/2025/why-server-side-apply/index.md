---
title: "为什么你需要Server-Side Apply"
description: "通过高 Server-Side Apply 来提高资源管理的可靠性"
slug: "server-side-apply"
date: 2025-04-20T23:00:00+08:00
image: cover.jpg
tags:
    - Golang
    - Kubernetes
categories:
    - Technology
math: 
license: 
hidden: false
comments: true
draft: false
---

## 介绍

**服务器端应用**（Server-side Apply, SSA）是 Kubernetes 中用于管理资源配置的一种机制

它将配置应用的逻辑从客户端（如 kubectl）移到 API 服务器，从而更好地处理多个用户或工具同时管理同一资源时的冲突。

>**一句话总结：合理使用 SSA 可以提高并发更新时的成功率，同时通过字段所有权避免意外的字段覆盖**

## 解决什么问题？

* 不同的controler/client更新同一个资源时的冲突问题
* 如果更新不同的字段，后尝试更新的用户需要重试拉取最新的资源再尝试更新
* 使用SSA后，可以由服务器判断是否更新成功而不是返回冲突

## 版本支持情况

Kubernetes 1.13 引入，1.18 Beta 增加 managerFileds 支持，1.22 GA
实现方式
通过对象元数据中的 managedFields 字段跟踪哪个管理器（比如 kubectl 或某个控制器）拥有哪个字段。

```yaml
metadata:
  managedFields:
  - manager: kubectl
    operation: Apply
    time: "2025-03-23T23:00:00Z"
    fieldsV1:
      f:spec:
        f:replicas: {}
```

当发送期望的配置时，服务器会比较当前状态，检查是否有冲突，并根据字段所有权应用更改。如果检测到冲突，服务器可能会拒绝更改或需要手动解决。

* 每个 Kubernetes 对象在元数据中包含一个 managedFields 数组，每个条目记录了管理该字段的管理器（manager）信息，包括：
    * manager：管理器的名称（如 "kubectl" 或 "some-controller"）。
    * operation：操作类型（如 "Apply"）。
    * time：最后更新时间。
    * fieldsV1：一个字段列表，描述该管理器拥有的具体字段。

* 当用户或控制器通过 SSA 应用配置时，客户端发送完整的期望配置到 API 服务器，服务器会：
    1. 比较当前状态与期望状态，计算差异。
    2. 检查 managedFields，确定每个字段的当前所有者。
    3. 如果检测到冲突（即另一个管理器已更改某个字段），服务器会拒绝更改或根据策略处理（例如返回冲突错误）。

## 使用姿势

### 通过kubectl

* 确保 Kubernetes 集群版本为 1.22 或更高，SSA 默认启用。
* 使用 kubectl apply 应用配置时，注意可能出现的冲突，可以使用 --force-conflicts 标志强制覆盖，但要谨慎。

#### 获取

```shell
kubectl get configmap test \
  -o yaml \
  --show-managed-fields
```

#### 更新

```shell
kubectl apply -f config.yaml —server-side —field-manager=$(whoami)
```

#### 强制更新

```shell
kubectl apply -f deployment.yaml --server-side —field-manager=$(whoami) --force-conflicts
```

* 处理冲突： SSA 会检测字段冲突，如果发生冲突，kubectl apply 会返回错误。
    * 用户可以手动解决冲突，例如通过 kubectl edit 更新资源，或使用 --force-conflicts 忽略冲突。
    * 建议定期检查资源的状态，确保没有意外覆盖。
* 测试与验证： 在应用更改前，使用 --dry-run=server 进行服务器端 dry run，验证更改是否会引发冲突。例如：

```shell
kubectl apply -f deployment.yaml --dry-run=server -o yaml
```

### 通过controller

SSA 为控制器引入了一种新的 patch 类型，设置内容类型为 application/apply-patch+yaml 或 application/apply-patch+json 来应用 SSA。

确保控制器只管理它负责的字段，避免与其他人或工具的更改冲突。处理冲突时，可以实现重试机制或提供解决冲突的逻辑，比如强制覆盖特定字段。

* k8s.io/client-go/applyconfigurations
* 尽量使用 Client go applyconfigurations 提供的 Builder 模式来构建 patch， 而不是go struct来构建 apply patch，防止隐式的required字段的默认零值导致的意外修改
* k8s.io/code-generator
* applyconfiguration-gen
* 使用 clientset的Apply()方法（code-gen —apply-configuration-package）

## 总结

SSA 特别适合多团队或多工具协作管理的场景。遵循最佳实践以最大化利用 SSA 的优势，减少冲突并提高资源管理的可靠性。

| 项目 | 详情 |
| --- | --- |
| 解决的问题 | 多个客户端同时修改资源时的冲突，防止无意覆盖。|
| 版本支持 | 从 1.22 版本开始 GA，默认启用，需确保集群版本 >= 1.22。|
| 实现方式 | API 服务器通过 managedFields 跟踪字段所有权，检测并处理冲突。|
| kubectl 最佳实践 | 确保集群支持 SSA，使用 kubectl apply，处理冲突，测试 dry run。|
| 控制器最佳实践 | 使用字段 Builder 模式，管理字段所有权，处理冲突，测试多管理器场景。|

## 参考

* [服务器端应用（Server-Side Apply） | Kubernetes](https://kubernetes.io/zh-cn/docs/reference/using-api/server-side-apply/)
* [Adopting Server Side Apply in Knative - a Case Study - Dave Protasowski, VMware](https://www.youtube.com/watch?v=1rlTPjQ9dco)
* [SIG API Machinery Deep Dive - App... Abu Kashem & Stefan Schimanski, Joe Betz & Federico Bongiovanni](https://www.youtube.com/watch?v=oiC2w1PVjrQ)
* [applyconfigurations package - k8s.io/client-go/applyconfigurations - Go Packages](https://pkg.go.dev/k8s.io/client-go/applyconfigurations)
* https://www.zeng.dev/post/2023-k8s-api-codegen/

![notbyai-white](/images/notbyai/en/Written-By-Human-Not-By-AI-Badge-white.png)
![notbyai-black](/images/notbyai/jp/Written-By-Human-Not-By-AI-Badge-black.png)