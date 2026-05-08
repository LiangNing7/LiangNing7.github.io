+++
title = "K8s Aggregation Server Controller"
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

series_order = 19

+++

# K8s Aggregation Server Controller 

聚合器的控制器不涉及由控制器管理器负责运行的内置控制器，它们全部同聚合器一起跑在核心 API Server 程序中。在前几节中谈到不少控制器，各自起着非常重要的作用。

## 自动注册控制器与 CRD 注册控制器

因为 APIService 实例和 API 组版本间有一一对应的关系，所以每当有新 API 组的引入都需要为其每个版本建立 APIService 实例。不同类型的 API 建立 APIService 实例的方式不同：

1. 内置 API 的引入需要编码，重启 API Server 时它们的 APIService 实例会被创建并加载。
2. 通过聚合 Server 引入新的 API 组，当聚合 Server 完成部署后，需要管理员手工为其创建 APIService 实例，这部分也不用程序特殊处理。
3. 通过定义 CRD 来引入的客制化 API 则需要代码创建 APIService 实例。这个问题是由自动注册控制器和 CRD 注册控制器联手解决。它们的协作过程如图所示。

![image-20251215155729319](http://images.liangning7.cn/typora/202512151557386.png)

1. CRD 注册控制器会用一个 CRD Informer 关注 CRD 的增删改事件，把目标 CRD 实例放入工作队列。
2. `handleVersionUpdate()` 方法是 CRD 注册控制器的控制循环主逻辑，工作队列中有 CRD 实例时它会被调用。为 CRD 实例定义的客制化 API 制作 APIService 实例，并放入自动注册控制器的工作队列。
3. 自动注册控制器的控制循环主逻辑是方法 `checkAPIService()`，当工作队列中有 APIService 实例时，它会被调用，将接收到的 APIService 实例最新信息持久化到 ETCD：该创建就创建，该修改就修改。这样 CRD 中定义的客制化 API 的 APIService 实例体现到数据库中。

在这一过程中，CRD 注册控制器扮演了自动注册控制器的工作队列内容生产者的角色，而自动注册控制器则负责将最新的 APIService 信息保存进数据库。数据库中 APIService 实例的变化又会触发其它关注 APIService 变化的控制器去执行操作，形成连锁反应。

这两个控制器的创建是在 `createAggregatorServer()` 方法中完成的，代码片段如下，从中 看到 CRD 注册控制器的构造方法需要一个自动注册控制器实例为入参，因为前者需要把“产品”放入后者的工作队列。同样是在这个方法中，这两个控制器的启动被制作为启动钩子函数，在 Server 启动后被执行。

```go
// 代码: pkg\controlplane\apiserver\aggregator.go#L128-L195
func CreateAggregatorServer(aggregatorConfig aggregatorapiserver.CompletedConfig, delegateAPIServer genericapiserver.DelegationTarget, crds apiextensionsinformers.CustomResourceDefinitionInformer, crdAPIEnabled bool, apiVersionPriorities map[schema.GroupVersion]APIServicePriority) (*aggregatorapiserver.APIAggregator, error) {
	...
	autoRegistrationController := autoregister.NewAutoRegisterController(aggregatorServer.APIRegistrationInformers.Apiregistration().V1().APIServices(), apiRegistrationClient)
	apiServices := apiServicesToRegister(delegateAPIServer, autoRegistrationController, apiVersionPriorities)

	type controller interface {
		Run(workers int, stopCh <-chan struct{})
		WaitForInitialSync()
	}
	var crdRegistrationController controller
	if crdAPIEnabled {
		crdRegistrationController = crdregistration.NewCRDRegistrationController(
			crds,
			autoRegistrationController)
	}
    ...
}
```

这两个控制器分别基于结构体 autoRegisterController 和 crdRegistrationController。

## APIService 注册控制器

考虑下如下三个问题：

1. 在制作 OpenAPI 端点响应器时该如何监控 APIService 实例的变动；
2. 制作端点/apis 的响应器时该如何监控 APIService 实例的变动；
3. 为每个 API 组版本的端点制作响应器（proxyHandler）时该如何监控 APIService 实例的变动。

### 1. 控制器的创建

`APIService` 注册控制器的创建发生在 `NewWithDelegate()` 方法中，工厂函数 `NewAPIServiceRegistratioinController()` 被调用创建了它，它的代码如下：

```go
// 代码: staging\src\k8s.io\kube-aggregator\pkg\apiserver\apiservice_controller.go#L60-L80
// NewAPIServiceRegistrationController returns a new APIServiceRegistrationController.
func NewAPIServiceRegistrationController(apiServiceInformer informers.APIServiceInformer, apiHandlerManager APIHandlerManager) *APIServiceRegistrationController {
	c := &APIServiceRegistrationController{
		apiHandlerManager: apiHandlerManager,
		apiServiceLister:  apiServiceInformer.Lister(),
		apiServiceSynced:  apiServiceInformer.Informer().HasSynced,
		queue: workqueue.NewTypedRateLimitingQueueWithConfig(
			workqueue.DefaultTypedControllerRateLimiter[string](),
			workqueue.TypedRateLimitingQueueConfig[string]{Name: "APIServiceRegistrationController"},
		),
	}

	apiServiceInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    c.addAPIService,
		UpdateFunc: c.updateAPIService,
		DeleteFunc: c.deleteAPIService,
	})

	c.syncFn = c.sync

	return c
}
```

`NewAPIServiceRegistratioinController()` 函数接收两个入参：

1. 名是 APIServiceInformer 的 Informer，用来监控 ETCD 中 APIService 实例的变化并发出事件；

2. 参数类型是接口 APIHandlerManager，实参用的就是构建中的聚合器实例，聚合器是实 现了该接口的。APIHandlerManager 定义如下：

   ```go
   // 代码: staging\src\k8s.io\kube-aggregator\pkg\apiserver\apiservice_controller.go#L39-L42
   // APIHandlerManager defines the behaviour that an API handler should have.
   type APIHandlerManager interface {
   	AddAPIService(apiService *v1.APIService) error
   	RemoveAPIService(apiServiceName string)
   }
   ```

### 2. 控制器的内部结构

APIService 注册控制器遵从了标准的控制器模式，基于前面对各种控制器的介绍并不难 以理解其内部构造。它的实现基于结构体 `APIServiceRegistrationController`，其字段如图

![image-20251215160926850](http://images.liangning7.cn/typora/202512151609928.png)

理解一个控制器的关键是理解其控制循环的主逻辑，这通常是控制器基座结构体上的一个方法，而且这个方法名字一般都以 sync 为前缀。对于 APIService 注册控制器来说，这个 方法就是 `sync()`，它的逻辑非常简洁和清晰：

```go
// 代码: staging\src\k8s.io\kube-aggregator\pkg\apiserver\apiservice_controller.go#L82-L93
func (c *APIServiceRegistrationController) sync(key string) error {
	apiService, err := c.apiServiceLister.Get(key)
	if apierrors.IsNotFound(err) {
		c.apiHandlerManager.RemoveAPIService(key)
		return nil
	}
	if err != nil {
		return err
	}

	return c.apiHandlerManager.AddAPIService(apiService)
}
```

`sync()` 方法通过查询 ETCD 的方式确定目标 APIService 实例是否被删除，是的话调用 `apiHandlerManager`（也就是聚合器实例）的 `RemoveAPIService()` 方法删除它；否则属于新增或删除的情况，调用 AddAPIService()方法添加。就是这么简单。那么聚合器的 `RemoveAPIService()` 和 `AddAPIService()` 方法都做了什么事情就是关键了。

### 聚合器的 AddAPIService()

当新增和修改 APIService 实例时，聚合器的这个方法被调用，它的签名如下：

```go
func (s *APIAggregator) AddAPIService(apiService *v1.APIService) error
```

该方法内部处理如下几件事情：

1. 为该 APIService 实例所决定的端点制作（或修改）端点响应器

   一个 APIService 实例决定了一组以这个字符串开头的端点：`/apis/<组名>/<版本>`，客户端通过向 `/apis/<组名>/<版本>` 发 Get、Post 等请求来完成操作。当目标 API 由聚合 Server 提供时，需要反向代理做请求的转发，聚合器通过创造一个 proxyHandler 结构体实例来作为这组端点的响应器，它实现了 `http.Handler` 接口并且内置了反向代理服务用于请求转发，端点响应器代码如下：

   ```go
   // 代码: staging\src\k8s.io\kube-aggregator\pkg\apiserver\apiserver.go#L536-L544
   // register the proxy handler
   proxyHandler := &proxyHandler{
       localDelegate:              s.delegateHandler,
       proxyCurrentCertKeyContent: s.proxyCurrentCertKeyContent,
       proxyTransportDial:         s.proxyTransportDial,
       serviceResolver:            s.serviceResolver,
       rejectForwardingRedirects:  s.rejectForwardingRedirects,
       tracerProvider:             s.tracerProvider,
   }
   proxyHandler.updateAPIService(apiService) //要点①
   ```

   要点①处以该 APIService 实例为入参调用了 proxyHandler 的 `updateAPIService()` 方法，这确保了 proxyHandler 能够从中抽取目标聚合 Server 的地址等信息。

   这回答了 APIService 注册控制器实现了监控 APIService 实例的变更并触发 proxyHandler 的创建和初始化。

2. 通知 API 发现聚合控制器和 OpenAPI 聚合控制器

   APIService 注册控制器来替聚合控制器和 OpenAPI 聚合控制器关注 APIService 实例的变化在本控制器的 `AddAPIService()` 方法中具有如下的代码：

   ```go
   	// 代码: staging\src\k8s.io\kube-aggregator\pkg\apiserver\apiserver.go#L545-L553
   	if s.openAPIAggregationController != nil {
   		s.openAPIAggregationController.AddAPIService(proxyHandler, apiService)
   	}
   	if s.openAPIV3AggregationController != nil {
   		s.openAPIV3AggregationController.AddAPIService(proxyHandler, apiService)
   	}
   	if s.discoveryAggregationController != nil {
   		s.discoveryAggregationController.AddAPIService(apiService, proxyHandler)
   	}
   ```

   这两个控制器都有名为 `AddAPIService()` 的方法，它们会把目标 APIService 实例放入各自的工作队列，在后续控制循环中去更新它们内部信息。

3. 为该 APIService 实例所决定的 API 组设置发现端点响应器

   一个 APIService 实例决定了一个 API 组发现端点：`/apis/<组名>`，通过向这个端点发送 GET 请求会得到该组所有可用版本列表。结构体 apiGroupHandler 负责提供相应实现，它的 ServeHTTP()方法从 ETCD 读出所有 APIService 实例，把属于该组的版本信息读取出来返回。以下是制作组发现响应器以及向端点绑定的代码：

   ```go
   // 代码: staging\src\k8s.io\kube-aggregator\pkg\apiserver\apiserver.go#L571-L582
   // it's time to register the group aggregation endpoint
   groupPath := "/apis/" + apiService.Spec.Group
   groupDiscoveryHandler := &apiGroupHandler{
       codecs:    aggregatorscheme.Codecs,
       groupName: apiService.Spec.Group,
       lister:    s.lister,
       delegate:  s.delegateHandler,
   }
   // aggregation is protected
   s.GenericAPIServer.Handler.NonGoRestfulMux.Handle(groupPath, groupDiscoveryHandler)
   s.GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandle(groupPath+"/", groupDiscoveryHandler)
   s.handledGroupVersions[apiService.Spec.Group] = sets.New[string](apiService.Spec.Version)
   ```

   到现在为止，`端点/apis`、`/apis/<组名>`、` /apis/<组名>/< 版本>` 和 `/apis/<组名>/<版本>/<资源>` 都具有了响应器，发向它们的 HTTP 请求都会被处理。

4. 聚合器的 RemoveAPIService

   当删除一个 APIService 实例时，聚合器的这个方法被调用，相对于 AddAPIService 方法 它的逻辑要简单的多，只需进行一些清理工作：

   1. 从聚合器的 proxyHandlers 列表中移除该 APIServer 的 proxyHandler 实例；
   2. 移除端点/apis/<组名>/<版本> 上的响应器；
   3. 调用 OpenAPI 聚合控制器和发现聚合控制器的 `RemoveAPIService` 方法，从而从它们内部清除相应信息

## API Service 状态监测控制器

聚合 Server 的存在使得 API Server 成为分布式的结构，这种体系结构下各组件的可用性是需要特别关注的。聚合器会对每一个 APIService 实例的增删改进行监控，事件发生时去检测支撑它的 Kubernetes Service 是否处于连通可用的状态，据此更新该 APIService 实例的 Status 信息——`status.conditions.status` 和 `status.conditions.type`。

上述检测和更新是 API Service 控制器完成的。该控制器基于结构体 `AvailableConditionController`，其上有控制器模式的一般属性，例如工作队列 queue，指向控制循环核心逻辑的属性 syncFn，访问三种 API 用的 Lister 属性。控制器模式标准方法也都被赋予了该结构体，如 `runWorker()`、`processNextWorkItem()`、向工作队列添加内容的 `addXXX()`、`updateXXX()` 和 `deleteXXX()` 方法。控制器的运作机制如图所示。

![image-20251215161957894](http://images.liangning7.cn/typora/202512151619985.png)控制循环的核心逻辑是方法 `sync()`。一个 APIService 的资源定义中有 `spec.service` 属性代表了这个 API 组版本所在的 Server。如果目标是核心 API Server 则这个属性是空。对于来自聚合 Server 的 API 组版本，`spec.service` 指向一个集群内的 Service API 实例，聚合 Server 通过该实例暴露自己的连接信息。一个 Service 会通过一个同名的 Endpoints 实例来管理其所在 Pod 的地址，如果部署在多个 Pod 上则它们都会出现在 Endpoints 的信息上，一个 Service 实例如下图所示，它的 Endpoints 如下下图所示。所以当 APIService、Service 和 Endpoints 发生增删改时，需要启动对相关 APIService 实例关联的 Service 可达性进行检验，检测的方式是首先从 Service 对象提取连接信息，然后用一个反向代理服务去请求该地址上的端点 “/apis/<API 组>/<API 版本>” （如果是核心 API 则是`/api/<版本>`），如果得到的响应状态码不在区间 [200,300)内，代表服务失败；连续尝试 5 次，没有一次成功就认定该 Service 不能服务，改变 APIService 实例的状态信息。

![image-20251215162431107](http://images.liangning7.cn/typora/202512151624206.png)

![image-20251215162443482](http://images.liangning7.cn/typora/202512151624576.png)

除此之外 `sync()` 方法还有一些简单的检查，例如 Service 干脆就没有则直接返回错误。 
