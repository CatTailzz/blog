---
title: NiFi调研报告
subtitle: 
date: 2024-07-02T01:47:25+08:00
slug: 291cb4c
draft: false
tags:
  - NiFi
categories:
  - NiFI
password: 
message:
---
# 背景

为了解决过去数据同步问题中，不同人员实现方式不一、时长较难预估的问题，决定引入公共的数据同步组件。通过统一的开发工具和标准化开发流程，确保任务完成的及时性和一致性。

# 目标

- 提效数据同步及 OpenAPI 开发，（从 x~xx 天 提效为 xx分钟~x天 ）
- 完成相关功能历史场景的可行性验证，确认方案的覆盖场景和覆盖率。
- 解决多源数据同步问题，包括字段映射、加工、性能、权限等问题

# NiFi介绍

Apache NiFi 是一个易于使用、功能强大而且可靠的数据拉取、数据处理和分发系统，用于自动化管理系统间的数据流。

它支持高度可配置的指示图的数据路由、转换和系统中介逻辑，支持从多种数据源动态拉取数据。

NiFi原来是NSA(National Security Agency [美国国家安全局])的一个项目，目前已经代码开源，是Apache基金会的顶级项目之一

用户可以为数据处理定义为一个流程，然后进行处理，后台具有数据处理引擎、任务调度等组件。

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020233348.png)

# 调研结果

## 目标一：数据源适配问题

问题：是否适合不同数据源之间的数据同步包括但不限于kafka、syslog、mongodb、openapi、clickhouse 之间的同步问题？

结论：适合各种数据源之间的数据同步，包括向外开放OpenAPI及调用OpenAPI获取数据

方案验证①：mysql—>Kafka数据同步

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020233252.png)

方案验证②：Kafka—>Kafka数据同步

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020234291.png)

方案验证③：openAPI->kafka数据同步

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020234544.png)

方案验证④：mongoDB->kafka数据同步

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020234756.png)

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020205086.png)

## 目标二：多数据源关联

问题：是否可在处理过程中引入第三方数据源

结论：可通过调用多个数据源，动态传递查询参数

方案验证①：从kafka拿取数据，根据拿到的json字符串对应属性值去mongo中查询—>将查询出的数据吐到kafka

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020234776.png)

## 目标三：对数据流进行加工

问题：如何方便快速地对数据进行清洗、聚合等加工操作。

结论：可通过组件或脚本的方式对数据源进行加工处理

方案验证①：从kafka拿取数据，根据拿到的json字符串对应属性值去mongo中查询—>将查询出的数据吐到kafka

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020235075.png)

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020235770.png)

## 目标四：数据同步调度策略

问题：是否支持调度策略的配置，是否需要引入第三方调度工具

结论：可在节点面板中配置对应的同步策略，配置对应周期

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020235312.png)

## 目标五：将数据输出为文件

问题：是否支持数据输出持久化到日志文件

结论：可通过相关组件将数据输出为对应格式

方案验证①：从mysql拿数据，并将数据输出到指定目录

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020235246.png)

## 目标六：根据数据属性进行路由

问题：是否支持数据处理中的多策略路由分发

结论：可根据对应数据中的json属性分发数据到不同接收器中

方案验证①：消费kafka数据，根据json字符串角色属性分发到不同的kafkaTopic中

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202408020235623.png)

# 运行时注意事项

## 最小运行配置

推荐配置在2h6g，依据数据量及性能要求制定

## 产品适配方式 

1. iframe 方式集成到后台管理界面
2. 自主实现前端，openapi 方式接入
3. 开源项目 fork 一份后，自主维护（扩散成通用编排底座？）

## 任务配置管理

已配置的任务可导出为 xml 格式，在需要时引入

# NIFI测试报告

## 单节点压力测试

容器资源显示为2h6g

200个节点，28条任务流资源占用2h5.5G

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231502431.png)

![](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407231502852.png)

## 数据量测试

1. NIFI自动重启后，队列中缓存的数据不会被清空，自动重启后任务会继续执行。
2. 容器资源配置不小于2H6G。
3. 生成100万条数据吐到kafka，NIFI从kafka拉取数据并进行一字段映射，拼接操作并插入mysql，处理效率约为1分钟15万条左右,2500条/s，可通过增加线程数量优化，达到达到10000条/s

## NiFi 高性能调优分析

我们以数据同步为例，整个链路中有性能瓶颈的为以下几处

- 数据发送源和数据写入方的读写能力瓶颈，这种可以参照通用解决方案
- 中间处理数据流的能力瓶颈
	- 资源瓶颈，NiFi 运行在 JVM 上，堆栈空间和垃圾回收策略都影响效率
	- 处理能力瓶颈，引入第三方数据源时的查效率、CPU密集型任务的计算效率、处理器本身对于任务的适配程度
- 多线程、集群部署替换单机，来提升性能

任何一处的瓶颈都会导致整个处理流的效率低下，最简单的例子，比如两个万级吞吐的处理器之间，只有一个容量为10的队列，那么这个中间队列就成了整个链路的瓶颈，无法发挥处理器的高性能优势。所以整个的思路就是哪里慢就去改哪里，我们已经提供了常见的改进方向。

# NIFI统一数据同步方案优势

1. 可复用性强、配置方便，可通过创建完成的模板，对配置进行属性修改，即可使用。
2. 降低排错成本，可通过web页面事实查看数据同步过程。
3. 降开发成本，发生需求变更时，只需修改配置，即可重新完成数据兼容。

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407241805646.png)
