---
title: API技术，REST｜GraphQ｜RPC
subtitle: 
date: 2024-08-19T20:02:19+08:00
slug: 1c1a10b
draft: true
tags:
  - 计算机网络
categories:
  - 计算机网络
password: 
message:
---
# 什么是API？

API，全称为 application programming interface，意思是应用程序接口，负责多个软件之间的调用、交互。

现在的 API 往往是前后端的资源交换渠道，API 的设计和架构往往关系到双方的工作量和代码是否优雅。

# REST

REpresentational State Transfer，资源的表示性状态传输。

## 6个约束

- 客户端和服务端
- 统一接口
- 无状态
- 缓存
- 分层系统
- 代码按需加载

## 缺陷

- 获取不足和过度获取，比如有的功能实现依赖于多个 API 的搭配，有的功能却只需要某个资源的一小个子集。
- 随着系统复杂度加深，调用链路逐渐增长，总 RT 增加，排错困难。
- 很难应对需求变更，频频打补丁。
- 十分依赖文档编写。

# GraphQL

不再是【前端】-【后端】的交互，而是【前端】-【GraphQL 中间层】-【后端】，中间的 GraphQL 是服务器性质的，定义一些双方约定的数据模型 schema，通常有着严格的类型校验，GraphQL 服务器负责接收前端的资源请求，再去后端服务获取数据。

GraphQL 也有 REST 类似的语义，它规定了 Query、Mutation、Subscription 这些操作，其中 Subscription 更是实时交互的利器，还可以实现推送、轮询等功能。

相比 REST 的优势是，不再需要多个请求了，而是一个 URL 对应一大批资源。对于后端程序员来说把 N 个小请求变成了 1 个大请求，工作量也许不相上下，但复杂度高了很多，风险也集中了。对于前端程序员来说，获取资源更加方便了，再也不必因为缺少特定需求的 API 而等待后段程序猿帮你写，实现了双方人员的解耦。

似乎在国内没有很好的普及开来，可能有营销的原因，也可能因为国内软件开发环境和需求环境不同于外面。

参考一下：

[# 最心疼前端程序员的架构【让编程再次伟大#13】](https://www.bilibili.com/video/BV1iE421w7AN/?spm_id_from=333.999.0.0&vd_source=2217ffdee0afbd23565ec6a929840035)

[# 【GraphQL Codegen】神器，前端无法拒绝的开发体验](https://www.bilibili.com/video/BV1ra4y127eK/?spm_id_from=333.999.top_right_bar_window_history.content.click&vd_source=2217ffdee0afbd23565ec6a929840035)

## 缺陷

- 后端优化查询会十分困难
- 开发成本高、学习成本高

# RPC

远程服务调用，调用一个其他机子上的函数，写法就像本地调用一样

## 常见疑问

- 为什么要远程？
	- 通常在微服务场景下使用，多个服务各自负责逻辑独立的模块，当需要借助其他模块的功能时就是一个远程调用的场景。
- RPC 和 HTTP 有什么区别？
	- RPC 是一种思想，在这个层面上 HTTP 就是一种 RPC
	- 有一些实现比如 gRPC，他是基于 HTTP2 来实现的
- 如何实现数据的传输？
	- 通过序列化和反序列化的方式，将数据从内存的格式转换为适合网络传输的格式。
- 性能瓶颈在哪？
	- 网络IO、数据压缩方式

