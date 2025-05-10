---
title: 使用NiFi实现OpenAPI：构建可订阅的数据服务
subtitle: 
date: 2024-07-31T03:36:48+08:00
slug: 2eb3078
draft: false
tags:
  - NiFi
categories:
  - NiFI
password: 
message:
---
# 需求

RESTful API 作为应用程序间通信的一种标准方式，通过 HTTP 请求对数据进行操作。尽管它们设计为开放接口以适应不断变化的需求，但在实践中，它们往往与应用程序紧密耦合，这可能导致开发过程中的复杂性和臃肿。

在这些开放接口中，特别是涉及数据订阅的接口，我们之前的系列文章提到 Apache NiFi 特别擅长管理数据流。自然而然，一个问题出现了：NiFi 是否能够将数据流处理的结果封装为 OpenAPI 调用的一部分并公开？答案是肯定的。

我们的目标是实现数据订阅类 OpenAPI 的解耦，将其从复杂的系统架构中独立出来，并建立一套开发规范，交给 NiFi 去实现。通过这种方式，我们可以显著提高开发效率并简化维护成本。

# 实现思路

NiFi 有两个非常适合处理 HTTP 请求的处理器：`HandleHTTPRequest` 和 `HandleHTTPResponse` ，前者用于监听特定端口的 HTTP 请求，后者用于产生响应结果，我们可以在二者中间补充数据处理逻辑。

## HandleHTTPRequest

### 基础信息设置

对于请求处理程序，我们只需要确定监听的端口，以及支持的 HTTP 方法，比如是否支持 GET、POST 等，还有 Allowed Paths 就是请求路径，比如 /user，这里我们可以填 * 也可以不填，理由在后文会提到。

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407310447615.png)

### URL 分发问题

这里要注意的是，我们的多个 Open API 可能监听同一个端口，但 NiFi 不允许多个 `HandleHTTPRequest` 实例同时监听同一个端口，这就意味着我们需要使用单一的 `HandleHTTPRequest` 处理器来统一处理所有进入的 HTTP 请求。

为了实现这一点，我们将利用 NiFi 的能力来区分不同的 URL 路径或查询参数，从而实现对不同 API 端点的请求分发。这通常涉及到对请求的 URL 或其他属性进行解析，并根据这些属性将请求路由到相应的处理逻辑。

这里我们使用到了 `RouteOnAttribute` 处理器，他可以根据传递过来的 url 参数来 match 不同的规则，我们只需要做好从 url 参数到路由结果的规则定义。类似于普通处理器的默认 `success` 和 `failure` 两种 `Relationship`，`RouteOnAttribute` 允许我们定义更多样化的路由路径。例如，我们可以基于 URL 中的特定参数或路径片段，设置如 `toUser` 或 `toRole` 等自定义路由规则，从而实现更为灵活和细致的请求分发策略。

此外，`RouteOnAttribute` 提供的路由灵活性也意味着我们可以轻松扩展新的路由规则，以适应未来可能出现的新需求或变化，而无需对现有工作流进行大规模的修改。

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407241804294.png)
 
### 参数设置以及接口文档

一个全面的 OpenAPI 规范不仅定义了接口的交互方式，还应包括详尽的接口文档，这通常涵盖请求方法、请求 URL、输入参数、输出参数以及请求和响应的示例等信息。传统开发流程中，我们往往先确定接口规范，然后据此创建接口文档，有时也会使用工具如 Swagger 自动生成文档。

在 NiFi 中使用 `HandleHTTPRequest` 处理器时，我们可以灵活地设置各种类型的请求参数，并支持将这些参数定义为自定义属性，以适应不同请求的需求。然而，对于接口文档的生成，NiFi 并没有内建的解决方案。

为了解决这个问题，我们提出了一种轻量级的方法，旨在最小化手动编写工作量的同时，以类似 Swagger 的格式提供接口文档。我们的方法包括在每个 OpenAPI 实现中引入一个 `UpdateAttribute` 处理器。这个处理器在数据流处理中不扮演任何角色，但它用于标准化和记录接口文档所需的关键字段。

由于我们的 `HandleHTTPRequest` 是统一处理所有 API 请求的，很多接口的详细信息直到请求实际发生时才会动态生成。这就要求我们不能依赖于动态生成的信息来创建文档，而是需要在每个 API 的 `UpdateAttribute` 处理器中预先定义和记录接口文档的相关信息。

通过这种方法，我们可以确保每个 OpenAPI 都有一个清晰、一致且预先定义的文档记录，这有助于维护和未来的开发工作，同时也可以作为团队成员理解和使用 API 的参考。

为了展示和整合接口信息，我们为 NiFi SDK 新增了对于 Open API 的支持，我们约定接口文档的数据模型：

```Java
@Data
public class ApiDocument {
    private String method;
    private String url;
    private String description;
    private List<Parameter> requestParameters;
    private List<Parameter> responseParameters;
    private String requestExample;
    private String responseExample;

    @Data
    public static class Parameter {
        private String name;
        private String type;
        private String description;
        private boolean required;

    }
}
```

我们目前是通过流程组名的 `-openapi` 后缀来定位到 Open API 类型的流程组，以及通过 `-config` 来定位接口文档相关处理器的，具体逻辑如下

```Java
// 获取所有openapi的config processor
public JsonNode getOpenApiProcessGroups(String parentGroupId) throws Exception {
	List<ApiDocument> apiDocuments = new ArrayList<>();
	getProcessGroupsRecursive(parentGroupId, apiDocuments);

	ApiDocumentUtils apiDocumentUtils = new ApiDocumentUtils();
	List<JsonNode> documents = new ArrayList<>();
	for (ApiDocument apiDocument : apiDocuments) {
		documents.add(apiDocumentUtils.toJson(apiDocument));
	}
	return new ObjectMapper().createArrayNode().addAll(documents);
}

private void getProcessGroupsRecursive(String parentGroupId, List<ApiDocument> apiDocuments) throws Exception {
	String url = getNifiUrl() + PROCESS_GROUPS_ENDPOINT + parentGroupId;
	HttpGet get = new HttpGet(url);

	executeRequest(get, rootNode -> {
		rootNode.get("processGroupFlow").get("flow").get("processGroups").forEach(processGroup -> {
			String groupName = processGroup.get("component").get("name").asText();
			if (groupName.endsWith("-openapi")) {
				String childGroupId = processGroup.get("component").get("id").asText();
				try {
					getProcessGroupsRecursive(childGroupId, apiDocuments);
				} catch (Exception e) {
					throw new RuntimeException(e);
				}
				// 获取processors ending with -config
				try {
					getProcessors(childGroupId).forEach(processor -> {
						String processorName = processor.get("component").get("name").asText();
						if (processorName.endsWith("-config")) {
							ApiDocumentUtils apiDocumentUtils = new ApiDocumentUtils();
							ApiDocument apiDocument = null;
							try {
								apiDocument = apiDocumentUtils.parseConfigProcessor(processor);
							} catch (JsonProcessingException e) {
								throw new RuntimeException(e);
							}
							apiDocuments.add(apiDocument);
						}
					});
				} catch (Exception e) {
					throw new RuntimeException(e);
				}
			}
		});
		return null;
	});
}
```

在成功捕获并记录了所有 OpenAPI 接口的详细信息之后，我们便可以将这些信息传递给前端来展示了。Swagger 通过将文档编写工作转变为编写代码的方式，极大地提高了文档的准确性和可维护性。同样，我们的方法通过将文档编写工作转化为配置 NiFi 处理器的属性，也实现了类似的目标。这种转变不仅简化了文档的生成过程，而且确保了文档与实际接口实现的同步更新，从而减少了手动维护文档的负担和出错的可能性。

## HandleHTTPResponse

响应处理则更简单，只需要在之前的流程中处理好数据的展示，在此处只设置一个状态码即可，大部分情况下都是成功状态码200，如果有一些其他复杂需求，则需要在中间过程中做好路由，各自连接到不同状态码的 `HandleHTTPResponse` 处理器。

以下是一个普通的案例，只需要查库并做一些格式转换，然后响应，只做了成功的结果。

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407241804994.png)

## 调用方式

当 Open API 的流准备就绪后，我们就可以调用它，通过 `https://<ip>:<port>/url` 的路径，并设置需要的请求头和参数即可。这里写的时候发现 postman 的 history 没有保留历史数据信息的 body，没内网一时也访问不到，但其他信息还是在的，还是展示一下吧

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202407310555262.png)

# 总结

我们已经确立了在 Apache NiFi 上开发 Open API 的基础策略，目的是将主要的数据处理和数据订阅功能迁移至 NiFi 平台。这一转变有助于减少对主系统的依赖，避免了因小幅度修改开放接口而需重新部署整个系统的问题。目前，我们展示的案例较为基础，并未涵盖生产环境中可能遇到的复杂情况。对于更深入的实施细节和高级应用，建议参考本系列的其他以及后续文章~~（其实我也懒得写实施细节🐶）~~。