---
title: NiFi调研
subtitle: 
date: 2024-07-23T14:53:28+08:00
slug: 8e9f633
draft: true
tags:
  - NiFi
categories:
  - NiFI
password: 
message:
---
# 1.0 背景

在过去一年中，根据统计API产品开发过程中涉及到的OpenAPI及数据同步的人天约为106人天，每个开发人员都有自己不同的解决方案，不同的数据同步工具，资源占用较高、开发效率较慢这一问题，提升这方面的开发效率迫在眉睫！

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231454869.png)

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231455064.png)

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231455957.png)

产研部门规划提效方案：项目定开提效任务—>后端定开提效——>统一数据同步/开放能力

目标：提升数据同步及OpenAPI开发提效30%（从 2~20 天 提效为 30分钟~半天 ）![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231456195.png)

## 2.0 NIFI组件介绍

Apache NiFi 是一个易于使用、功能强大而且可靠的数据拉取、数据处理和分发系统，用于自动化管理系统间的数据流。

它支持高度可配置的指示图的数据路由、转换和系统中介逻辑，支持从多种数据源动态拉取数据。

NiFi原来是NSA(National Security Agency [美国国家安全局])的一个项目，目前已经代码开源，是Apache基金会的顶级项目之一

用户可以为数据处理定义为一个流程，然后进行处理，后台具有数据处理引擎、任务调度等组件。

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/f48c45ac-ef75-40c4-8733-f4282f332a02.png)

## 3.0调研结果

### 3.1目标一：数据源适配问题

是否适合不同数据源之间的数据同步包括但不限于kafka\syslog\mongodb\openapi\clickhouse 之间的同步问题？

结论：适合各种数据源之间的数据同步，包括向外开放OpenAPI及调用OpenAPI获取数据

方案验证①：mysql—>Kafka数据同步

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/433e607a-0d5d-4340-9bb4-9a5188ae76a6.png)

方案验证②：Kafka—>Kafka数据同步

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/e437893e-05bd-4551-a1f3-155a73776be4.png)

方案验证③：openAPI->kafka数据同步

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/a60f194a-3151-45cf-9b1f-3166bb54726a.png)

方案验证④：mongoDB->kafka数据同步

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/a035a44a-9437-4c69-80a1-ebe9e7edc0b8.png)

### 3.2目标二：多数据源关联

结论：可通过调用多个数据源，动态传递查询参数

方案验证①：从kafka拿取数据，根据拿到的json字符串对应属性值去mongo中查询—>将查询出的数据吐到kafka

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/e2c8f24c-b13a-4e02-a884-a93f0ed35490.png)

### 3.3目标三：对数据流进行加工

结论：可通过组件或脚本的方式对数据源进行加工处理

方案验证①：从kafka拿取数据，根据拿到的json字符串对应属性值去mongo中查询—>将查询出的数据吐到kafka

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/227286f0-765d-4ac6-b945-cf0d2c0ee1aa.png)

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/bc3e45b6-9278-43a0-adef-6ae26f6d5a36.png)

### 3.4目标四：数据周期同步配置

结论：可在节点面板中配置对应的同步策略，配置对应周期

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/f8a13640-f159-45db-8bc2-e8eee8a46559.png)

### 3.5目标五：将数据输出为文件

结论：可通过相关组件将数据输出为对应格式

方案验证①：从mysql拿数据，并将数据输出到指定目录

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/928a1473-ad68-4ef4-ab20-34c5c119d2c3.png)

### 3.6目标六：根据数据属性进行路由

结论：可根据对应数据中的json属性分发数据到不同接收器中

方案验证①：消费kafka数据，根据json字符串角色属性分发到不同的kafkaTopic中

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/da1d6fbc-a29f-4e58-ae92-dc10c9b48822.png)

## 4.0 最小运行配置

推荐配置在2h6g，依据数据量及性能要求制定）

## 4.1 产品适配方式 

1、iframe 方式集成到后台管理界面

2、自主实现前端，openapi 方式接入

3、开源项目 fork 一份后，自主维护（扩散成通用编排底座？）

## 4.2 任务配置管理

可导出为 xml 文件进行管理

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/1f8f3b03-c58b-44a0-b782-7b64d83472f3.png)
















# 1.0 背景

       在过去一年中，根据统计API产品开发过程中涉及到的OpenAPI及数据同步的人天约为106人天，每个开发人员都有自己不同的解决方案，不同的数据同步工具，资源占用较高、开发效率较慢这一问题，提升这方面的开发效率迫在眉睫！

# 2.0目标

产研部门规划提效方案：项目定开提效任务—>后端定开提效——>统一数据同步/开放能力

目标：提升数据同步及OpenAPI开发提效30%（从 2~20 天 提效为 30分钟~半天 ）![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231500446.png)

# 3.NIFI介绍

Apache NiFi 是一个易于使用、功能强大而且可靠的数据拉取、数据处理和分发系统，用于自动化管理系统间的数据流。

它支持高度可配置的指示图的数据路由、转换和系统中介逻辑，支持从多种数据源动态拉取数据。

NiFi原来是NSA(National Security Agency [美国国家安全局])的一个项目，目前已经代码开源，是Apache基金会的顶级项目之一

用户可以为数据处理定义为一个流程，然后进行处理，后台具有数据处理引擎、任务调度等组件。

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/f48c45ac-ef75-40c4-8733-f4282f332a02.png)

# 4.数据源适配

 目前已完成OpenApi、Mysql、ClickHouse、MongoDB、Kafka、sysLog，共6种数据源同步的模板封装，及详细的文档说明。

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231500944.png)

# 5.数据流加工问题

可通过组件或脚本的方式对数据源进行加工处理

方案验证：从kafka拿取数据，根据拿到的json字符串对应属性值去mongo中查询—>将查询出的数据吐到kafka

![](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NpQlKaEE0Dz0qDvL/img/227286f0-765d-4ac6-b945-cf0d2c0ee1aa.png)

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231501250.png)

# 6.部署及更新方式

目前已将NIFI组件适配到K8s中封装为一个pod，并完成将配置我文件、日志进行挂载，在部署安装后，任务流配置及任务状态不丢失。

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231501033.png)

可通过固定地址进行访问

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231501554.png)

客户环境下任务配置修改：只需替换目录下配置文件，并重启容器，即可完成任务流修改。

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231502268.png)

# 7.NIFI测试报告

## 节点压力测试

容器资源显示为2h6g

200个节点，28条任务流资源占用2h5.5G

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231502431.png)

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231502852.png)

## 数据量测试

1.NIFI自动重启后，队列中缓存的数据不会被清空，自动重启后任务会继续执行。

2.容器资源配置不小于2H6G。

3.生成100万条数据吐到kafka，NIFI从kafka拉取数据并进行一字段映射，拼接操作并插入mysql，处理效率约为1分钟15万条左右,2500条/s，可通过增加线程数量优化，达到达到10000条/s

# 8.NIFI统一数据同步方案优势

1. 可复用性强、配置方便，可通过创建完成的模板，对配置进行属性修改，即可使用。
    
2. 降低排错成本，可通过web页面事实查看数据同步过程。
    
3. 降开发成本，发生需求变更时，只需修改配置，即可重新完成数据兼容。


![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407241803576.png)
![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407241804294.png)
![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407241804994.png)
![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407241805646.png)
