---
layout: post
title:  聊一聊开源 Serverless 平台 Apache OpenWhisk
date:   2018-07-20 01:01:28 -0700
excerpt: Apache OpenWhisk 是 IBM 开源的 FaaS 平台。这篇介绍一下 OpenWhisk 的系统架构，也梳理一下自己对 OpenWhisk 的理解。
categories: Blog Serverless 无服务器 OpenWhisk
---

* TOC
{:toc}

这段时间开始有意识的去看一些开源的 FaaS 框架的实现，原因一半是出于好玩儿，另一方面也是想通过对比 AWS Lambda，加深自己对工作中碰到的问题的理解。上个礼拜试着上手 OpenFaaS，搞了一块[树莓派](https://www.weibo.com/2663109067/GpS6o8haD)，在上面运行了一个 OpenFaaS 的 cluster。这礼拜偶然看到一篇关于Apache OpenWhisk的[博文](https://medium.com/openwhisk/uncovering-the-magic-how-serverless-platforms-really-work-3cb127b05f71)，觉得讲的非常好，借着那篇文章，搜集了一些资料，整理整理自己的想法。

## Apache OpenWhisk 简介
OpenWhisk 是属于 Apache 基金会的开源 FaaS 计算平台[官网链接](https://openwhisk.apache.org/), 由 IBM 在2016年公布并贡献给开源社区（[github页面](https://github.com/apache/incubator-openwhisk)），IBM Cloud 本身也提供完全托管的 OpenWhisk FaaS服务 [IBM Cloud Function](https://console.bluemix.net/openwhisk/)。从业务逻辑上看，OpenWhisk 同 AWS Lambda 一样，为用户提供基于事件驱动的无状态的计算模型，并直接支持多种编程语言（理论上可以将任何语言的 runtime 打包上传，间接调用）。

![示意图]({{ "/assets/apache-openwhisk/illustration-openwhisk-architecture.svg" | absolute_url }})

OpenWhisk的特点：
- 高性能，高扩展性的分布式 FaaS 计算平台（注：说实话说了等于没说 XD）
- 函数的代码及运行时全部在Docker容器中运行，利用 Docker engine 实现 FaaS 函数运行的管理、负载均衡、扩展
- 同时 OpenWhisk 架构中的所有其他组件（如：API网关，控制器，触发器等）也全部运行在 Docker 容器中。这使得 OpenWhisk 全栈可以很容易的部署在任意 IaaS/PaaS平台上。
- 更重要的是，相比其他 FaaS 实现（比如 OpenFaaS），OpenWhisk 更像是一套完整的 Serverless 解决方案，除了容器的调用和函数的管理，OpenWhisk 还包括了用户身份验证/鉴权、函数异步触发等功能。

## 系统概览 
（注：sytem high-level overview 这几个词到底咋翻译成中文。。。）

首先介绍一下 OpenWhisk 中从事件触发到函数执行完毕的流程。

![示意图]({{ "/assets/apache-openwhisk/openwhisk-system-overview.png" | absolute_url }})

OpenWhisk 中，代码是基于**事件（Event）**触发的。事件产生于**事件源（feed）**，可以用于触发函数的事件多种多样：可以是 IoT 设备传感器发出的信号，可以是一个 Github repo 的 push，也可以是最简单的一个前端 HTTP 请求。事件与对应的函数代码，通过**规则（Rule）**绑定。通过匹配事件对应的规则，OpenWhisk 会触发对应的**行为（Action）**。值得注意的是，多个 Action 可以串联，完成复杂的操作。

## 内部实现 Under The Hood

下图为 OpenWhisk 的核心模块（图片来自这篇[博客](https://medium.com/openwhisk/uncovering-the-magic-how-serverless-platforms-really-work-3cb127b05f71)）。可以看到 OpenWhisk 本身完全构建与开源技术栈之上的（注：不由感叹一下，OpenWhisk 能最大程度的利用已有的开源组件，构建起一个完整的 FaaS 系统，真的很考验技术团队的**工程能力**）。图中的序号代表一个函数从触发到运行结束的顺序过程。
![示意图]({{ "/assets/apache-openwhisk/openwhisk-internal-implementation.png" | absolute_url }})

### 1 面向用户的 REST API （Nginx)
OpenWhisk 的核心系统通过 Rest API 接收函数触发和函数的CRUD请求。例如一个函数触发的 POST 请求格式如下
···
POST /api/v1/namespaces/$userNamespace/actions/myAction
···
此处的 nginx 服务器主要用于接收 HTTPS 请求（SSL termination)，并将处理后的 HTTP 请求直接转发给控制器（Controller）

### 2 真正进入系统：控制器（Controller）
控制器是真正开始处理请求的地方。控制器是用 Scala 语言实现的，并提供了对应的 REST API，接收 Nginx 转发的请求。Controller 分析请求内容，进行下一步处理。

### 3 CouchDB：身份验证和鉴权
继续用上一步用户发出的函数触发 POST 请求为例，控制器首先需要验证用户的身份和权限。用户的身份信息（credentials）保存在 CouchDB 的用户身份数据库（subjects database）中。验证无误后，控制器进行下一步处理。

### 4 还是 CouchDB：得到对应的Action的代码及配置
确认用户的身份后，控制器需要从 CouchDB 中读取将要被触发的函数（OpenWhisk 将要执行的代码片段抽象成为 **Action**，为了简便，此处直接称之为函数）。函数对应的数据存储在 CouchDB 的 whisk 数据库，主要包含要被执行的代码、默认参数、被执行代码的权限、及CPU/内存使用限制。

### 5 Consul 和负载均衡 
到了这一步，控制器已经有了触发函数所需要的全部信息，在将数据发送给触发器（Invoker）之前，控制器需要和 Consul 确认，从 Consul 获取处于空闲状态的触发器的地址。

Consul 是一个开源的服务注册/发现系统，在 OpenWhisk 中 Consul 负责记录跟踪所有触发器的状态信息。当控制器向 Consul 发出请求，Consul 从后台随机选取一个空闲的触发器信息，并返回。

值得注意的是：无论是同步还是异步触发模式，控制器都不会直接调用触发器API，所有触发请求都会通过 Kafka 传递（此处槽点非常之多），会在下一部分解释。

### 6 触发请求送进 Kafka

考虑设计系统时常出现的两种情况：
1. 系统瘫痪。已经送入 OpenWhisk 的触发请求，可能会丢失。
2. 系统负载过大（如：请求数突然增加，而后端运行函数代码的触发器集群无法及时扩容），没有足够的触发器执行函数代码，新进来的请求需要等待之前的请求完成后才能执行。
为了解决上述两个问题，OpenWhisk 的控制器并不会直接调用触发器 API，所有请求都会通过 Kafka 数据流异步发送给对应（从 Consul 中获取的）触发器。**值得注意的是，为了利用 Consul 实现负载均衡，每一个触发器都会有一个单独对应的 Kafka topic，前端控制器得到空闲触发器信息后，会将触发请求直接发送给对应的 Kafka topic，实现负载均衡**

Kafka 充当了 Controller 和 Invoker 之间的缓存，当后端 Invoker 负载过大，没有及时处理 Kafka 数据流中的请求时，Controller 依然可以将请求送入 Kafka，无需阻塞当前线程。同时所有送进 Kafka 的请求消息都会被以 log 的形式的形式保存在文件系统中，即使系统瘫痪，已经由 Controller 发出的请求也不会丢失。

考虑异步触发的情况，当控制器得到 Kafka 收到请求消息的的确认后，会直接向发出请求的用户返回一个 ActivationId，当用户收到确认的 ActivationId，即可认为请求已经成功存入到 Kafka 队列中。用户可以稍后通过 ActivationId 索取函数运行的结果。同步触发是通过后端的触发器通过 Kafka 反向发送给 Controller 函数运行结束的确认而实现的（TODO：具体的实现并不是很清楚，将来会给补上）

### 7 触发器运行用户代码
触发器从对应的 Kafka topic 中接收控制器传来的请求，并执行响应代码。OpenWhisk 的触发器是构建在 Docker 之上的，每一个函数触发都运行在一个独立的 Docker 容器之内。

设想最简单的一种情况，假设一个用户 Action 第一次被触发：
1. Invoker 接收到请求，创建一个新的 Docker 容器 （运行 docker run 命令），同时获取容器的 IP 地址 （docker inspect 命令）
2. 容器初始化。对一些语言来说这一步会涉及用户代码的编译。
3. 运行函数代码
4. 在 Couch DB 中保存运行结果（会在下一部分解释）

### 8 CouchDB 保存运行结果
触发器执行结果最终会被保存在 CouchDB 中的 whisk 数据库里。保存格式如下：
```
{
   "activationId": "31809ddca6f64cfc9de2937ebd44fbb9",
   "response": {
       "statusCode": 0,
       "result": {
           "hello": "world"
       }
   },
   "end": 1474459415621,
   "logs": [
       "2016-09-21T12:03:35.619234386Z stdout: Hello World"
   ],
   "start": 1474459415595,
}
```
保存结果包括用户函数的返回值，及日志记录。对异步触发用户，可以通过步骤6中返回的 activationID 取回函数运行结果。同步触发的的结果和异步触发一样保存在 CouchDB 里，控制器在得到触发结束的确认后，从 CouchDB 中取得运行结果，直接返回给用户。

## 小结
Apache OpenWhisk 本身构建在开源技术栈之上，最大程度的利用了已有的开源模块。写完这篇博客，也仅仅是了解一点皮毛，依旧有很多问（cao）题（dian）需要进一步深究：
1. OpenWhisk 通过 Consul 实现负载均衡，负载均衡的度量是什么（每个触发器的请求数量 / CPU负载 / …）?
2. 为了达到架构的一致性，对于同步触发和异步触发两种情况，OpenWhisk 都采取了通过 Kafka 队列缓冲触发请求。对同步触发的请求，额外增加的 latency 有多少？
3. 文章中提到，发送给 Kafka 的请求中包含触发函数的代码。对于代码中的依赖，需要和代码一起[打包上传](https://console.bluemix.net/docs/openwhisk/openwhisk_actions.html#openwhisk_actions)。然而对于如 Java 这样的语言，动则几十兆的 jar 包通过 Kafka 传递，会不会对性能产生很大影响？
4. 为了最大程度减少函数触发过程中，容器初始化带来的额外开销， OpenWhisk 会对容器使用采取一定的优化。[这篇文章](https://medium.com/openwhisk/squeezing-the-milliseconds-how-to-make-serverless-platforms-blazing-fast-aea0e9951bd0)讲解了 OpenWhisk 如何提升容器使用效率，解决冷启动问题。
5. 每一个触发器都对应一个 Kafka 的 topic，每增加一个触发器，就需要额外在 Kafka 中创建一个 topic，这会不会成为限制 OpenWhisk 扩展性的瓶颈？

...

Anyway 需要学习的东西还是太多了。
