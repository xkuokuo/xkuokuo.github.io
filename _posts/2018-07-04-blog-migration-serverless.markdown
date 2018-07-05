---
layout: post
title:  "去服务器化的尝试：从Wordpress迁移到Github Pages+ AWS Serverless"
date:   2018-07-04 12:13:28 -0700
excerpt: 记录一下自己把Blog从Wordpress迁移到Serverless架构的尝试。
categories: Wordpress Blog Serverless
---

决定把自己运行Wordpress的AWS服务器关了。虽然选的是最便宜的EC2 t2.micro，一个月也要十刀。最让人头疼的是DDOS，不知道为什么从Blog上线第一天起就频繁的[DDOS攻击](https://www.weibo.com/2663109067/EmnjOrM89)，量不大，但是Mysql数据库（部署在同一台EC2上）很容易就挂了。来来回回折腾这么几次，也没心情再更新。

## 自己的需求
想来想去自己的需求就这么几个：
1. 少花钱。访问量本来就不大，没必要每个月给AWS上供（此处忍不住吐槽一下公司，也不给员工打个折扣）。
2. 稳定，至少不会轻易挂掉。
3. 简单轻便，尽量不要太过于依赖于AWS的服务，也不要太依赖于某一个CSM框架，将来迁移方便。
4. 如果能做到，全栈Serverless。毕竟在AWS Lambda工作，应该亲自感受一下作为用户将整套服务迁移到Serverless的体验。

## 流行的方案
在网上胡乱搜了一天，可能的备选方案：
### [Github Pages](https://pages.github.com/)

Github推出的静态网站/页面托管服务，直接把一个github的repo变成一个静态网站，用户需要做的只剩下提供HTML/CSS/JS等静态资源。更为方便的是Github和[Jekyll](http://jekyllrb.com)的整合。Jekyll是一个Ruby写的静态网站生成器，可以将Markdown语法的文档转换为HTML，并且对Blog形式的网站有很好的支持。

Pros:
* 不，要，钱
* 简单易维护
* 支持Markdown语法，也可以直接手写HTML/JS/CSS
* Jekyll本身对Blog就有很好的支持，自带Blog模板
* 博客更新流程方便：将repo下载到本地，直接在vim里（vim大法好）编辑，本地测试完之后直接push上去

Cons:
* 只支持静态资源。如果想要和后端服务有一些交互，还需要另外想办法
* 没有现成的metrics和monitoring，最基本的访问量数据也没有
* 不付费的话（$7一个月）自己的博客的repo只能是public

### [AWS S3静态网站托管](https://aws.amazon.com/getting-started/projects/host-static-website/services-costs/)

其实和Github Pages一样，将网站静态资源存放在S3的bucket上。

Pros：
* 正常情况下几乎不要钱。S3按文件对象大小和读写次数收取[费用](https://aws.amazon.com/govcloud-us/pricing/s3/)。简单易维护。如果后续添加后端API，可以和API一起部署在AWS上

Cons：
* 博客更新需要“手动”上传生成好的HTML。亦或者可以利用[Githubt提供的Webhook](https://developer.github.com/webhooks/)，实现代码自动打包上传
* 如果被DDOS，大量的S3读取请求会会产生[额外开销](https://www.reddit.com/r/aws/comments/7z6uc3/ddos_potential_cost/)。S3不直接支持限流，需要针对请求数等real-time metrics监控，当大规模异常请求到来时，手动或自动关闭S3的endpoint。

### AWS Serverless Application Model [SAM](https://github.com/awslabs/serverless-application-model)

AWS力推的Serverless programming model，基于AWS APIGateway, AWS Lambda，和AWS DynamoDB，构建一个Serverless的应用，本质上就是提供了一套CLI，将AWS已有的服务打个包推销给客户。。。

此外AWS SAM还有一个基于Node.js的[子项目](https://github.com/awslabs/aws-serverless-express)，可以直接将一个Express应用包在一个Lambda function里。静态资源也可以一起打包上传，并通过API Gatewway调用。上手玩了一下，对于小的应用的确是很方便。

Pros:
* 正常情况下花费几乎为零。以AWS Lambda为例，一百万次Lambda运行收取的费用是二十美分（如果我有一百万个有效访问，我就不上班了）。
* API Gateway支持限流，可以设置API的最大TPS。
* 可以与前两种方案结合，前后端彻底分离，Serverless作为后端API。
* 可以直接使用CloudWatch等现成的监控工具。
* 自己的域名目前托管在AWS Route53上，与其他AWS服务整合非常方便。
Cons:
* 过度依赖于AWS服务。

## 目前的解决方案

1. 前后端彻底分离。前端静态资源部署在Github Pages，一些简单的后端API（统计监控）放在AWS。
2. 前端页页面生成使用Jekyll。

![示意图]({{ "/assets/blog_system_diagram.png" | absolute_url }})

虽然用了AWS API Gateway，AWS Lambda和AWS DynamoDB，但考虑各家云服务商都有类似产品，并且后端API目前只负责流量监测等简单功能，即使遇到特殊情况需要迁移后端API，保存在Github Pages上面的文章不会变。Markdown格式的post也方便以后迁移到其他平台。
