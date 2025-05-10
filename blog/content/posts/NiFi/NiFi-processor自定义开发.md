---
title: NiFi Processor自定义开发：从零开始创建自己的第一个Processor
subtitle: 
date: 2024-07-08T11:35:50+08:00
slug: 247fe3e
tags:
  - NiFi
categories:
  - NiFI
password: 
message: 
draft: false
---

# 什么是NiFi Processor？

Apache NiFi 是一个数据流自动化同步工具，它本身提供了丰富的内置组件来处理数据集成，包括了各种数据源的读写，数据的拆分聚合，以及自定义脚本的处理。它们就是一个个 `processor` ，负责独立的子功能，多个 `processor` 连接在一起，构成复杂的 `processor group` 数据流任务。

# 为什么要自己开发？

其实， `Apache NiFi` 为了满足对于数据流处理的定制化需求，提供了 `ExecuteScript` 这样一个内置的 `processor` ，能够支持自定义 python、Groovy、lua、JS 脚本，但开发人员需要额外写一个脚本放到这个组件的 `Script Body` 中执行。

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407081526572.png)


让我们来分析一下这么做有什么问题：
- 增强了耦合，不利于复用，维护性差，太"专"了，类似的需求能不能抽象成模板？
- 很多时候，是算法组的成员先实现了数据处理的功能，比如一个模板映射的 SDK 、一个标签识别的 SDK ，这时候还要转换为脚本，本末倒置。
- 性能较低，脚本是解释执行的，相比编译后的 Java 代码性能较差
- 测试不方便，必须放入 `processor` 中并且让整个数据流跑起来，才能发现问题

所以，如果只是做简单的任务和快速验证，那么可以选择 `ExecuteScript` 处理器，但如果任务复杂、开发周期长、对性能有更高的要求，都推荐自定义开发 `Processor` ，事实上，当自己的需求无法被 NiFi 现有 `Processor` 满足时，都需要自定义一个。

# 创建 NiFi Processor 的基本步骤

## 了解文件结构

我们下载 NiFi 的源码，可以看到 `/nifi-external` 下有一个 `nifi-example-bundle` 的包，这是他内部的结构树

``` shell
.
├── nifi-nifi-example-nar
│   └── pom.xml
├── nifi-nifi-example-processors
│   ├── nifi-nifi-example-processors.iml
│   ├── pom.xml
│   └── src
│       └── main
│           ├── java
│           │   └── org
│           │       └── apache
│           │           └── nifi
│           │               └── processors
│           │                   ├── JsonProcessor.java
│           │                   └── WriteResourceToStream.java
│           └── resources
│               ├── META-INF
│               │   └── services
│               │       └── org.apache.nifi.processor.Processor
│               └── file.txt
└── pom.xml
```

它就是 NiFi 官方提供的扩展方式，提供了一个小 demo ，其中 `nifi-nifi-example-nar` 下只有一个 pom 文件：

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements. See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License. You may obtain a copy of the License at
  http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.apache.nifi</groupId>
        <artifactId>nifi-example-bundle</artifactId>
        <version>1.23.2</version>
    </parent>

    <artifactId>nifi-nifi-example-nar</artifactId>
    <packaging>nar</packaging>
    <properties>
        <maven.javadoc.skip>true</maven.javadoc.skip>
        <source.skip>true</source.skip>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.nifi</groupId>
            <artifactId>nifi-nifi-example-processors</artifactId>
        </dependency>
    </dependencies>
</project>

```

他所做的就是把主要功能部分的 `nifi-nifi-example-processors` 的 jar 包打包为 nar 包。

`nifi-nifi-example-processors` 是主要部分，我们需要编写自定义的 `Processor` ，并让他继承 `AbstractProcessor` ，重写其中的方法，并在 resources 的 META-INF 下声明实现类的全路径，有点类似 `spring-boot-starter` 的开发方式。

我们要做的就是：
1. 在 `nifi-nifi-example-processors` 的 src 下开发自定义的组件
2. 在 `nifi-nifi-example-processors` 的 resources 下声明自定义组件的全路径
3. 打包为 nar 包（ NiFi 自定义的）
4. 部署 nar 包到 nifi 环境的 lib 下

## 官方核心代码介绍

官方提供了一个自定义的 `Processor` ，叫 `WriteResourceToStream` ，以他为例我们可以了解一个自定义 `Processor` 应该如何开发。

``` Java
@Tags({ "example", "resources" })
@CapabilityDescription("This example processor loads a resource from the nar and writes it to the FlowFile content")
public class WriteResourceToStream extends AbstractProcessor {

    public static final Relationship REL_SUCCESS = new Relationship.Builder()
            .name("success")
            .description("files that were successfully processed").build();
    public static final Relationship REL_FAILURE = new Relationship.Builder()
            .name("failure")
            .description("files that were not successfully processed").build();

    private Set<Relationship> relationships;

    private String resourceData;

    @Override
    protected void init(final ProcessorInitializationContext context) {

        final Set<Relationship> relationships = new HashSet<Relationship>();
        relationships.add(REL_SUCCESS);
        relationships.add(REL_FAILURE);
        this.relationships = Collections.unmodifiableSet(relationships);
        final InputStream resourceStream = getClass()
                .getClassLoader().getResourceAsStream("file.txt");
        try {
            this.resourceData = IOUtils.toString(resourceStream, Charset.defaultCharset());
        } catch (IOException e) {
            throw new RuntimeException("Unable to load resources", e);
        } finally {
            IOUtils.closeQuietly(resourceStream);
        }

    }

    @Override
    public Set<Relationship> getRelationships() {
        return this.relationships;
    }

    @OnScheduled
    public void onScheduled(final ProcessContext context) {

    }

    @Override
    public void onTrigger(final ProcessContext context,
            final ProcessSession session) throws ProcessException {
        FlowFile flowFile = session.get();
        if (flowFile == null) {
            return;
        }

        try {
            flowFile = session.write(flowFile, new OutputStreamCallback() {

                @Override
                public void process(OutputStream out) throws IOException {
                    IOUtils.write(resourceData, out, Charset.defaultCharset());

                }
            });
            session.transfer(flowFile, REL_SUCCESS);
        } catch (ProcessException ex) {
            getLogger().error("Unable to process", ex);
            session.transfer(flowFile, REL_FAILURE);
        }
    }
}
```

我们分模块来解析一下各个部分的作用：

### 1. 注解和声明

``` Java
@Tags({ "example", "resources" })
@CapabilityDescription("This example processor loads a resource from the nar and writes it to the FlowFile content")
public class WriteResourceToStream extends AbstractProcessor {
```

- @Tags 用于给 processor 添加标签，可以在选择 processor 时快速检索
- @CapabilityDescription 是描述处理器的功能

### 2. 定义关系与成员变量

``` Java
public static final Relationship REL_SUCCESS = new Relationship.Builder()
        .name("success")
        .description("files that were successfully processed").build();
public static final Relationship REL_FAILURE = new Relationship.Builder()
        .name("failure")
        .description("files that were not successfully processed").build();

private Set<Relationship> relationships;  
  
private String resourceData;
```

- 其中 `REL_SUCCESS` 和 `REL_FAILURE` 表示与下一个任务的转移关系，在后面可以使用 `session.transfer(flowFile, REL_SUCCESS);` 来转移，因此也可以定义其他类型比如warn、match等
- 定义需要的成员变量，比如这里的 `resourceData` 就是这个 demo 功能所需的定义

### 3. 初始化方法

``` Java
@Override
protected void init(final ProcessorInitializationContext context) {
    final Set<Relationship> relationships = new HashSet<Relationship>();
    relationships.add(REL_SUCCESS);
    relationships.add(REL_FAILURE);
    this.relationships = Collections.unmodifiableSet(relationships);

    final InputStream resourceStream = getClass()
            .getClassLoader().getResourceAsStream("file.txt");
    try {
        this.resourceData = IOUtils.toString(resourceStream, Charset.defaultCharset());
    } catch (IOException e) {
        throw new RuntimeException("Unable to load resources", e);
    } finally {
        IOUtils.closeQuietly(resourceStream);
    }
}

```

- 把定义的关系加入到关系集合中，给需要的成员变量赋值，这里是根据 demo 的功能，加载了一个资源文件到成员变量

### 4. 调度方法

``` Java
@OnScheduled
public void onScheduled(final ProcessContext context) {
}
```

- 该方法在处理器倍调度时调用，这里没有体现

### 5. 触发方法

``` Java
@Override
public void onTrigger(final ProcessContext context,
        final ProcessSession session) throws ProcessException {
    FlowFile flowFile = session.get();
    if (flowFile == null) {
        return;
    }

    try {
        flowFile = session.write(flowFile, new OutputStreamCallback() {
            @Override
            public void process(OutputStream out) throws IOException {
                IOUtils.write(resourceData, out, Charset.defaultCharset());
            }
        });
        session.transfer(flowFile, REL_SUCCESS);
    } catch (ProcessException ex) {
        getLogger().error("Unable to process", ex);
        session.transfer(flowFile, REL_FAILURE);
    }
}

```

- 核心逻辑，在运行组件时就会调用触发方法
	- 这里的 `session` 是全局会话，可以认为 `session` 中的 `flowFile` 也是全局的数据流，因此我们可以在每一步给 `flowFile` 添加属性，以供后面的节点方便使用
	- try-catch 里的就是对 `flowFile` 的具体操作了，这里是在数据流中写入了 `resourceData` 
	- 如果成功就转移到 `REL_SUCCESS` 关系，否则就记录日志并转移到 `REL_FAILURE` 

## 自己动手实现一个

想自己根据需求写一个 processor 有两方面的难点，一方面是自己的功能如何实现，另一方面是如何利用官方封装的组件更好地开发。这里我们关注第二方面，因为官方提供的 demo 实在简陋，遗漏了一些开发所需的重要功能，比如属性。

功能：假设我需要对数据流进行敏感打标，具体的实现已经有SDK算子了，在这里只需要调用，并把打标结果存储到 `flowFile` 的属性中，以供后续节点处理并持久化

这里只展示一下额外需要新增哪些
### 1. 定义属性

``` Java
private List<PropertyDescriptor> properties;

public static final PropertyDescriptor TEXT_PROPERTY = new PropertyDescriptor
            .Builder().name("Text")
            .description("The text to be scanned")
            .required(true)
            .expressionLanguageSupported(ExpressionLanguageScope.FLOWFILE_ATTRIBUTES)
            .addValidator((subject, input, context) -> new ValidationResult.Builder()
                    .subject(subject)
                    .input(input)
                    .valid(input != null && !input.trim().isEmpty())
                    .explanation(input == null || input.trim().isEmpty() ? "text cannot be null or empty" : "")
                    .build())
            .build();
```

- `flowFile` 分为正文（content）和属性（Attributes）两部分，为了不影响实际数据内容，对于一些中间结果我们可以选择保存在属性中，这里设置了一个名为 `Text` 的属性，并支持 EL 表达式，以从上一个节点的数据流提取 / 继承属性信息

### 2. 触发部分实现

```Java
@Override
    public void onTrigger(ProcessContext context, ProcessSession session) throws ProcessException {
        FlowFile flowFile = session.get();
        if (flowFile == null) {
            return;
        }

        final String text = context.getProperty(TEXT_PROPERTY).evaluateAttributeExpressions(flowFile).getValue();

        try {
            ClassifyScanService classifyScanService = new ClassifyScanService();
            Collection<ScanResult> textScanResults = classifyScanService.labelTextWithPolicy(text);

            // Convert scan results to a JSON string
            ObjectMapper objectMapper = new ObjectMapper();
            String resultString = objectMapper.writeValueAsString(textScanResults);

            // Set the result as an attribute
            flowFile = session.putAttribute(flowFile, "result", resultString);

            session.transfer(flowFile, REL_SUCCESS);
        } catch (JsonProcessingException e) {
            getLogger().error("Failed to convert scan results to JSON", e);
            session.transfer(flowFile, REL_FAILURE);
        } catch (Exception e) {
            getLogger().error("Failed to process FlowFile", e);
            session.transfer(flowFile, REL_FAILURE);
        }
    }
```

- 这里获取到属性值后，将其进行敏感打标处理，存入一个集合中
- 将扫描结果转换为 JSON ，并写入到 `flowFile` 的新属性 `result` 中
- 进行关系转移

## Deployment

- 打包：执行 mvn clean install 命令进行打包
- 上传：在 `nifi-nifi-example-nar` 的 target 下找到 nar 包，上传至 NiFi 部署目录的 lib 目录下。
- 重启：在 NiFi 部署目录下执行sh脚本重启服务
- 进入 UI 界面开始使用！

