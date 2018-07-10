---
layout: post
title:  到底什么是去服务器Serverless
date:   2018-07-09 12:13:28 -0700
excerpt: 借Martin Fowler老爷子博客里的文章，整理一下对Serverless的看法。
categories: Blog Serverless 去服务器化
---

这段时间自己开始上手尝试完全基于Serverless的服务搭建Web App，碰巧看到Martin Fowler老爷子的博客上po的[一篇文章](https://martinfowler.com/articles/serverless.html)（是老爷子请别人写的），自己也想把对Serverless的看法总结一下，放在blog上当做记录。

## 什么是去服务器化 serverless
去服务器化（Serverless）其实是两种不同技术趋势的整合
1. Serverless一词最初用于形容在云平台托管的后端服务，对应的模式被称为Backend As A Service (BaaS)。云服务商将后端服务包装抽象成为独立的service提供给用户。这一类Serverless产品中最具代表性的有AWS NonSQL数据库服务[DynamoDB](https://aws.amazon.com/cn/dynamodb/?nc1=h_ls)，消息队列服务SQS（阿里云对标产品为MQ/MNS）等。以DynamoDB为例，用户只需设置需要的容量（IOPS)和数据库的Schema，创建/配置/读写数据库全部通过AWS的public API，而AWS根据用户设置的读写容量和实际存储数据的大小收取费用。正常情况下单个DynamoDB table可支持160,000次/秒读取和80,000次/秒写入请求。
2. 近两年更火的Serverless，其实特指云服务厂商推出的Functions as a Service（FaaS）服务，如AWS Lambda，Azure Functions，阿里云的Compute Function等。所有这些产品本质上都是基于容器技术实现的，基于事件（event）触发的无状态（stateless）计算模型。用户只需上传一段代码，FaaS服务会在用户指定的场景下处理运行（事件触发，或直接通过API调用）。

值得注意的是，Serverless不能按照字面意思简单理解为没有服务器，例如Heroku和Google App Engine这样的PaaS平台虽然也是平台完全托管代码，但出发点和使用场景与BaaS和FaaS平台有本质的不同，这点会在后文进一步说明。

（未完待续）
