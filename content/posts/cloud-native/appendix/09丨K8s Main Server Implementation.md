+++
title = "K8s Main Server Implementation"
date = 2026-05-08
draft = false

categories = [
    "cloud-native"
]

tags = [
    "Go",
    "云原生"
]

series = [
    "k8s-api-server"
]

series_order = 12

+++

# 09丨K8s Main Server Implementation

通过之前的学习，已经学习过了如下知识：

1. 注册表的填充
2. Server 的 Options To Configs
3. API 的注入

而想要完整走完 K8s 主 Server 的实现，我们只需要再学习在 API 注入之前，主 Server 如何构造出 API 的输入参数的。

## 注入 API

主 Server 的 Kubernetes API 分两批注入底座 Generic Server：

* 注入核心 API，调用 Generic Server 实例的 `InstallLegacyAPIGroup()`方法完成。

  ```go
  // 代码: pkg\controlplane\apiserver\apis.go#L140
  if err := s.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); err != nil {
  ```

* 注入其它内置 API，调用 Generic Server 实例的 `InstallAPIGroups()`方法完成。

  ```go
  // 代码: pkg\controlplane\apiserver\apis.go#L149
  if err := s.GenericAPIServer.InstallAPIGroups(nonLegacy...); err != nil {
  ```

注入接口的核心输入参数是 `APIGroupInfo` 结构体的实例或数组，每个 API Group 有一个该类型实例。API 的注入过程在讲解 Generic Server 时介绍过，主 Server 如何**构造出该输入参数**是接下来的要点。

1. `New` 函数中，会先构建出 `restStorageProviders`，然后将构建出的 `restStorageProviders` 传给 `InstallAPIs` 函数，用来注入 API。

   ```go
   // 代码: pkg\controlplane\instance.go#L316-L384
   func (c CompletedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
   	...
   	restStorageProviders, err := c.StorageProviders(client)
   	if err != nil {
   		return nil, err
   	}
   	if err := s.ControlPlane.InstallAPIs(restStorageProviders...); err != nil {
   		return nil, err
   	}
   }
   ```

   先看是如何构建的 `restStorageProviders`：

   ```go
   func (c CompletedConfig) StorageProviders(client *kubernetes.Clientset) ([]controlplaneapiserver.RESTStorageProvider, error) {
       // 1. 首先创建 Core API (Legacy) Provider【后续会讲解】
   	legacyRESTStorageProvider, err := corerest.New(...)
   	if err != nil {
   		return nil, err
   	}
       // 2. 返回所有 Provider 的列表
   	providers := []controlplaneapiserver.RESTStorageProvider{
   		legacyRESTStorageProvider,
           // 3. 添加非核心 API
   		apiserverinternalrest.StorageProvider{},
   		...
       }
   	return providers, nil
   }
   ```

   得到 `restStorageProviders` 后，再将其交给 `InstallAPIs` 进行API 的注入。

2. `InstallAPIs` 代码如下：

   ```go
   // InstallAPIs will install the APIs for the restStorageProviders if they are enabled.
   func (s *Server) InstallAPIs(restStorageProviders ...RESTStorageProvider) error {
       nonLegacy := []*genericapiserver.APIGroupInfo{} // 要点①
   	...
       for _, restStorageBuilder := range restStorageProviders {
   		groupName := restStorageBuilder.GroupName() // 要点②
   		apiGroupInfo, err := restStorageBuilder.NewRESTStorage(s.APIResourceConfigSource, s.RESTOptionsGetter) // 要点③
   		...
           if len(groupName) == 0 { // 要点④
   			// the legacy group for core APIs is special that it is installed into /api via this special install method.
   			if err := s.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); err != nil {
   				return fmt.Errorf("error in registering legacy API: %w", err)
   			}
   		} else {
   			// everything else goes to /apis
   			nonLegacy = append(nonLegacy, &apiGroupInfo) // 要点⑤
   		}
   	}
   
   	if err := s.GenericAPIServer.InstallAPIGroups(nonLegacy...); err != nil { // 要点⑥
   		return fmt.Errorf("error in registering group versions: %w", err) 
   	}
   	return nil
           
   ```

   1. 对于 core API 组，通过要点②处的 `groupName := restStorageBuilder.GroupName()` 得到该 `groupName = ""`，然后调用要点③处的 `NewRESTStorage` 函数得到 `apiGroupInfo`，通过要点④处的判断后交给要点⑤处的 `nonLegacy = append(nonLegacy, &apiGroupInfo)` 进行注册。
   2. 对于其他 API 组，先定义出要点①的 `nonLegacy`，然后通过要点③处的 `NewRESTStorage` 函数得到 `apiGroupInfo`，由于无法通过要点④处的判断，会加入到 `nonLegacy` 中，等待所有 `restStorageProviders` 都被访问过后，交给要点⑥进行集中注册。

   注意，`NewRESTStorage` 函数是每个实现 RESTStorageProvider 接口的结构体自己需要实现的方法，该函数签名如下：

   ```go
   func NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, error)
   ```

   输入参数：

   * `apiResourceConfigSource` (APIResourceConfigSource 接口)
     * 作用：判断哪些资源被启用/禁用
     * 方法：
       * `ResourceEnabled(resource)` - 检查某个资源是否启用
       * `AnyResourceForGroupEnabled(group)` - 检查某个组是否有任何资源启用
   * `restOptionsGetter` (RESTOptionsGetter 接口)
     * 作用：提供存储配置（如何连接 etcd）
     * 方法：`GetRESTOptions(resource, example)` - 获取资源的存储选项

   输出结果：

   * `APIGroupInfo` 结构体，包含：

     * `PrioritizedVersions` - API 组的版本优先级列表

     * `VersionedResourcesStorageMap` - 核心！资源到 Storage 的映射，是否很熟悉？这个正是注入API 中使用的。

       * 结构：`map[version]map[resource]Storage`

       * 例如：

         ```go
         {
           "v1": {
             "deployments": deploymentStorage,
             "deployments/status": deploymentStatusStorage,
             "deployments/scale": deploymentScaleStorage,
             "statefulsets": statefulSetStorage,
             "daemonsets": daemonSetStorage,
             // ...
           }
         }
         ```

     * `Scheme` - 序列化/反序列化方案

     * `NegotiatedSerializer` - 内容协商序列化器

     * `ParameterCodec` - 查询参数编解码器

## NewRESTStorage

以 apps 为例，来看转换的过程：

```go
// 代码: pkg\registry\apps\rest\storage_apps.go#L38-L50
// NewRESTStorage returns APIGroupInfo object.
func (p StorageProvider) NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, error) {
    // 要点①
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apps.GroupName, legacyscheme.Scheme, legacyscheme.ParameterCodec, legacyscheme.Codecs)
    
    // 要点②
	if storageMap, err := p.v1Storage(apiResourceConfigSource, restOptionsGetter); err != nil {
		return genericapiserver.APIGroupInfo{}, err
	} else if len(storageMap) > 0 {
        // 要点③
        apiGroupInfo.VersionedResourcesStorageMap[appsapiv1.SchemeGroupVersion.Version] = storageMap
	}

	return apiGroupInfo, nil
}
```

1. 创建默认的 APIGroupInfo

2. 调用 v1Storage 来填充 storageMap.

3. 将填充好的 storageMap，写入 VersionedResourcesStorageMap 中.

```go
// 代码: pkg\registry\apps\rest\storage_apps.go#L52-L108
func (p StorageProvider) v1Storage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (map[string]rest.Storage, error) {
	storage := map[string]rest.Storage{}

	// deployments
	if resource := "deployments"; apiResourceConfigSource.ResourceEnabled(appsapiv1.SchemeGroupVersion.WithResource(resource)) { // 要点①
		deploymentStorage, err := deploymentstore.NewStorage(restOptionsGetter) // 要点①
		if err != nil {
			return storage, err
		}
        // 要点③
		storage[resource] = deploymentStorage.Deployment
		storage[resource+"/status"] = deploymentStorage.Status
		storage[resource+"/scale"] = deploymentStorage.Scale
	}
    ...
    return storage, nil
}
```

1. 通过要点①处的 `apiResourceConfigSource.ResourceEnabled` 检查资源是否启用，如果启用则创建 Storage；
2. 在要点②处使用 restOptionsGetter 创建 Deployment 的 Storage。
3. 然后将 Storage 注册到 map 中【要点③】。

## 构建 core API RestStorageProviders

在构建的 `restStorageProviders` 时，可以发现，其他 API 组的 API 构建十分简单，只需要进行声明即可：

```go
apiserverinternalrest.StorageProvider{},
```

而 core API 组的 `restStorageProviders` 的构建还有些麻烦：

```go
// 代码: pkg\controlplane\instance.go#L387-L399
legacyRESTStorageProvider, err := corerest.New(corerest.Config{
    GenericConfig: *c.ControlPlane.NewCoreGenericConfig(),
    Proxy: corerest.ProxyConfig{
        Transport:           c.ControlPlane.Extra.ProxyTransport,
        KubeletClientConfig: c.Extra.KubeletClientConfig,
    },
    Services: corerest.ServicesConfig{
        ClusterIPRange:          c.Extra.ServiceIPRange,
        SecondaryClusterIPRange: c.Extra.SecondaryServiceIPRange,
        NodePortRange:           c.Extra.ServiceNodePortRange,
        IPRepairInterval:        c.Extra.RepairServicesInterval,
    },
})
```

**核心 API 组**需要特殊构建的原因：

* 管理集群基础设施（Service IP、NodePort 分配）

  ```go
  // 核心 API 需要管理 Service 的 ClusterIP 和 NodePort 分配
  type legacyProvider struct {
      primaryServiceClusterIPAllocator ipallocator.Interface
      serviceClusterIPAllocators       map[api.IPFamily]ipallocator.Interface
      serviceNodePortAllocator         *portallocator.PortAllocator
      
      startServiceNodePortsRepair      func(...)
      startServiceClusterIPRepair      func(...)
  }
  ```

* 需要与 Kubelet 交互（Pod logs、exec、port-forward）

  ```go
  // 核心 API 需要代理到 Kubelet 的请求（如 logs, exec, port-forward）
  Proxy: corerest.ProxyConfig{
      Transport:           c.ControlPlane.Extra.ProxyTransport,
      KubeletClientConfig: c.Extra.KubeletClientConfig,
  }
  ```

* 需要修复控制器保证分配一致性

* 是集群的基础，需要更多的初始化和状态管理

  * Pod、Service、Node、Endpoint 等是集群的基础
  * 需要与底层基础设施（Kubelet、网络）交互
  * 需要特殊的分配和修复逻辑

**其他 API 组**简单的原因：

* 只是应用层资源（Deployment、Job 等）
* 不需要特殊的分配器或控制器
* 无状态，只需要标准的 etcd 存储

### 1. 调用 `corerest.New()` 创建 Provider

```go
// 代码: pkg\controlplane\instance.go#L387-L399
legacyRESTStorageProvider, err := corerest.New(corerest.Config{
    // 1. 通用配置
    GenericConfig: *c.ControlPlane.NewCoreGenericConfig(),
    
    // 2. Proxy 配置（用于 Pod logs, exec, port-forward）
    Proxy: corerest.ProxyConfig{
        Transport:           c.ControlPlane.Extra.ProxyTransport,
        KubeletClientConfig: c.Extra.KubeletClientConfig,
    },
    
    // 3. Service 配置（IP 和 Port 分配）
    Services: corerest.ServicesConfig{
        ClusterIPRange:          c.Extra.ServiceIPRange,          // 10.96.0.0/12
        SecondaryClusterIPRange: c.Extra.SecondaryServiceIPRange, // IPv6 范围
        NodePortRange:           c.Extra.ServiceNodePortRange,    // 30000-32767
        IPRepairInterval:        c.Extra.RepairServicesInterval,  // 修复间隔
    },
})
```

### 2. `corerest.New()` 内部实现

```go
// 代码: pkg\registry\core\rest\storage_core.go#L106-L149
func New(c Config) (*legacyProvider, error) {
    // 2.1 创建 Service IP 和 Port 分配器
    rangeRegistries, serviceClusterIPAllocator, serviceIPAllocators, 
        serviceNodePortAllocator, err := c.newServiceIPAllocators()
    
    // 2.2 创建 legacyProvider 实例，保存分配器
    p := &legacyProvider{
        Config: c,
        primaryServiceClusterIPAllocator: serviceClusterIPAllocator,  // ClusterIP 分配器
        serviceClusterIPAllocators:       serviceIPAllocators,        // 多 IP 族分配器
        serviceNodePortAllocator:         serviceNodePortAllocator,   // NodePort 分配器
    }
    
    // 2.3 创建 Kubernetes 客户端
    client, err := kubernetes.NewForConfig(c.LoopbackClientConfig)
    
    // 2.4 创建 NodePort 修复控制器
    // 作用：定期检查并修复 NodePort 分配冲突
    p.startServiceNodePortsRepair = portallocatorcontroller.NewRepair(
        c.Services.IPRepairInterval,
        client.CoreV1(),
        client.EventsV1(),
        c.Services.NodePortRange,
        rangeRegistries.nodePort,
    ).RunUntil
    
    // 2.5 创建 ClusterIP 修复控制器
    // 作用：定期检查并修复 ClusterIP 分配冲突
    p.startServiceClusterIPRepair = serviceipallocatorcontroller.NewRepair(
        c.Services.IPRepairInterval,
        client.CoreV1(),
        client.EventsV1(),
        &c.Services.ClusterIPRange,
        rangeRegistries.clusterIP,
        &c.Services.SecondaryClusterIPRange,
        rangeRegistries.secondaryClusterIP,
    ).RunUntil
    
    return p, nil
}
```

### 3. `legacyProvider.NewRESTStorage()` 创建资源 Storage

当后续调用 `NewRESTStorage()` 时，会调用 `pkg\registry\core\rest\storage_core.go#L151` 处的函数：

```go
// 代码: pkg\registry\core\rest\storage_core.go#L151-L323
func (p *legacyProvider) NewRESTStorage(
    apiResourceConfigSource serverstorage.APIResourceConfigSource,
    restOptionsGetter generic.RESTOptionsGetter,
) (genericapiserver.APIGroupInfo, error) {
    
    // 3.1 先调用 GenericConfig.NewRESTStorage() 创建基础资源
    // (Event, ResourceQuota, Secret, ConfigMap, Namespace 等)
    apiGroupInfo, err := p.GenericConfig.NewRESTStorage(apiResourceConfigSource, restOptionsGetter)
    
    // 3.2 创建需要特殊配置的资源 Storage
    
    // Pod Storage - 需要 Kubelet 连接信息和代理
    podStorage, err := podstore.NewStorage(
        restOptionsGetter,
        nodeStorage.KubeletConnectionInfo,  // Kubelet 连接信息
        p.Proxy.Transport,                  // HTTP 代理传输
        podDisruptionClient,
    )
    
    // Service Storage - 需要 IP 和 Port 分配器
    serviceRESTStorage, serviceStatusStorage, serviceRESTProxy, err := servicestore.NewREST(
        restOptionsGetter,
        p.primaryServiceClusterIPAllocator.IPFamily(),
        p.serviceClusterIPAllocators,       // ClusterIP 分配器
        p.serviceNodePortAllocator,         // NodePort 分配器
        endpointsStorage,
        podStorage.Pod,
        p.Proxy.Transport,
    )
    
    // Node Storage - 需要 Kubelet 客户端配置
    nodeStorage, err := nodestore.NewStorage(
        restOptionsGetter,
        p.Proxy.KubeletClientConfig,        // Kubelet 配置
        p.Proxy.Transport,                  // HTTP 传输
    )
    
    // 3.3 将所有资源注册到 storage map
    storage := apiGroupInfo.VersionedResourcesStorageMap["v1"]
    
    storage["pods"] = podStorage.Pod
    storage["pods/status"] = podStorage.Status
    storage["pods/log"] = podStorage.Log           // 需要 Kubelet 代理
    storage["pods/exec"] = podStorage.Exec         // 需要 Kubelet 代理
    storage["pods/portforward"] = podStorage.PortForward  // 需要 Kubelet 代理
    
    storage["services"] = serviceRESTStorage       // 需要 IP 分配器
    storage["services/status"] = serviceStatusStorage
    storage["services/proxy"] = serviceRESTProxy
    
    storage["nodes"] = nodeStorage.Node
    storage["nodes/status"] = nodeStorage.Status
    storage["nodes/proxy"] = nodeStorage.Proxy     // 需要 Kubelet 代理
    
    // ... 其他资源
    
    return apiGroupInfo, nil
}
```

至此，Kubernetes Main Server 的构建流程就梳理完毕了。
