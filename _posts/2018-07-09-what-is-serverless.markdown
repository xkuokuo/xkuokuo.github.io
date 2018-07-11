---
layout: post
title:  什么是无服务器Serverless
date:   2018-07-09 12:13:28 -0700
excerpt: 借Martin Fowler老爷子博客里的文章，整理一下对Serverless的看法。
categories: Blog Serverless 无服务器 AWS
---

这段时间自己开始上手尝试完全基于Serverless的服务搭建Web App，碰巧看到Martin Fowler老爷子的博客上po的[一篇文章](https://martinfowler.com/articles/serverless.html)（是老爷子请别人写的），自己也想把对Serverless的看法总结一下，放在blog上当做记录。

## Serverless无服务器名词的出现

* 目前网路上能找到的最早对Serverless这一概念的描述来自于一篇[2012年的博文](https://readwrite.com/2012/10/15/why-the-future-of-software-and-apps-is-serverless/)。 然而这篇文章中对Serverless概念的描述仅停留在服务器托管和服务器虚拟化这一层面。
* 2014年AWS发布AWS Lambda
* 2015年AWS发布API Gateway。这两个产品的出现和结合，使得完全构建在Serverless架构上的应用变为可能
* 2015年的AWS Re:Invent专门介绍[PlayOnSports](http://www.playonsports.com/)这家公司的客户案例，证明Serverelss架构完全可以应用于生产环境当中。
* 2016年第一届Serverless Conf在纽约举办
* 同年AWS推出Serverless的编程架构模型Serverless Architecture Model（SAM）

。。。。。。

## Serverless代表的两种技术趋势
无服务器（Serverless）其实是两种不同技术趋势的整合
1. Serverless一词最初用于形容在云平台托管的后端服务，对应的模式被称为Backend As A Service (BaaS)。云服务商将后端服务包装抽象成为独立的service提供给用户。这一类Serverless产品中最具代表性的有AWS NonSQL数据库服务[DynamoDB](https://aws.amazon.com/cn/dynamodb/?nc1=h_ls)，消息队列服务SQS（阿里云对标产品为MQ/MNS）等。以DynamoDB为例，用户只需设置需要的容量（IOPS)和数据库的Schema，创建/配置/读写数据库全部通过AWS的public API，而AWS根据用户设置的读写容量和实际存储数据的大小收取费用。正常情况下单个DynamoDB table可支持160,000次/秒读取和80,000次/秒写入请求。
2. 近两年更火的Serverless，其实特指云服务厂商推出的Functions as a Service（FaaS）服务，如AWS Lambda，Azure Functions，阿里云的Compute Function等。所有这些产品本质上都是基于容器技术实现的，基于事件（event）触发的无状态（stateless）计算模型。用户只需上传一段代码，FaaS服务会在用户指定的场景下处理运行（事件触发，或直接通过API调用）。

值得注意的是，Serverless不能按照字面意思简单理解为没有服务器，例如Heroku和Google App Engine这样的PaaS平台虽然也是平台完全托管代码，但出发点和使用场景与BaaS和FaaS平台有本质的不同，这点会在后文进一步说明。

## 去服务器化的应用实例
文中列举了两个Serverless的应用场景。
### 案例一：UI驱动的Web应用
一个简单的web应用，浏览器中的客户端和同一个服务器（或同一API后的服务器集群）交互，同一个后端应用负责数据库的CRUD和所有后端业务逻辑（身份验证，购买下单，历史记录查询等）。
![示意图]({{ "/assets/what-is-serverless/web-ui-example.svg" | absolute_url }})
当引入Serverless之后，系统结构变为下图：
![示意图]({{ "/assets/what-is-serverless/serverless-web-ui-example.svg" | absolute_url }})

可以看出：
1. 前端已不仅局限于和单一服务器交互。一些后端功能被抽象成为独立的BaaS Service，前端的JS完全可以直接其他BaaS的服务沟通，完成以前只有后端逻辑可能完成的工作，如身份验证 （例如身份验证服务平台[Auth0](https://auth0.com/), 和博客评论平台[Disqus](https://disqus.com/)）。
2. 数据库也被包装成了Serverless的服务，前端甚至可以直接访问数据库服务的内容（注：生产环境当中不常见，也不推荐，但是理论上可行）。
3. 把前两点结合起来，我们可以说一部分后端的功能，由于BaaS服务的出现，被转移到了前端。
4. 后端依然保有自己的业务逻辑，例如网站搜索功能。这部分功能可以通过一个FaaS函数实现，每次一个请求都会（通过API Gateway）出发一个新的FaaS函数，函数读取后端数据库，返回搜索结果。
上面的例子中，BaaS和FaaS的结合，将原先单一的后端应用拆分为多个相互独立的服务。这和微服务架构中choreography over orchestration的概念是不偶尔喝的。

### 案例二：事件触发的信息处理应用
与同步请求的web应用不同，很多后端应用需要处理大量的异步的触发事件。例如一个广告处理系统，每当一个用户点击产生一个用户事件，随后被后端应用处理并存入数据库。无论后端应用是采用Push还是Pull的方式得到事件请求，后端服务器需要一直等待新的出发事件。
![示意图]({{ "/assets/what-is-serverless/event-driven-example.svg" | absolute_url }})
而考虑无服务器的解决方案：
![示意图]({{ "/assets/what-is-serverless/serverless-event-driven-example.svg" | absolute_url }})
每当一个用户事件到来，FaaS平台为该事件创造一个单独的处理函数，函数执行完事件处理逻辑后自动退出

（未完待续）
