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

## 解构FaaS函数既服务
### FaaS的特点 
各大云计算厂商都相继退出了自己的FaaS产品。最早的FaaS产品是在2014年10月发布的[hook.io](https://hook.io), 今年（2018）加入FaaS的有阿里云的函数计算Compute Function，和腾讯云的无服务器云函数SCF。所有FaaS产品的结构和功能都大同小异，并且都可以和BaaS的API网关服务搭配使用。
1. 随时调用。所有FaaS产品的编程模型中，都是用随时可以调用、用完直接退出的函数/代码块，取代了传统的长时间运行的服务器进程。虽然借助一些手段可以用FaaS模拟一个长时间运行的进程（注：在项目中处于节省时间的需要，我曾经尝试过用一个Lambda function运行一个for loop，直到运行时间超过Lambda规定的timeout，随后立即触发第二个Lambda。），但这不是FaaS的编程模型所提倡的。
2. 无状态（Stateless）。Fans掉用完直接退出，对用户而言每一个函数调用之间是没有联系的。如果想要保存计算结果，必须依赖于第三方（数据库，云盘，分布式缓存等）。
3. 不受语言和运行平台限制。虽然各个FaaS产品都有自己优先支持的语言，例如Lambda优先支持直接支持Java/Python/Go/JS等，但实际上完全可以在Lambdas函数运行时开启一个新的进程，安装一个新的Runtime（或者将源代码编译成binary executable），运行不直接支持的语言代码。各个厂商有自己优先直接支持的语言，大多是是出于business的考量。
4. 代码的部署流程极大简化。由于不再需要自己管理Infrastructure，代码部署简化为代码打包上传。
5. 无须担心弹性扩容。无论是1000个TPS和1个TPS，对用户而言没有任何区别，无需考虑资源的调度分配。
6. 支持异步事件触发。很多应用场景下FaaS在系统中起到胶水的作用，充当多个应用之间信息传递的Adaptor。以Lambda为例，在AWS的网页控制台里，用户可以直接将Lambda和十几种AWS的服务直接连接。用户甚至可以将Lambda部署到CDN节点，通过CDN触发Lambda，在网络边缘节点完成计算（也就是所谓的边缘计算，Edge Computing，此处是一大坑）（TODO：此处插图）
7. 可以与API网关应用整合，构建Rset API（注：这种情况下API网关实际起到了传统web应用框架中Routing的作用）。
### 开源实现
FaaS开源项目主要围绕在FaaS的工具／应用框架，和开源的FaaS平台实现两方面展开。
对应工具和应用框架，开源社区活跃的项目有：
* [Serverless Framework](https://serverless.com/)： 一套整合了各个云厂商Serverless产品的工具包，本身提供的Cli相比原声工具更为简单易用。
* Serverless Application Model [SAM](https://github.com/awslabs/aws-sam-cli)：AWS开源的针对AWS服务的部署构建工具包。
* [Zappa](https://github.com/Miserlou/Zappa)：给予python的Serverless构建工具
* [Apex](https://github.com/apex/apex)：创建AWS Lambda应用的工具包

其余的开源项目者着眼于开源的FaaS平台实现：
* [Apache OpenWhisk](https://github.com/apache/incubator-openwhisk)： Apache的开源FaaS实现，最早由IBM领导，后贡献给开源社区。IBM的FaaS产品[Cloud Function](https://www.ibm.com/cloud/functions)就是构建在Apache OpenWhisk之上的。
* [Azure Function](https://github.com/Azure/azure-functions-host)：微软将自家Azure Function的Runtime开源（注：突然想到微软爸爸买下了Github之后，会不会将来贡献更多开源项目）。
* [OpenFaaS](https://github.com/openfaas/faas)：目前最火的FasS开源实现。整套项目用Golang写成，配套的测试／监控／工具比较完善。2017年OpenFasS的创建者展示了[如何将OpenFass部署在树莓派集群上](https://blog.alexellis.io/your-serverless-raspberry-pi-cluster/).

（未完待续）
