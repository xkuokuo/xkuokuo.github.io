---
layout: post
title:  聊一聊开源Serverless平台Apache OpenWhisk
date:   2018-07-20 01:01:28 -0700
excerpt: Apache OpenWhisk 是IBM开源的FaaS平台。整理一下OpenWhisk的系统架构，和自己对OpenWhisk的理解。
categories: Blog Serverless 无服务器 AWS
---

* TOC
{:toc}

这段时间开始有意识的去看一些OpenSource的FaaS框架的实现，原因一半是出于好玩儿，另一方面也是想通过对比AWS Lambda，加深自己对工作中碰到的问题的理解。上个礼拜试着上手OpenFaaS，搞了一块[树莓派](https://www.weibo.com/2663109067/GpS6o8haD)，在上面运行了一个OpenFaaS的cluster。这礼拜偶然看到一篇关于Apache OpenWhisk的[博文](https://medium.com/openwhisk/uncovering-the-magic-how-serverless-platforms-really-work-3cb127b05f71)，觉得讲的非常好，借着那篇文章，搜集了一些资料，整理整理自己的想法。

## Apache OpenWhisk简介
OpenWhisk是属于Apache基金会的开源FaaS计算平台[官网链接](https://openwhisk.apache.org/), 由IBM在2016年公布并贡献给开源社区（[github页面](https://github.com/apache/incubator-openwhisk)），IBM Cloud本身也提供完全托管的OpenWhisk FaaS服务IBM Cloud Function。从业务逻辑上看，OpenWhisk同AWS Lambda一样，为用户提供基于事件驱动的无状态的计算模型，并直接支持多种编程语言（理论上可以将任何语言的runtime打包上传，间接调用）。

![示意图]({{ "/assets/apache-openwhisk/illustration-openwhisk-architecture.svg" | absolute_url }})

- 高性能，高扩展性的分布式FaaS计算平台（注：说实话说了等于没说 XD）
- 函数的代码及运行时全部在Docker容器中运行，利用Docker engine实现FaaS函数运行的管理、负载均衡、扩展
- 同时OpenWhisk架构中的所有其他组件（如：API网关，控制器，触发器等）也全部运行在Docker容器中。这使得OpenWhisk全栈可以很容易的部署在任意IaaS/PaaS平台上。

## 系统概览 
（注：sytem high-level overview 这几个词到底咋翻译成中文。。。）

首先介绍一下OpenWhisk中从事件触发到函数执行完毕的流程。

![示意图]({{ "/assets/apache-openwhisk/openwhisk-system-overview.png" | absolute_url }})

OpenWhisk中，代码是基于**事件（Event）**触发的。事件产生于**事件源（feed）**，可以用于触发函数的事件多种多样：可以是IoT设备传感器发出的信号，可以是一个Github repo的push，也可以是最简单的一个前端HTTP请求。事件与对应的函数代码，通过**规则（Rule）**绑定。通过匹配事件对应的规则，OpenWhisk会触发对应的**行为（Action）**。值得注意的是，多个Action可以串联，完成复杂的操作。

## 内部实现 Under The Hood

下图为OpenWhisk的核心模块（图片来自这篇[博客](https://medium.com/openwhisk/uncovering-the-magic-how-serverless-platforms-really-work-3cb127b05f71)）
![示意图]({{ "/assets/apache-openwhisk/openwhisk-internal-implementation.png" | absolute_url }})

（未完待续）
