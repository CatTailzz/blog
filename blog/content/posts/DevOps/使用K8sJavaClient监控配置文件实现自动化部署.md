---
title: 使用K8sJavaClient监控配置文件，实现自动化部署
subtitle: 
date: 2024-07-23T16:16:16+08:00
slug: "8100467"
draft: false
tags:
  - K8s
  - DevOps
categories:
  - DevOps
password: 
message:
---
# 需求背景

公司的大部分产品是toB的，并且需要面对不同的客户做不同的定制化开发。以往的定开采取了特事特办，切新分支的方式，在准备环境、部署、资源消耗、可复用性方面都非常差，管理越来越混乱。为此引入了定开范式，以简化每次定开所需的成本，并更科学地管理不同定开版本。

作为本文的背景，再此做一些关于定开在后端实现方案的简单介绍。把项目全都挪到apps包下管理，其中app-start包作为大版本的基础包，提供的是产品的通用功能形态，而app-xxx包则是某客户的定开包，在依赖复用大版本的情况下实现定开功能的开发，最后把app-xxx打包部署交付即可。

本文要解决的是定开范式落地中，关于内部部署测试的解决方案。过去在测试时面临巨大的虚拟机压力和环境准备压力，在同时进行多个定开时还需要多个服务器，且版本管理混乱。现在想要在一台机器里就可以轻松实现同一大版本下不同定开版本的快速切换和功能验证，其实前面的后端实现方案已经做好了准备，只需要部署不同的app-xxx包就可以了。本文所做的本质就是把手动部署的动作以及一系列配置实现自动化，以最小的代价做快速切换。

在过去，公司在部署 K8s 服务时需要在多个环节手动修改配置和部署，特别是对于基础产品 Starter 和一系列能力包（abcd）的组合。这种手动操作不仅繁琐，还容易出错，严重影响了工作效率和部署的可靠性。

为了解决这些问题并提升部署效率，我们决定引入 DevOps 实践，实现自动化部署。我们从一个源头配置文件入手，通过监控最上游文件的变更，实现整条链路的自动化配置变更和服务部署。具体如下图

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202409042206874.png)


一个比较小的需求，正好可以熟悉一下 DevOps 的思想和部分实践。

# 实现计划

## 分析

由于整个任务是围绕着 K8s 服务来的，对比以往的手动操作，现在要做的就是把以前的操作包装并自动化，我们考虑使用 K8s 为 Java 提供的 client 来实现，把整个任务作为一个 pod 部署。

为了和原本的服务 pod 区分，我们管自己的任务叫 monitor pod，原本服务叫 
service pod。

## 如何监控源配置文件

监控文件变更有多种方式，最简单的可以把解析文件的过程写入 init 方法，然后在变更文件的同时去重启一下 monitor pod ，重新执行 init 方法。

要进一步自动化，我们可以在部署 monitor pod 时，用 Java 的 `WatchService` 监控文件，或者用 Linux 内核提供的 `inotify` 来监控，但综合对比了风险、复杂度和资源消耗等，暂时选择了简单的手动重启 pod 。

## 如何变更下游服务

这里的关键是对于配置文件中关键字段的一些约定，核心思路就是在检测到源配置文件变更时，触发解析动作，获取到关键字段值。用 K8s Java Client 去修改 service pod 的 deployment 中环境变量，接着对 service pod 做缩扩容重启，在重启过程中会有一个 sh 脚本继续获取 service pod 的环境变量，决定起哪个 jar 包，最终实现整个服务随配置文件的变更。

# 实现过程

## 监控源配置文件的实现

使用 `@PostConstruct` 注解，在应用启动时执行 init 方法，获取到配置文件并初始化

``` Java
@PostConstruct
public void init() {
	license = new License();
	String path = LICENSE_FULL_PATH;
	String meta = LICENSE_STORAGE_PATH + File.separator +LICENSE_METADATA_NAME;
	File licenseFile = new File(path);
	File metaFile = new File(meta);
	if (licenseFile.exists() && metaFile.exists()){
		try (BufferedReader reader = new BufferedReader(new FileReader(metaFile))){
			String proId = reader.readLine();
			license.initLicense(path, PRO_VERSION, proId);
		} catch (Exception e) {
			log.error("init license error,", e);
		}
	}
}
```

在 init 方法内部，执行获取文件流，并使用 `updateLicense` 方法解析

``` Java
public boolean initLicense(String path, String version, String proId) throws LicenseErrorException {
	File file = new File(path);
	if (!file.exists()) {
		log.error("file not exist: " + path);
		throw new LicenseErrorException(ErrorCodeConst.FILE_NOT_EXIST);
	}
	try {
		//加载文件
		byte[] fileByte = Base64Utils.fileToByte(path);
		return updateLicense(fileByte, version, proId);
	} catch (IOException e) {
		log.error("初始化license出错", e);
		throw new LicenseErrorException(ErrorCodeConst.LICENSE_READ_FAILED);
	} catch (Exception e) {
		log.error("初始化license出错", e);
	}
	return false;
}
```

在 updateLicense 方法内部，可以做一些签名校验的动作，当通过后，就需要获取其中的关键字段，来执行 service pod 的变更了。

## 使用 Java K8s Client 实现 Deployment 变更和缩放

Kubernetes 提供了强大的容器编排功能，在某些情况下，我们需要通过代码动态地管理 Kubernetes 资源，例如更新环境变量、缩放 Deployment 等。本文将使用 Kubernetes Java Client 实现这些操作。

### 1. 引入 Kubernetes Java Client

首先，我们需要在项目中引入 Kubernetes Java Client 的依赖。在 `pom.xml` 文件中添加以下依赖：

``` xml
<dependency>
	<groupId>io.kubernetes</groupId>
	<artifactId>client-java</artifactId>
	<version>18.0.0</version>
</dependency>
```

### 2. 创建 Kubernetes 客户端

在开始操作 Kubernetes 资源之前，我们需要创建一个 Kubernetes 客户端：

``` Java
import io.kubernetes.client.openapi.ApiClient;
import io.kubernetes.client.openapi.Configuration;
import io.kubernetes.client.util.Config;

public class KubernetesClientUtil {
    public static ApiClient createApiClient() throws IOException {
        ApiClient client = Config.defaultClient();
        Configuration.setDefaultApiClient(client);
        return client;
    }
}
```

### 3. 读取和更新 Deployment

通过 namespace 和 deploymentName 定位到具体的 deployment 资源，就可以像操作一个 JSON 一样操作内部的配置，进行一些键值对的修改，最后再把修改后的 deployment 提交更新。

``` Java
// 获取 DeploymentAppsV1Api 
appsApi = new AppsV1Api();  
V1Deployment deployment = appsApi.readNamespacedDeployment(deploymentName, namespace, null);

// 更新环境变量  
boolean envVarUpdated = updateEnvVar(deployment, "project_name", newProjectName);

private static boolean updateEnvVar(V1Deployment deployment, String envVarName, String newValue) {
	List<V1Container> containers = Optional.ofNullable(deployment.getSpec())
			.map(V1DeploymentSpec::getTemplate)
			.map(V1PodTemplateSpec::getSpec)
			.map(V1PodSpec::getContainers)
			.orElse(Collections.emptyList());

	boolean envVarUpdated = false;

	for (V1Container container : containers) {
		List<V1EnvVar> envVars = container.getEnv();

		if (envVars == null) {
			envVars = new ArrayList<>();
			container.setEnv(envVars);
		}

		boolean envVarFound = false;

		for (V1EnvVar envVar : envVars) {
			if (envVar.getName().equals(envVarName)) {
				envVarFound = true;
				if (!Objects.equals(envVar.getValue(), newValue)) {
					envVar.setValue(newValue);
					envVarUpdated = true;
				}
				break; // 如果找到了就不再继续遍历，假设只有一个容器的一个变量需要改
			}
		}

		if (!envVarFound) {
			V1EnvVar newEnvVar = new V1EnvVar().name(envVarName).value(newValue);
			envVars.add(newEnvVar);
			envVarUpdated = true;
		}
	}

	if (!envVarUpdated) {
		logger.warn("project_name is same as  {}", newValue);
		return false;
	}
	return true;
}

```

### 4. 更新配置，缩放副本数实现 pod 重启

为了实现动态扩缩容，我们可以使用 Kubernetes Java Client 调整 Deployment 的副本数。

{{< admonition warning "重要">}}
值的注意的是，调整副本数是异步的，并且需要一个时间来完成，如果直接调用两次而不等待将会导致指令错乱。这里我们使用了简单的轮询机制来监测 replicas 是否已经修改完成，再放行后续步骤。
{{< /admonition >}}

```Java
if (envVarUpdated) {
	// 更新 Deployment 配置
	replaceDeployment(appsApi, deploymentName, namespace, deployment);

	// 缩容 Deployment
	scaleDeployment(appsApi, deploymentName, namespace, 0);

	// 等待完成
	waitForDeploymentReplicas(appsApi, deploymentName, namespace, 0);

	// 扩容 Deployment
	scaleDeployment(appsApi, deploymentName, namespace, 1);

	// 等待完成
	waitForDeploymentReplicas(appsApi, deploymentName, namespace, 1);
} else {
	return;
}

private static void replaceDeployment(AppsV1Api appsApi, String deploymentName, String namespace, V1Deployment deployment) throws ApiException {
	appsApi.replaceNamespacedDeployment(deploymentName, namespace, deployment, null, null, null, null);
}

private static void scaleDeployment(AppsV1Api appsApi, String deploymentName, String namespace, int replicas) {
	try {
		V1Deployment deployment = appsApi.readNamespacedDeployment(deploymentName, namespace, null);
		V1DeploymentSpec spec = deployment.getSpec();
		if (spec != null) {
			spec.setReplicas(replicas);
			appsApi.replaceNamespacedDeployment(deploymentName, namespace, deployment, null, null, null, null);
		}
	} catch (ApiException e) {
		logger.error("Exception when scaling deployment: ", e);
	}
}

private static void waitForDeploymentReplicas(AppsV1Api api, String deploymentName, String namespace, int replicas) throws ApiException {
	while (true) {
		V1Deployment deployment = api.readNamespacedDeployment(deploymentName, namespace, null);
		V1DeploymentStatus status = deployment.getStatus();
		// 判断是否完成扩缩容，对于0需要特判
		if ((replicas == 0 && status.getReplicas() == null) || (status.getReplicas() != null && status.getReplicas() == replicas)) {
			break;
		}

		try {
			Thread.sleep(2000);  // 等待 2 秒钟保证完成扩缩容
		} catch (InterruptedException e) {
			Thread.currentThread().interrupt();
			throw new ApiException("Interrupted while waiting for deployment to scale");
		}
	}
}
```

至此就完成了整个 service pod 的配置动态修改和重启过程，在重启过程中会有 sh 脚本继续执行类似的监控行为，并起对应的 jar 包。由于修改 deployment 的配置后会生成新的 YAML 配置文件，所以后续的行为也算有个监控源。

# 结语

通过以上步骤，我们成功实现了一个基于 DevOps 理念的自动化部署系统，从监控源配置文件到自动化变更下游服务的流程。这简化了 K8s 的部署过程，提高了工作效率和可靠性。虽然在实现过程中遇到了一些挑战，但通过合理的设计和工具选择，我们达到了预期的目标。

本人也通过这次小需求，初窥 DevOps 的门径，希望未来能继续深入研究相关实践，了解并解决更复杂的需求和场景💪～