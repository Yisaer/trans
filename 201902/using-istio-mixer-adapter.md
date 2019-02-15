---
author: Yisaer
title: "通过自定义Istio Mixer Adapter在JWT场景下实现用户封禁"
description: "本文介绍了作者如何通过自定义Istio Mixer Adapter在JWT场景下实现用户封禁的原理与步骤"
categories: "原创"
tags: ["istio","服务网格","Service Mesh"]
date: 2019-2-15
---

# 通过自定义Istio Mixer Adapter在JWT场景下实现用户封禁

## 介绍

互联网服务离不开用户认证。JSON Web Token(后简称JWT)是一个轻巧，分布式的用户授权鉴权规范。和过去的session数据持久化的方案相比，JWT有着分布式鉴权的特点，避免了session用户认证时单点失败引起所有服务都无法正常使用的窘境，从而在微服务架构设计下越来越受欢迎。然而JWT单点授权，分布鉴权的特点也给我们带来了一个问题，即服务端无法主动回收或者BAN出相应的Token，使得即使某个服务主动封禁了一个用户时，这个用户同样可以使用之前的JWT来从其他服务获取资源。本文我们将阐述利用Istio Mixer Adapter的能力，来将所有请求在服务网格的入口边缘层进行JWT检查的例子,从而实现用户封禁与主动逐出JWT等功能。

## 背景

在我之中的[投稿](http://www.servicemesher.com/blog/practice-for-coohom-using-istio-in-production/)中，描绘了一个非常简单的基于K8S平台的业务场景，在这里我们将会基于这个场景来进行讨论。
对于一个简单的微服务场景，我们有着三个服务在Istio服务网格中管理。同时集群外的请求将会通过nginx-ingress转发给istio-ingressgateway以后，通过Istio VirtualService的HTTPRoute的能力转发给对应的服务，这里不再赘述。
![](http://ww1.sinaimg.cn/mw690/007pL7qRgy1g07dm5m93jj30wa0k2wg9.jpg)

从上图的架构模式中，我们可以看到所有的请求在进入网格时，都会通过istio-ingressgateway这个边缘节点，从而涌现出了一个非常显而易见的想法，即如果我们在所有的请求进入服务网格边缘时，进行特定的检查与策略，那么我们就能将某些不符合某种规则的请求拒绝的网格之外，比如那些携带被主动封禁JWT的HTTP请求。

## 了解Istio Mixer

为了达到我们上述的目的，我们首先需要了解一下[Istio Mixer](https://istio.io/docs/concepts/policies-and-telemetry/)这个网格控制层的组件。
Istio Mixer 提供了一个适配器模型，它允许我们通过为Mixer创建用于外部基础设施后端接口的处理器来开发适配器。Mixer还提供了一组模版,每个模板都为适配器提供了不同的元数据集。在我们的场景下，我们将使用[Auhtorization](https://istio.io/docs/reference/config/policy-and-telemetry/templates/authorization/)模板来获取我们每个请求中的元数据，然后通过Mixer check的模式来将在HTTP请求通过istio-ingressgateway进入服务网格前，通过Mixer Adapter来进行检查。这意味着我们将要编写一个自定义的Istio Mixer Adapter。这比起所有Istio Doc中的Task都来得有点复杂，但是不用担心，我们将在这里一步步来做。
