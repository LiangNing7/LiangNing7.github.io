+++
title = "K8s Aggregation Server Implementation"
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

series_order = 18

+++

# K8s Aggregation Server Implementation

与扩展 Server 类似，聚合器同样处于单独的模块：`k8s.io/kube-aggregator` ，形成单独代码库。由于基于 Generic Server，所以同样具有准入控制、登录鉴权等基础功能；而且这个模块也可以被编译成为单独应用程序去运行。在设计思路上和扩展 Server 的如出一辙，这 里不再赘述。

## API Service 简介

聚合器具有由它管理和使用的 Kubernetes API：`apiregistration.k8s.io` 组内的 APIService。 

一个 APIService 实例代表一个 API Group 和 Version 的组合。在聚合器内，API Server 的每一个API Group的每个Version都会有一个APIService实例与之对应。这一点特别重要，聚合器依赖这些信息确定一个请求的响应 Server 并进行请求委派或代理转发。一个 APIService 的 spec 中具有的信息如下：

```go
// 代码: staging\src\k8s.io\kube-aggregator\pkg\apis\apiregistration\types.go#L46-L86
// APIServiceSpec contains information for locating and communicating with a server.
// Only https is supported, though you are able to disable certificate verification.
type APIServiceSpec struct {
	// Service is a reference to the service for this API server.  It must communicate
	// on port 443.
	// If the Service is nil, that means the handling for the API groupversion is handled locally on this server.
	// The call will simply delegate to the normal handler chain to be fulfilled.
	// +optional
	Service *ServiceReference
	// Group is the API group name this server hosts
	Group string
	// Version is the API version this server hosts.  For example, "v1"
	Version string

	// InsecureSkipTLSVerify disables TLS certificate verification when communicating with this server.
	// This is strongly discouraged.  You should use the CABundle instead.
	InsecureSkipTLSVerify bool
	// CABundle is a PEM encoded CA bundle which will be used to validate an API server's serving certificate.
	// If unspecified, system trust roots on the apiserver are used.
	// +listType=atomic
	// +optional
	CABundle []byte

	// GroupPriorityMinimum is the priority this group should have at least. Higher priority means that the group is preferred by clients over lower priority ones.
	// Note that other versions of this group might specify even higher GroupPriorityMinimum values such that the whole group gets a higher priority.
	// The primary sort is based on GroupPriorityMinimum, ordered highest number to lowest (20 before 10).
	// The secondary sort is based on the alphabetical comparison of the name of the object.  (v1.bar before v1.foo)
	// We'd recommend something like: *.k8s.io (except extensions) at 18000 and
	// PaaSes (OpenShift, Deis) are recommended to be in the 2000s
	GroupPriorityMinimum int32

	// VersionPriority controls the ordering of this API version inside of its group.  Must be greater than zero.
	// The primary sort is based on VersionPriority, ordered highest to lowest (20 before 10).
	// Since it's inside of a group, the number can be small, probably in the 10s.
	// In case of equal version priorities, the version string will be used to compute the order inside a group.
	// If the version string is "kube-like", it will sort above non "kube-like" version strings, which are ordered
	// lexicographically. "Kube-like" versions start with a "v", then are followed by a number (the major version),
	// then optionally the string "alpha" or "beta" and another number (the minor version). These are sorted first
	// by GA > beta > alpha (where GA is a version with no suffix such as beta or alpha), and then by comparing major
	// version, then minor version. An example sorted list of versions:
	// v10, v2, v1, v11beta2, v10beta3, v3beta1, v12alpha1, v11alpha2, foo1, foo10.
	VersionPriority int32
}
```

`APIServiceSpec` 对理解聚合器工作机制非常重要，它的主要字段含义为：

1. `Group` 和 `Version` 字段记录这个 APIService 实例为哪一个 API 组与版本所创建，这印证了每个 Group 和 Version 的组合会有一个 APIService 实例。
2. `Service` 代表目标子 Server 的地址：如果该子 Server 是一个聚合 Server，则 Service 会是引用一个 `Kubernetes Service` 实例；如果是聚合器、主 Server 或扩展 Server，则 Service 字段将会是 nil，因为这三者同处核心 API Server，相对聚合器来说为本地。
3. `GroupPriorityMinimum` 和 `VersionPriority` 字段决定了这个组和版本出现在发现信息列表的位置：各个组之间用 `GroupPriorityMinimum` 排序；同组内的各个版本用 `VersionPriority` 排序。都是倒序。
4. `CABundle` 字段，当聚合 Server 与核心 API Server 联络时，核心 API Server 需要能够验签聚合 Server 所出示的证书，且要求该证书是颁给 `<service>.<namespace>.svc` 的。验签过程需要签发聚合 Server HTTPS 证书的 CA 证书，这个 CABundle 字节数组存放该 CA 证书 base64 编码后的内容。注意：聚合 Server 也有认证核心 API Server 的要求，也就是说核心 API Server 也要向聚合 Server 出示证书并且聚合 Server 要能验签它，这实际上在二者之间建立了 `mutual-TLS` 关系。

API Service 资源示例如下：

```yaml
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1alpha1.dummy
spec:
  caBundle: <base64-encoded-serving-ca-certificate>
  group: dummyGroup
  version: v1alpha1
  groupPriorityMinimum: 1000
  versionPriority: 15
  service:
    name: dummy-server
    namespace: dummy-namespace
  status:
    …
```

## 准备 Server 运行配置

为了创建聚合器首先要得到它的初始配置信息，函数 `createAggregatorConfig()` 会完成这项工作。

```yaml
┌──────────────────────────────────────────────────────────────────────────────────┐
│  起点：Run() 函数                                                                 │
│  cmd/kube-apiserver/app/server.go:148                                            │
├──────────────────────────────────────────────────────────────────────────────────┤
│  func Run(ctx context.Context, opts options.CompletedOptions) error {            │
│      ...                                                                         │
│      config, err := NewConfig(opts)  // 👈 进入配置构建                           │
│      ...                                                                         │
│  }                                                                               │
└──────────────────────────────────────────────────────────────────────────────────┘
                                       ↓
┌──────────────────────────────────────────────────────────────────────────────────┐
│  第一步：NewConfig() 函数                                                         │
│  cmd/kube-apiserver/app/config.go:74                                             │
├──────────────────────────────────────────────────────────────────────────────────┤
│  func NewConfig(opts options.CompletedOptions) (*Config, error) {                │
│      ...                                                                         │
│      aggregator, err := controlplaneapiserver.CreateAggregatorConfig( // :101    │
│          *kubeAPIs.ControlPlane.Generic,                                         │
│          opts.CompletedOptions,                                                  │
│          kubeAPIs.ControlPlane.VersionedInformers,                               │
│          serviceResolver,                                                        │
│          kubeAPIs.ControlPlane.ProxyTransport,                                   │
│          kubeAPIs.ControlPlane.Extra.PeerProxy,                                  │
│          pluginInitializer,                                                      │
│      )                                                                           │
│      c.Aggregator = aggregator                                                   │
│      ...                                                                         │
│  }                                                                               │
└──────────────────────────────────────────────────────────────────────────────────┘
                                       ↓
┌──────────────────────────────────────────────────────────────────────────────────┐
│  第二步：CreateAggregatorConfig() 函数                                            │
│  pkg/controlplane/apiserver/aggregator.go:53                                     │
├──────────────────────────────────────────────────────────────────────────────────┤
│  func CreateAggregatorConfig(                                                    │
│      kubeAPIServerConfig genericapiserver.Config,                                │
│      commandOptions options.CompletedOptions,                                    │
│      externalInformers kubeexternalinformers.SharedInformerFactory,              │
│      serviceResolver aggregatorapiserver.ServiceResolver,                        │
│      proxyTransport *http.Transport,                                             │
│      peerProxy utilpeerproxy.Interface,                                          │
│      pluginInitializers []admission.PluginInitializer,                           │
│  ) (*aggregatorapiserver.Config, error) {                                        │
│      ...                                                                         │
│  }                                                                               │
└──────────────────────────────────────────────────────────────────────────────────┘
```

该函数不必从零开始，只需在主 Server 的底座 Generic Server 运行配置信息基础上进行修改便可，下面来看看 `CreateAggregatorConfig` 的具体逻辑：

```go
// 代码: pkg/controlplane/apiserver/aggregator.go#L52-L126
func CreateAggregatorConfig(
	kubeAPIServerConfig genericapiserver.Config,
	commandOptions options.CompletedOptions,
	externalInformers kubeexternalinformers.SharedInformerFactory,
	serviceResolver aggregatorapiserver.ServiceResolver,
	proxyTransport *http.Transport,
	peerProxy utilpeerproxy.Interface,
	pluginInitializers []admission.PluginInitializer,
) (*aggregatorapiserver.Config, error) {
	// make a shallow copy to let us twiddle a few things
	// most of the config actually remains the same.  We only need to mess with a couple items related to the particulars of the aggregator
	genericConfig := kubeAPIServerConfig
	genericConfig.PostStartHooks = map[string]genericapiserver.PostStartHookConfigEntry{}
	genericConfig.RESTOptionsGetter = nil
	// prevent generic API server from installing the OpenAPI handler. Aggregator server
	// has its own customized OpenAPI handler.
	genericConfig.SkipOpenAPIInstallation = true

	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.StorageVersionAPI) &&
		utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerIdentity) {
		// Add StorageVersionPrecondition handler to aggregator-apiserver.
		// The handler will block write requests to built-in resources until the
		// target resources' storage versions are up-to-date.
		genericConfig.BuildHandlerChainFunc = genericapiserver.BuildHandlerChainWithStorageVersionPrecondition
	}

	if peerProxy != nil {
		originalHandlerChainBuilder := genericConfig.BuildHandlerChainFunc
		genericConfig.BuildHandlerChainFunc = func(apiHandler http.Handler, c *genericapiserver.Config) http.Handler {
			// Add peer proxy handler to aggregator-apiserver.
			// wrap the peer proxy handler first.
			apiHandler = peerProxy.WrapHandler(apiHandler)
			return originalHandlerChainBuilder(apiHandler, c)
		}
	}

	// copy the etcd options so we don't mutate originals.
	// we assume that the etcd options have been completed already.  avoid messing with anything outside
	// of changes to StorageConfig as that may lead to unexpected behavior when the options are applied.
	etcdOptions := *commandOptions.Etcd
	etcdOptions.StorageConfig.Codec = aggregatorscheme.Codecs.LegacyCodec(v1.SchemeGroupVersion, v1beta1.SchemeGroupVersion)
	etcdOptions.StorageConfig.EncodeVersioner = runtime.NewMultiGroupVersioner(v1.SchemeGroupVersion, schema.GroupKind{Group: v1beta1.GroupName})
	etcdOptions.SkipHealthEndpoints = true // avoid double wiring of health checks
	if err := etcdOptions.ApplyTo(&genericConfig); err != nil {
		return nil, err
	}

	// override MergedResourceConfig with aggregator defaults and registry
	if err := commandOptions.APIEnablement.ApplyTo(
		&genericConfig,
		aggregatorapiserver.DefaultAPIResourceConfigSource(),
		aggregatorscheme.Scheme); err != nil {
		return nil, err
	}

	aggregatorConfig := &aggregatorapiserver.Config{
		GenericConfig: &genericapiserver.RecommendedConfig{
			Config:                genericConfig,
			SharedInformerFactory: externalInformers,
		},
		ExtraConfig: aggregatorapiserver.ExtraConfig{
			ProxyClientCertFile:       commandOptions.ProxyClientCertFile,
			ProxyClientKeyFile:        commandOptions.ProxyClientKeyFile,
			PeerAdvertiseAddress:      commandOptions.PeerAdvertiseAddress,
			ServiceResolver:           serviceResolver,
			ProxyTransport:            proxyTransport,
			RejectForwardingRedirects: commandOptions.AggregatorRejectForwardingRedirects,
		},
	}

	// we need to clear the poststarthooks so we don't add them multiple times to all the servers (that fails)
	aggregatorConfig.GenericConfig.PostStartHooks = map[string]genericapiserver.PostStartHookConfigEntry{}

	return aggregatorConfig, nil
}
```

1. 浅拷贝与清理钩子

   ```go
   genericConfig := kubeAPIServerConfig
   genericConfig.PostStartHooks = map[string]genericapiserver.PostStartHookConfigEntry{}
   genericConfig.RESTOptionsGetter = nil
   ```

   * `genericConfig := kubeAPIServerConfig`：对主 Server 配置做浅拷贝，聚合器大部分配置与主 Server 相同，只需微调少量字段；
   * `PostStartHooks = map[...]{}`: 清空启动后钩子。主 Server 会作为 delegation 传给聚合器，聚合器构造 GenericServer 时会自动从 delegation 抽取钩子，不清空会导致重复注册；
   * `RESTOptionsGetter = nil`：清空 REST 选项获取器，聚合器会通过后续的 `etcdOptions.ApplyTo()` 重新设置自己的。

2. 跳过 OpenAPI 安装

   ```go
   genericConfig.SkipOpenAPIInstallation = true
   ```

   * `GenericServer` 的 `PrepareRun()` 默认会为 `/openapi/v2` 和 `/openapi/v3` 安装处理器；
   * 聚合器需要聚合多个后端 Server 的 OpenAPI spec，有自己定制的处理器；
   * 设为 true 阻止 GenericServer 安装默认处理器，避免冲突。

3. StorageVersion 前置条件处理器

   ```go
   if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.StorageVersionAPI) &&
       utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerIdentity) {
       genericConfig.BuildHandlerChainFunc = genericapiserver.BuildHandlerChainWithStorageVersionPrecondition
   }
   ```

   * 当 StorageVersionAPI 和 APIServerIdentity 两个特性门控都启用时；
   * 设置自定义的 handler chain 构建函数；
   * 该处理器会阻塞对内置资源的写请求，直到目标资源的存储版本是最新的；
   * 这是为了保证多 API Server 实例间的存储版本一致性。

4. Peer Proxy 处理器

   ```go
   if peerProxy != nil {
       originalHandlerChainBuilder := genericConfig.BuildHandlerChainFunc
       genericConfig.BuildHandlerChainFunc = func(apiHandler http.Handler, c *genericapiserver.Config) http.Handler {
           apiHandler = peerProxy.WrapHandler(apiHandler)
           return originalHandlerChainBuilder(apiHandler, c)
       }
   }
   ```

   * 如果启用了 peer proxy（用于多 API Server 实例间的请求路由）；
   * 保存原有的 handler chain 构建函数；
   * 创建新的构建函数：先用 peerProxy 包装 handler，再调用原构建函数；
   * 这样 peer proxy 处理器会在请求链的最外层，可以将请求路由到正确的 peer 实例。

5. ETCD 配置

   ```go
   etcdOptions := *commandOptions.Etcd
   etcdOptions.StorageConfig.Codec = aggregatorscheme.Codecs.LegacyCodec(v1.SchemeGroupVersion, v1beta1.SchemeGroupVersion)
   etcdOptions.StorageConfig.EncodeVersioner = runtime.NewMultiGroupVersioner(v1.SchemeGroupVersion, schema.GroupKind{Group: v1beta1.GroupName})
   etcdOptions.SkipHealthEndpoints = true
   if err := etcdOptions.ApplyTo(&genericConfig); err != nil {
       return nil, err
   }
   ```

   * `etcdOptions := *commandOptions.Etcd`：拷贝 ETCD 配置，避免修改原始配置；
   * `Codec`：设置 APIService 资源的编解码器，支持 v1 和 v1beta1 两个版本；
   * `EncodeVersioner`：指定存储时优先使用 v1 版本，同时支持 v1beta1 组；
   * `SkipHealthEndpoints = true`：跳过健康检查端点注册，避免与主 Server 重复；
   * `ApplyTo`：将 ETCD 配置应用到 genericConfig，设置 RESTOptionsGetter 等。

6. API 启用配置

   ```go
   if err := commandOptions.APIEnablement.ApplyTo(
       &genericConfig,
       aggregatorapiserver.DefaultAPIResourceConfigSource(),
       aggregatorscheme.Scheme); err != nil {
       return nil, err
   }
   ```

   * 根据命令行参数（如 `--runtime-config`）配置哪些 API 版本启用/禁用；
   * `DefaultAPIResourceConfigSource()`：聚合器默认的 API 资源配置；
   * `aggregatorscheme.Scheme`：聚合器的 scheme，包含 APIService 等资源的类型注册；
   * 这决定了 `apiregistration.k8s.io/v1` 和 `v1beta1` 是否可用。

7. 构造聚合器配置

   ```go
   aggregatorConfig := &aggregatorapiserver.Config{
       GenericConfig: &genericapiserver.RecommendedConfig{
           Config:                genericConfig,
           SharedInformerFactory: externalInformers,
       },
       ExtraConfig: aggregatorapiserver.ExtraConfig{
           ProxyClientCertFile:       commandOptions.ProxyClientCertFile,
           ProxyClientKeyFile:        commandOptions.ProxyClientKeyFile,
           PeerAdvertiseAddress:      commandOptions.PeerAdvertiseAddress,
           ServiceResolver:           serviceResolver,
           ProxyTransport:            proxyTransport,
           RejectForwardingRedirects: commandOptions.AggregatorRejectForwardingRedirects,
       },
   }
   ```

   * `GenericConfig`：通用配置，包含前面调整过的 genericConfig 和共享 Informer 工厂。
   * `ExtraConfig`：聚合器特有配置：
     * `ProxyClientCertFile/KeyFile`：代理到后端 Server 时使用的客户端证书/私钥；
     * `PeerAdvertiseAddress`：peer 间通信的广播地址；
     * `ServiceResolver`：解析 Service 到实际端点的解析器；
     * `ProxyTransport`：代理请求使用的 HTTP Transport；
     * `RejectForwardingRedirects`：是否拒绝转发重定向响应。

8. 再次清理钩子

   ```go
   aggregatorConfig.GenericConfig.PostStartHooks = map[string]genericapiserver.PostStartHookConfigEntry{}
   
   return aggregatorConfig, nil
   ```

   * 再次清空 `PostStartHooks`，确保不会重复添加到所有 Server；
   * 这是双重保险，因为前面已经清过一次，但 `ApplyTo` 等操作可能会添加新的钩子；
   * 返回构造好的聚合器配置。

初始配置构建完成后，只需要使用 `Completed` 即可得到运行配置，其流程图如下：

```go
┌─────────────────────────────────────────────────────────────────────────────┐
│                    cmd/kube-apiserver/app/server.go                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Run(ctx, opts options.CompletedOptions)                                    │
│       │                                                                     │
│       ▼                                                                     │
│  config, err := NewConfig(opts)  ─────────────────────────────────────────┐ │
│       │                                                                   │ │
│       │                                                                   │ │
│       ▼                                                                   │ │
│  completed, err := config.Complete()                                      │ │
│       │                                                                   │ │
│       ▼                                                                   │ │
│  server, err := CreateServerChain(completed)                              │ │
│       │                                                                   │ │
│       ▼                                                                   │ │
│  prepared.Run(ctx)                                                        │ │
│                                                                           │ │
└───────────────────────────────────────────────────────────────────────────┼─┘
                                                                            │
┌───────────────────────────────────────────────────────────────────────────┼─┐
│                    cmd/kube-apiserver/app/config.go                       │ │
├───────────────────────────────────────────────────────────────────────────┼─┤
│                                                                           │ │
│  NewConfig(opts) ◄────────────────────────────────────────────────────────┘ │
│       │                                                                     │
│       ├──► BuildGenericConfig()                                             │
│       │         └──► genericConfig, versionedInformers, storageFactory      │
│       │                                                                     │
│       ├──► CreateKubeAPIServerConfig()                                      │
│       │         └──► c.KubeAPIs = *controlplane.Config                      │
│       │                                                                     │
│       ├──► CreateAPIExtensionsConfig()                                      │
│       │         └──► c.ApiExtensions = *apiextensionsapiserver.Config       │
│       │                                                                     │
│       └──► CreateAggregatorConfig()                                         │
│                 └──► c.Aggregator = *aggregatorapiserver.Config             │
│                                                                             │
│  返回: &Config{KubeAPIs, ApiExtensions, Aggregator}                         │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  (c *Config) Complete() CompletedConfig                                     │
│       │                                                                     │
│       ├──► c.Aggregator.Complete()    ──────────────────────────────────┐   │
│       │         └──► aggregatorapiserver.CompletedConfig                │   │
│       │                                                                 │   │
│       ├──► c.KubeAPIs.Complete()                                        │   │
│       │         └──► controlplane.CompletedConfig                       │   │
│       │                                                                 │   │
│       └──► c.ApiExtensions.Complete()                                   │   │
│                 └──► apiextensionsapiserver.CompletedConfig             │   │
│                                                                         │   │
│  返回: CompletedConfig{Aggregator, KubeAPIs, ApiExtensions}             │   │
│                                                                         │   │
└─────────────────────────────────────────────────────────────────────────┼───┘
                                                                          │
┌─────────────────────────────────────────────────────────────────────────┼───┐
│        staging/src/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go    │   │
├─────────────────────────────────────────────────────────────────────────┼───┤
│                                                                         │   │
│  (cfg *Config) Complete() CompletedConfig  ◄────────────────────────────┘   │
│       │                                                                     │
│       ├──► cfg.GenericConfig.Complete()                                     │
│       │         └──► genericapiserver.CompletedConfig                       │
│       │                                                                     │
│       └──► c.GenericConfig.EnableDiscovery = false  // 要点                  │
│                                                                             │
│  返回: CompletedConfig{GenericConfig, ExtraConfig}                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

可以看到这里设置了 `c.GenericConfig.EnableDiscovery = false`，这是因为，kube-aggregator 需要自己实现 Discovery 机制，而不是使用 GenericAPIServer 的默认实现。

```go
// 代码: staging/src/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go
s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", apisHandlerWithAggregationSupport)
s.GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandle("/apis/", apisHandler)
```

如果启用默认 Discovery，会和 aggregator 自己的实现冲突。

## 创建聚合器

聚合器的创建是在 `CreateServerChain()` 函数内触发的。聚合器是核心 API Server 内的 Server 链头，所以它最后一个被构建出来，它在链上的下级 Server 是主 Server。聚合器构建完成后，整个 `CreateServerChain()` 函数也随之结束，聚合器实例被作为最终结果返回。

```go
┌──────────────────────────────────────────────────────────────────────────────────┐
│  起点：Run() 函数                                                                 │
│  cmd/kube-apiserver/app/server.go:148                                            │
├──────────────────────────────────────────────────────────────────────────────────┤
│  func Run(ctx context.Context, opts options.CompletedOptions) error {            │
│      ...                                                                         │
│      server, err := CreateServerChain(completed)  // 👈 创建服务器链 :161          │
│      ...                                                                         │
│  }                                                                               │
└──────────────────────────────────────────────────────────────────────────────────┘
                                         ↓
┌──────────────────────────────────────────────────────────────────────────────────┐
│  第一步：CreateServerChain() 函数                                                 │
│  cmd/kube-apiserver/app/server.go:171                                            │
├──────────────────────────────────────────────────────────────────────────────────┤
│  func CreateServerChain(config CompletedConfig) (                                │
│      *aggregatorapiserver.APIAggregator, error,                                  │
│  ) {                                                                             │
│      ...                                                                         │
│      aggregatorServer, err := controlplaneapiserver.CreateAggregatorServer(      │
│          config.Aggregator,                           // :183                    │
│          kubeAPIServer.ControlPlane.GenericAPIServer,                            │
│          apiExtensionsServer.Informers.Apiextensions().V1().CustomResourceDef... │
│          crdAPIEnabled,                                                          │
│          apiVersionPriorities,                                                   │
│      )                                                                           │
│      ...                                                                         │
│  }                                                                               │
└──────────────────────────────────────────────────────────────────────────────────┘
                                         ↓
┌──────────────────────────────────────────────────────────────────────────────────┐
│  终点：CreateAggregatorServer() 函数                                              │
│  pkg/controlplane/apiserver/aggregator.go:128                                    │
├──────────────────────────────────────────────────────────────────────────────────┤
│  func CreateAggregatorServer(                                                    │
│      aggregatorConfig aggregatorapiserver.CompletedConfig,                       │
│      delegateAPIServer genericapiserver.DelegationTarget,                        │
│      crds apiextensionsinformers.CustomResourceDefinitionInformer,               │
│      crdAPIEnabled bool,                                                         │
│      apiVersionPriorities map[schema.GroupVersion]APIServicePriority,            │
│  ) (*aggregatorapiserver.APIAggregator, error) {                                 │
│      ...                                                                         │
│  }                                                                               │
└──────────────────────────────────────────────────────────────────────────────────┘
```

`CreateAggregatorServer()` 源码如下：

```go
// 代码: pkg\controlplane\apiserver\aggregator.go#L128-L194
func CreateAggregatorServer(aggregatorConfig aggregatorapiserver.CompletedConfig, delegateAPIServer genericapiserver.DelegationTarget, crds apiextensionsinformers.CustomResourceDefinitionInformer, crdAPIEnabled bool, apiVersionPriorities map[schema.GroupVersion]APIServicePriority) (*aggregatorapiserver.APIAggregator, error) {
	aggregatorServer, err := aggregatorConfig.NewWithDelegate(delegateAPIServer)
	if err != nil {
		return nil, err
	}

	// create controllers for auto-registration
	apiRegistrationClient, err := apiregistrationclient.NewForConfig(aggregatorConfig.GenericConfig.LoopbackClientConfig)
	if err != nil {
		return nil, err
	}
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

	// Imbue all builtin group-priorities onto the aggregated discovery
	if aggregatorConfig.GenericConfig.AggregatedDiscoveryGroupManager != nil {
		for gv, entry := range apiVersionPriorities {
			aggregatorConfig.GenericConfig.AggregatedDiscoveryGroupManager.SetGroupVersionPriority(metav1.GroupVersion(gv), int(entry.Group), int(entry.Version))
		}
	}

	err = aggregatorServer.GenericAPIServer.AddPostStartHook("kube-apiserver-autoregistration", func(context genericapiserver.PostStartHookContext) error {
		if crdAPIEnabled {
			go crdRegistrationController.Run(5, context.Done())
		}
		go func() {
			// let the CRD controller process the initial set of CRDs before starting the autoregistration controller.
			// this prevents the autoregistration controller's initial sync from deleting APIServices for CRDs that still exist.
			// we only need to do this if CRDs are enabled on this server.  We can't use discovery because we are the source for discovery.
			if crdAPIEnabled {
				klog.Infof("waiting for initial CRD sync...")
				crdRegistrationController.WaitForInitialSync()
				klog.Infof("initial CRD sync complete...")
			} else {
				klog.Infof("CRD API not enabled, starting APIService registration without waiting for initial CRD sync")
			}
			autoRegistrationController.Run(5, context.Done())
		}()
		return nil
	})
	if err != nil {
		return nil, err
	}

	err = aggregatorServer.GenericAPIServer.AddBootSequenceHealthChecks(
		makeAPIServiceAvailableHealthCheck(
			"autoregister-completion",
			apiServices,
			aggregatorServer.APIRegistrationInformers.Apiregistration().V1().APIServices(),
		),
	)
	if err != nil {
		return nil, err
	}

	return aggregatorServer, nil
}
```

1. `CreateAggregatorServer` 的入参如下：

   * `aggregatorConfig aggregatorapiserver.CompletedConfig`：聚合器运行配置；
   * `delegateAPIServer genericapiserver.DelegationTarget`：主 Server 的 GenericAPIServer；
   * `crds apiextensionsinformers.CustomResourceDefinitionInformer` ：CRD Informer；
   * `crdAPIEnabled bool`：CRD API 是否启用；
   * `apiVersionPriorities map[schema.GroupVersion]APIServicePriority`：API 版本优先级。

   | 参数                   | 来源                              | 作用                                                         |
   | ---------------------- | --------------------------------- | ------------------------------------------------------------ |
   | `aggregatorConfig`     | `CreateAggregatorConfig()` 创建   | 聚合器的完整配置，包含 GenericConfig、证书、代理等           |
   | `delegateAPIServer`    | 主 Server (kubeAPIServer)         | 作为 delegation chain 的下一环，聚合器会将非聚合请求委托给它 |
   | `crds`                 | 扩展 Server (apiExtensionsServer) | 监听 CRD 变化，自动注册/注销对应的 APIService                |
   | `crdAPIEnabled`        | 配置检查                          | 决定是否启动 CRD 注册控制器                                  |
   | `apiVersionPriorities` | 预定义常量                        | 控制 API 组在 discovery 中的排序                             |

2. 调用 `NewWithDelegate` 创建 `APIAggregator` 实例，传入主 Server 作为委托目标。

   ```go
   aggregatorServer, err := aggregatorConfig.NewWithDelegate(delegateAPIServer)
   ```

   * 构建聚合器的 `GenericAPIServer`；
   * 从 `delegateAPIServer` 抽取 `PostStartHooks` 放入自己的 `GenericServer`；【这里后续会着重介绍】
   * 建立请求委托链。

3. 创建自动注册控制器

   ```go
   apiRegistrationClient, err := apiregistrationclient.NewForConfig(aggregatorConfig.GenericConfig.LoopbackClientConfig)
   if err != nil {
       return nil, err
   }
   autoRegistrationController := autoregister.NewAutoRegisterController(
       aggregatorServer.APIRegistrationInformers.Apiregistration().V1().APIServices(),
       apiRegistrationClient,
   )
   apiServices := apiServicesToRegister(delegateAPIServer, autoRegistrationController, apiVersionPriorities)
   ```

   * 创建 `apiRegistrationClient`：用于操作 APIService 资源的客户端；
   * 创建 `autoRegistrationController`：自动注册控制器，负责将内置 API 注册为 APIService；
   * `apiServicesToRegister`：遍历 delegateAPIServer 暴露的 API 路径，生成需要注册的 APIService 列表。

4. 创建 CRD 注册控制器

   ```go
   type controller interface {
       Run(workers int, stopCh <-chan struct{})
       WaitForInitialSync()
   }
   var crdRegistrationController controller
   if crdAPIEnabled {
       crdRegistrationController = crdregistration.NewCRDRegistrationController(
           crds,
           autoRegistrationController,
       )
   }
   ```

   * 定义 `controller` 接口：统一 `Run` 和 `WaitForInitialSync` 方法；
   * 如果 CRD API 启用，创建 `CRDRegistrationController`；
   * 该控制器监听 CRD 变化，自动为每个 CRD 创建/删除对应的 `APIService`。

5. 设置 API 组优先级

   ```go
   if aggregatorConfig.GenericConfig.AggregatedDiscoveryGroupManager != nil {
       for gv, entry := range apiVersionPriorities {
           aggregatorConfig.GenericConfig.AggregatedDiscoveryGroupManager.SetGroupVersionPriority(
               metav1.GroupVersion(gv),
               int(entry.Group),
               int(entry.Version),
           )
       }
   }
   ```

   将预定义的 API 版本优先级注入到聚合发现管理器，控制 kubectl api-resources 等命令的输出顺序。

6. 注册启动后钩子

   ```go
   err = aggregatorServer.GenericAPIServer.AddPostStartHook("kube-apiserver-autoregistration", func(context genericapiserver.PostStartHookContext) error {
       if crdAPIEnabled {
           go crdRegistrationController.Run(5, context.Done())
       }
       go func() {
           if crdAPIEnabled {
               klog.Infof("waiting for initial CRD sync...")
               crdRegistrationController.WaitForInitialSync()
               klog.Infof("initial CRD sync complete...")
           } else {
               klog.Infof("CRD API not enabled, starting APIService registration without waiting for initial CRD sync")
           }
           autoRegistrationController.Run(5, context.Done())
       }()
       return nil
   })
   ```

   注册 `kube-apiserver-autoregistration` 钩子，Server 启动后执行：

   1. 启动 CRD 注册控制器（5 个 worker）；
   2. 等待 CRD 控制器完成初始同步（防止误删 CRD 对应的 APIService）；
   3. 启动自动注册控制器（5 个 worker）。

7. 添加健康检查

   ```go
   err = aggregatorServer.GenericAPIServer.AddBootSequenceHealthChecks(
       makeAPIServiceAvailableHealthCheck(
           "autoregister-completion",
           apiServices,
           aggregatorServer.APIRegistrationInformers.Apiregistration().V1().APIServices(),
       ),
   )
   if err != nil {
       return nil, err
   }
   
   return aggregatorServer, nil
   ```

   添加启动序列健康检查 `autoregister-completion`：

   * 监控所有需要注册的 APIService；
   * 只有当所有 APIService 都变为 Available 状态后，健康检查才通过；
   * 这确保 API Server 在所有内置 API 就绪后才对外提供服务。

### Hooks 抽取

在聚合 Server 的构建 `NewWithDelegate()` 方法中，第一步就是：

```go
genericServer, err := c.GenericConfig.New("kube-aggregator", delegationTarget)
```

在 `c.GenericConfig.New` 中会抽取出委托链的 `Hooks` 保存到 Generic Server 中：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\config.go#L879-L886
// first add poststarthooks from delegated targets
for k, v := range delegationTarget.PostStartHooks() {
    s.postStartHooks[k] = v
}

for k, v := range delegationTarget.PreShutdownHooks() {
    s.preShutdownHooks[k] = v
}
```

这里的逻辑是这样的：

1. 首先要知道，扩展 Server、主 Server、聚合 Server 都依赖于 Generic Server 来进行构建，都有自己的 `postStartHooks`

   ```go
   apiExtensionsServer.GenericAPIServer  ← 实例 A，有自己的 postStartHooks map
   kubeAPIServer.GenericAPIServer        ← 实例 B，有自己的 postStartHooks map  
   aggregatorServer.GenericAPIServer     ← 实例 C，有自己的 postStartHooks map
   ```

2. 在扩展 Server、主 Server 的构建中，也是需要调用 `c.GenericConfig.New` 方法来抽取 Hooks 的：

   * 扩展 Server：

     ```go
     // staging/src/k8s.io/apiextensions-apiserver/pkg/apiserver/apiserver.go:128
     func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*CustomResourceDefinitions, error) {
         genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
         ...
     }
     ```

   * 主 Server：

     ```go
     // pkg/controlplane/apiserver/server.go:96
     func (c completedConfig) New(name string, delegationTarget genericapiserver.DelegationTarget) (*Server, error) {
         generic, err := c.Generic.New(name, delegationTarget)
         ...
     }
     ```

3. 这样，在 CreateServerChain 中，Hooks 的流程大致如下：

   ```go
   ┌─────────────────────────────────────────────────────────────────────────────┐
   │  扩展 Server (实例 A)                                                        │
   │    postStartHooks = {钩子1, 钩子2}   ← 自己的钩子，但不会被执行               │
   └─────────────────────────────────────────────────────────────────────────────┘
                       ↓ 复制钩子到
   ┌─────────────────────────────────────────────────────────────────────────────┐
   │  主 Server (实例 B)                                                          │
   │    postStartHooks = {钩子1, 钩子2, 钩子3, 钩子4}  ← 也不会被执行              │
   └─────────────────────────────────────────────────────────────────────────────┘
                       ↓ 复制钩子到
   ┌─────────────────────────────────────────────────────────────────────────────┐
   │  聚合器 (实例 C)                                                             │
   │    postStartHooks = {钩子1, 钩子2, 钩子3, 钩子4, 钩子5, 钩子6}               │
   │                      ↑ 只有这里的钩子会被 Run() 执行                          │
   └─────────────────────────────────────────────────────────────────────────────┘
   ```

### `NewWithDelegate()` 方法构建聚合器

```go
// 代码: staging/src/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go#L204-L449
func (c completedConfig) NewWithDelegate(delegationTarget genericapiserver.DelegationTarget) (*APIAggregator, error) {
	genericServer, err := c.GenericConfig.New("kube-aggregator", delegationTarget)
	if err != nil {
		return nil, err
	}

	apiregistrationClient, err := clientset.NewForConfig(c.GenericConfig.LoopbackClientConfig)
	if err != nil {
		return nil, err
	}
	informerFactory := informers.NewSharedInformerFactory(
		apiregistrationClient,
		5*time.Minute,
	)

	apiServiceRegistrationControllerInitiated := make(chan struct{})
	if err := genericServer.RegisterMuxAndDiscoveryCompleteSignal("APIServiceRegistrationControllerInitiated", apiServiceRegistrationControllerInitiated); err != nil {
		return nil, err
	}

	var proxyTransportDial *transport.DialHolder
	if c.GenericConfig.EgressSelector != nil {
		egressDialer, err := c.GenericConfig.EgressSelector.Lookup(egressselector.Cluster.AsNetworkContext())
		if err != nil {
			return nil, err
		}
		if egressDialer != nil {
			proxyTransportDial = &transport.DialHolder{Dial: egressDialer}
		}
	} else if c.ExtraConfig.ProxyTransport != nil && c.ExtraConfig.ProxyTransport.DialContext != nil {
		proxyTransportDial = &transport.DialHolder{Dial: c.ExtraConfig.ProxyTransport.DialContext}
	}

	s := &APIAggregator{
		GenericAPIServer:           genericServer,
		delegateHandler:            delegationTarget.UnprotectedHandler(),
		proxyTransportDial:         proxyTransportDial,
		proxyHandlers:              map[string]*proxyHandler{},
		handledGroupVersions:       map[string]sets.Set[string]{},
		lister:                     informerFactory.Apiregistration().V1().APIServices().Lister(),
		APIRegistrationInformers:   informerFactory,
		serviceResolver:            c.ExtraConfig.ServiceResolver,
		openAPIConfig:              c.GenericConfig.OpenAPIConfig,
		openAPIV3Config:            c.GenericConfig.OpenAPIV3Config,
		proxyCurrentCertKeyContent: func() (bytes []byte, bytes2 []byte) { return nil, nil },
		rejectForwardingRedirects:  c.ExtraConfig.RejectForwardingRedirects,
		tracerProvider:             c.GenericConfig.TracerProvider,
	}

	apiGroupInfo := apiservicerest.NewRESTStorage(c.GenericConfig.MergedResourceConfig, c.GenericConfig.RESTOptionsGetter, false)
	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
		return nil, err
	}

	enabledVersions := sets.NewString()
	for v := range apiGroupInfo.VersionedResourcesStorageMap {
		enabledVersions.Insert(v)
	}
	if !enabledVersions.Has(v1.SchemeGroupVersion.Version) {
		return nil, fmt.Errorf("API group/version %s must be enabled", v1.SchemeGroupVersion.String())
	}

	apisHandler := &apisHandler{
		codecs:         aggregatorscheme.Codecs,
		lister:         s.lister,
		discoveryGroup: discoveryGroup(enabledVersions),
	}

	apisHandlerWithAggregationSupport := aggregated.WrapAggregatedDiscoveryToHandler(apisHandler, s.GenericAPIServer.AggregatedDiscoveryGroupManager)
	s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", apisHandlerWithAggregationSupport)
	s.GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandle("/apis/", apisHandler)

	apiserviceRegistrationController := NewAPIServiceRegistrationController(informerFactory.Apiregistration().V1().APIServices(), s)
	if len(c.ExtraConfig.ProxyClientCertFile) > 0 && len(c.ExtraConfig.ProxyClientKeyFile) > 0 {
		aggregatorProxyCerts, err := dynamiccertificates.NewDynamicServingContentFromFiles("aggregator-proxy-cert", c.ExtraConfig.ProxyClientCertFile, c.ExtraConfig.ProxyClientKeyFile)
		if err != nil {
			return nil, err
		}
		if err := aggregatorProxyCerts.RunOnce(context.Background()); err != nil {
			return nil, err
		}
		aggregatorProxyCerts.AddListener(apiserviceRegistrationController)
		s.proxyCurrentCertKeyContent = aggregatorProxyCerts.CurrentCertKeyContent

		s.GenericAPIServer.AddPostStartHookOrDie("aggregator-reload-proxy-client-cert", func(postStartHookContext genericapiserver.PostStartHookContext) error {
			go aggregatorProxyCerts.Run(postStartHookContext, 1)
			return nil
		})
	}

	s.GenericAPIServer.AddPostStartHookOrDie("start-kube-aggregator-informers", func(context genericapiserver.PostStartHookContext) error {
		informerFactory.Start(context.Done())
		c.GenericConfig.SharedInformerFactory.Start(context.Done())
		return nil
	})

	metrics := availabilitymetrics.New()
	registerIntoLegacyRegistryOnce.Do(func() { err = metrics.Register(legacyregistry.Register, legacyregistry.CustomRegister) })
	if err != nil {
		return nil, err
	}

	local, err := localavailability.New(
		informerFactory.Apiregistration().V1().APIServices(),
		apiregistrationClient.ApiregistrationV1(),
		metrics,
	)
	if err != nil {
		return nil, err
	}
	s.GenericAPIServer.AddPostStartHookOrDie("apiservice-status-local-available-controller", func(context genericapiserver.PostStartHookContext) error {
		go local.Run(5, context.Done())
		return nil
	})

	if !c.ExtraConfig.DisableRemoteAvailableConditionController {
		remote, err := remoteavailability.New(
			informerFactory.Apiregistration().V1().APIServices(),
			c.GenericConfig.SharedInformerFactory.Core().V1().Services(),
			c.GenericConfig.SharedInformerFactory.Discovery().V1().EndpointSlices(),
			apiregistrationClient.ApiregistrationV1(),
			proxyTransportDial,
			(func() ([]byte, []byte))(s.proxyCurrentCertKeyContent),
			s.serviceResolver,
			metrics,
		)
		if err != nil {
			return nil, err
		}
		s.GenericAPIServer.AddPostStartHookOrDie("apiservice-status-remote-available-controller", func(context genericapiserver.PostStartHookContext) error {
			go remote.Run(5, context.Done())
			return nil
		})
	}

	s.GenericAPIServer.AddPostStartHookOrDie("apiservice-registration-controller", func(context genericapiserver.PostStartHookContext) error {
		go apiserviceRegistrationController.Run(context.Done(), apiServiceRegistrationControllerInitiated)
		select {
		case <-context.Done():
		case <-apiServiceRegistrationControllerInitiated:
		}
		return nil
	})

	s.discoveryAggregationController = NewDiscoveryManager(
		s.GenericAPIServer.AggregatedDiscoveryGroupManager.WithSource(aggregated.AggregatorSource),
	)

	s.GenericAPIServer.AddPostStartHookOrDie("apiservice-discovery-controller", func(context genericapiserver.PostStartHookContext) error {
		select {
		case <-context.Done():
			return nil
		case <-apiServiceRegistrationControllerInitiated:
		}

		discoverySyncedCh := make(chan struct{})
		go s.discoveryAggregationController.Run(context.Done(), discoverySyncedCh)

		select {
		case <-context.Done():
			return nil
		case <-discoverySyncedCh:
		}
		return nil
	})

	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.StorageVersionAPI) &&
		utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerIdentity) {
		s.GenericAPIServer.AddPostStartHookOrDie(StorageVersionPostStartHookName, func(hookContext genericapiserver.PostStartHookContext) error {
			kubeClient, err := kubernetes.NewForConfig(hookContext.LoopbackClientConfig)
			if err != nil {
				return err
			}
			if err := wait.PollImmediateUntil(100*time.Millisecond, func() (bool, error) {
				_, err := kubeClient.CoordinationV1().Leases(metav1.NamespaceSystem).Get(
					context.TODO(), s.GenericAPIServer.APIServerID, metav1.GetOptions{})
				if apierrors.IsNotFound(err) {
					return false, nil
				}
				if err != nil {
					return false, err
				}
				return true, nil
			}, hookContext.Done()); err != nil {
				return fmt.Errorf("failed to wait for apiserver-identity lease %s to be created: %v",
					s.GenericAPIServer.APIServerID, err)
			}
			go wait.PollImmediateUntil(10*time.Minute, func() (bool, error) {
				s.GenericAPIServer.StorageVersionManager.UpdateStorageVersions(hookContext.LoopbackClientConfig, s.GenericAPIServer.APIServerID)
				return false, nil
			}, hookContext.Done())
			wait.PollImmediateUntil(1*time.Second, func() (bool, error) {
				return s.GenericAPIServer.StorageVersionManager.Completed(), nil
			}, hookContext.Done())
			return nil
		})
	}

	return s, nil
}
```

1. `NewWithDelegate` 的入参如下：

   * `c completedConfig`：已完成的聚合器配置（方法接收者）；
   * `delegationTarget genericapiserver.DelegationTarget`：委托目标，即 delegation chain 中的下一个服务器。

   | 参数               | 来源                         | 作用                                                         |
   | ------------------ | ---------------------------- | ------------------------------------------------------------ |
   | `c`                | `Config.Complete()` 方法返回 | 包含 `GenericConfig` 和 `ExtraConfig`，提供聚合器所需的全部配置 |
   | `delegationTarget` | 主 Server (kubeAPIServer)    | 作为 delegation chain 的下一环，聚合器会将无法处理的请求委托给它 |

2. 创建 GenericAPIServer 和 Informer

   ```go
   genericServer, err := c.GenericConfig.New("kube-aggregator", delegationTarget)
   if err != nil {
       return nil, err
   }
   
   apiregistrationClient, err := clientset.NewForConfig(c.GenericConfig.LoopbackClientConfig)
   if err != nil {
       return nil, err
   }
   informerFactory := informers.NewSharedInformerFactory(
       apiregistrationClient,
       5*time.Minute,
   )
   ```

   * 调用 `GenericConfig.New()` 创建名为 `"kube-aggregator"` 的通用 API Server；
   * 创建 `apiregistrationClient`：用于操作 `APIService` 资源的客户端；
   * 创建 `informerFactory`：监听 `APIService` 资源变化，5 分钟刷新一次缓存。

3. 注册初始化完成信号

   ```go
   apiServiceRegistrationControllerInitiated := make(chan struct{})
   if err := genericServer.RegisterMuxAndDiscoveryCompleteSignal("APIServiceRegistrationControllerInitiated", apiServiceRegistrationControllerInitiated); err != nil {
       return nil, err
   }
   ```

   * 创建 `apiServiceRegistrationControllerInitiated` channel 作为信号；
   * 当 `APIServiceRegistrationController` 完成初始化时关闭该 channel；
   * **作用**：防止在控制器就绪前返回 404，避免影响 GC、Namespace 等关键控制器。

4. 配置代理传输层

   ```go
   var proxyTransportDial *transport.DialHolder
   if c.GenericConfig.EgressSelector != nil {
       egressDialer, err := c.GenericConfig.EgressSelector.Lookup(egressselector.Cluster.AsNetworkContext())
       if err != nil {
           return nil, err
       }
       if egressDialer != nil {
           proxyTransportDial = &transport.DialHolder{Dial: egressDialer}
       }
   } else if c.ExtraConfig.ProxyTransport != nil && c.ExtraConfig.ProxyTransport.DialContext != nil {
       proxyTransportDial = &transport.DialHolder{Dial: c.ExtraConfig.ProxyTransport.DialContext}
   }
   ```

   * 配置出站网络拨号器，用于连接后端扩展 API Server；
   * 优先使用 `EgressSelector`（Konnectivity 等安全隧道）；
   * 否则回退到 `ProxyTransport`。

5. 初始化 APIAggregator 结构体

   ```go
   s := &APIAggregator{
       GenericAPIServer:           genericServer,
       delegateHandler:            delegationTarget.UnprotectedHandler(),
       proxyTransportDial:         proxyTransportDial,
       proxyHandlers:              map[string]*proxyHandler{},
       handledGroupVersions:       map[string]sets.Set[string]{},
       lister:                     informerFactory.Apiregistration().V1().APIServices().Lister(),
       APIRegistrationInformers:   informerFactory,
       serviceResolver:            c.ExtraConfig.ServiceResolver,
       openAPIConfig:              c.GenericConfig.OpenAPIConfig,
       openAPIV3Config:            c.GenericConfig.OpenAPIV3Config,
       proxyCurrentCertKeyContent: func() (bytes []byte, bytes2 []byte) { return nil, nil },
       rejectForwardingRedirects:  c.ExtraConfig.RejectForwardingRedirects,
       tracerProvider:             c.GenericConfig.TracerProvider,
   }
   ```

   创建 `APIAggregator` 核心结构体：

   | 字段                   | 作用                                        |
   | ---------------------- | ------------------------------------------- |
   | `GenericAPIServer`     | 通用 API Server，处理请求路由、认证、审计等 |
   | `delegateHandler`      | 委托处理器，将无法处理的请求转发给下一层    |
   | `proxyTransportDial`   | 代理传输拨号器，用于连接扩展 API Server     |
   | `proxyHandlers`        | 代理处理器映射表，key 为 APIService 名称    |
   | `handledGroupVersions` | 已注册的 API Group/Version 集合             |
   | `lister`               | APIService 列表器，用于查询 APIService      |
   | `serviceResolver`      | 服务解析器，将 Service 解析为实际地址       |

6. 安装 apiregistration API Group

   ```go
   apiGroupInfo := apiservicerest.NewRESTStorage(c.GenericConfig.MergedResourceConfig, c.GenericConfig.RESTOptionsGetter, false)
   if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
       return nil, err
   }
   
   enabledVersions := sets.NewString()
   for v := range apiGroupInfo.VersionedResourcesStorageMap {
       enabledVersions.Insert(v)
   }
   if !enabledVersions.Has(v1.SchemeGroupVersion.Version) {
       return nil, fmt.Errorf("API group/version %s must be enabled", v1.SchemeGroupVersion.String())
   }
   ```

   * 创建 `apiregistration.k8s.io` API Group 的 REST 存储；
   * 安装该 API Group，使 `APIService` 资源可被 CRUD 操作；
   * 强制要求 v1 版本必须启用。

7. 配置 `/apis` 发现端点

   ```go
   apisHandler := &apisHandler{
       codecs:         aggregatorscheme.Codecs,
       lister:         s.lister,
       discoveryGroup: discoveryGroup(enabledVersions),
   }
   
   apisHandlerWithAggregationSupport := aggregated.WrapAggregatedDiscoveryToHandler(apisHandler, s.GenericAPIServer.AggregatedDiscoveryGroupManager)
   s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", apisHandlerWithAggregationSupport)
   s.GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandle("/apis/", apisHandler)
   ```

   * 创建 `apisHandler` 处理 `/apis` 端点请求；
   * `WrapAggregatedDiscoveryToHandler` 包装支持聚合发现协议；
   * 注册 `/apis` 和 `/apis/` 路由。

8. 配置动态证书加载

   ```go
   apiserviceRegistrationController := NewAPIServiceRegistrationController(informerFactory.Apiregistration().V1().APIServices(), s)
   if len(c.ExtraConfig.ProxyClientCertFile) > 0 && len(c.ExtraConfig.ProxyClientKeyFile) > 0 {
       aggregatorProxyCerts, err := dynamiccertificates.NewDynamicServingContentFromFiles("aggregator-proxy-cert", c.ExtraConfig.ProxyClientCertFile, c.ExtraConfig.ProxyClientKeyFile)
       if err != nil {
           return nil, err
       }
       if err := aggregatorProxyCerts.RunOnce(context.Background()); err != nil {
           return nil, err
       }
       aggregatorProxyCerts.AddListener(apiserviceRegistrationController)
       s.proxyCurrentCertKeyContent = aggregatorProxyCerts.CurrentCertKeyContent
   
       s.GenericAPIServer.AddPostStartHookOrDie("aggregator-reload-proxy-client-cert", func(postStartHookContext genericapiserver.PostStartHookContext) error {
           go aggregatorProxyCerts.Run(postStartHookContext, 1)
           return nil
       })
   }
   ```

   * 创建 `APIServiceRegistrationController`：监听 APIService 变化，注册/注销代理处理器；
   * 如果配置了代理客户端证书，启用动态加载；
   * 证书变更时通知 `apiserviceRegistrationController`，实现热更新；
   * 注册 `aggregator-reload-proxy-client-cert` 钩子启动证书监听。

9. 注册 Informer 启动钩子

   ```go
   s.GenericAPIServer.AddPostStartHookOrDie("start-kube-aggregator-informers", func(context genericapiserver.PostStartHookContext) error {
       informerFactory.Start(context.Done())
       c.GenericConfig.SharedInformerFactory.Start(context.Done())
       return nil
   })
   ```

   * 注册 `start-kube-aggregator-informers` 钩子；
   * Server 启动后启动所有 Informer，开始监听资源变化。

10. 创建可用性监控控制器

    ```go
    metrics := availabilitymetrics.New()
    registerIntoLegacyRegistryOnce.Do(func() { err = metrics.Register(legacyregistry.Register, legacyregistry.CustomRegister) })
    if err != nil {
        return nil, err
    }
    
    // 本地可用性控制器（始终运行）
    local, err := localavailability.New(
        informerFactory.Apiregistration().V1().APIServices(),
        apiregistrationClient.ApiregistrationV1(),
        metrics,
    )
    if err != nil {
        return nil, err
    }
    s.GenericAPIServer.AddPostStartHookOrDie("apiservice-status-local-available-controller", func(context genericapiserver.PostStartHookContext) error {
        go local.Run(5, context.Done())
        return nil
    })
    
    // 远程可用性控制器（可禁用）
    if !c.ExtraConfig.DisableRemoteAvailableConditionController {
        remote, err := remoteavailability.New(
            informerFactory.Apiregistration().V1().APIServices(),
            c.GenericConfig.SharedInformerFactory.Core().V1().Services(),
            c.GenericConfig.SharedInformerFactory.Discovery().V1().EndpointSlices(),
            apiregistrationClient.ApiregistrationV1(),
            proxyTransportDial,
            (func() ([]byte, []byte))(s.proxyCurrentCertKeyContent),
            s.serviceResolver,
            metrics,
        )
        if err != nil {
            return nil, err
        }
        s.GenericAPIServer.AddPostStartHookOrDie("apiservice-status-remote-available-controller", func(context genericapiserver.PostStartHookContext) error {
            go remote.Run(5, context.Done())
            return nil
        })
    }
    ```

    * 创建可用性指标收集器，注册到 Prometheus；
    * `local` 控制器：监控本地 APIService（如核心 API、CRD）的可用性；
    * `remote` 控制器：监控远程扩展 API Server 的健康状态（可通过配置禁用）；
    * 两个控制器都以 5 个 worker 运行。

11. 注册 APIService 注册控制器钩子

    ```go
    s.GenericAPIServer.AddPostStartHookOrDie("apiservice-registration-controller", func(context genericapiserver.PostStartHookContext) error {
        go apiserviceRegistrationController.Run(context.Done(), apiServiceRegistrationControllerInitiated)
        select {
        case <-context.Done():
        case <-apiServiceRegistrationControllerInitiated:
        }
        return nil
    })
    ```

    * 启动 `APIServiceRegistrationController`；
    * 等待控制器完成初始化后才让 hook 返回；
    * **作用**：确保 `/apis` 端点在控制器就绪后才开始服务请求。

12. 创建 Discovery 聚合控制器

    ```go
    s.discoveryAggregationController = NewDiscoveryManager(
        s.GenericAPIServer.AggregatedDiscoveryGroupManager.WithSource(aggregated.AggregatorSource),
    )
    
    s.GenericAPIServer.AddPostStartHookOrDie("apiservice-discovery-controller", func(context genericapiserver.PostStartHookContext) error {
        select {
        case <-context.Done():
            return nil
        case <-apiServiceRegistrationControllerInitiated:
        }
    
        discoverySyncedCh := make(chan struct{})
        go s.discoveryAggregationController.Run(context.Done(), discoverySyncedCh)
    
        select {
        case <-context.Done():
            return nil
        case <-discoverySyncedCh:
        }
        return nil
    })
    ```

    * 创建 `DiscoveryManager`，使用 `AggregatorSource` 作为来源标识；
    * 等待 `apiServiceRegistrationControllerInitiated` 信号后启动；
    * 从各个 APIService 收集 discovery 信息，合并成统一视图；
    * 等待 discovery 同步完成后 hook 才返回。

13. 注册 StorageVersion 更新钩子（Feature Gate 控制）

    ```go
    if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.StorageVersionAPI) &&
        utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerIdentity) {
        s.GenericAPIServer.AddPostStartHookOrDie(StorageVersionPostStartHookName, func(hookContext genericapiserver.PostStartHookContext) error {
            kubeClient, err := kubernetes.NewForConfig(hookContext.LoopbackClientConfig)
            if err != nil {
                return err
            }
            // 等待 apiserver-identity lease 创建
            if err := wait.PollImmediateUntil(100*time.Millisecond, func() (bool, error) {
                _, err := kubeClient.CoordinationV1().Leases(metav1.NamespaceSystem).Get(
                    context.TODO(), s.GenericAPIServer.APIServerID, metav1.GetOptions{})
                if apierrors.IsNotFound(err) {
                    return false, nil
                }
                if err != nil {
                    return false, err
                }
                return true, nil
            }, hookContext.Done()); err != nil {
                return fmt.Errorf("failed to wait for apiserver-identity lease %s to be created: %v",
                    s.GenericAPIServer.APIServerID, err)
            }
            // 每 10 分钟协调一次 StorageVersion
            go wait.PollImmediateUntil(10*time.Minute, func() (bool, error) {
                s.GenericAPIServer.StorageVersionManager.UpdateStorageVersions(hookContext.LoopbackClientConfig, s.GenericAPIServer.APIServerID)
                return false, nil
            }, hookContext.Done())
            // 等待首次更新完成
            wait.PollImmediateUntil(1*time.Second, func() (bool, error) {
                return s.GenericAPIServer.StorageVersionManager.Completed(), nil
            }, hookContext.Done())
            return nil
        })
    }
    
    return s, nil
    ```

    当启用 `StorageVersionAPI` 和 `APIServerIdentity` 特性时：

    1. 等待 `apiserver-identity` lease 创建（每 100ms 检查一次）；
    2. 每 10 分钟协调一次 `StorageVersion` 对象；
    3. 等待首次更新完成后 hook 才返回。

    **作用**：用于存储迁移，确保所有内置资源的存储版本信息正确。

## 启动聚合器

作为核心 API Server 的 Server 链头，聚合器的启动也就是核心 API Server 的启动。处于聚合器下游的主 Server 和扩展 Server 需要在启动前后执行的逻辑都会由聚合器的启动触发执行，例如逐个调用子 Server 所注册的启动钩子函数。

用户通过命令行启动 API Server 时，`Run()` 方法会被调用，它触发构造 Server 链，然后调用链头节点的 `PrepareRun()` 和 `Run()` 方法完成启动，关键代码如下所示：

```go
// 代码: cmd\kube-apiserver\app\server.go#L148-L173
// Run runs the specified APIServer.  This should never exit.
func Run(ctx context.Context, opts options.CompletedOptions) error {
	...
	prepared, err := server.PrepareRun()
	if err != nil {
		return err
	}

	return prepared.Run(ctx)
}

```

上述代码中变量 server 即是聚合器，由此可见，分析 `PrepareRun()` 与 `Run()` 的实现是剖析聚合器启动的关键。

### PrepareRun

顾名思义，`PrepareRun()` 方法进行启动准备。这包括聚合器的自身准备逻辑和调用其 delegation 的 `PrepareRun()` 方法。Delegation 实际上是主 Server 的底座 Generic Server，而它的 `PrepareRun()` 逻辑已经在 Generic Server 讲解过，这一小节聚焦聚合器所含两项工作准备逻辑分别在下面小节中深入讲解：

* **注册 OpenAPI 聚合控制器的 PostStartHook**，定义服务启动后要运行的控制器；
* 调用底层 Generic Server PrepareRun 函数；

* 为 **OpenAPI 的端点设置响应机制**：完善后的配置信息中已明确指出不希望其底层 Generic Server 为 OpenAPI 的端点设置响应器，而是由自己在这里单独设置。

```go
// 代码: staging\src\k8s.io\kube-aggregator\pkg\apiserver\apiserver.go#L453-L501
// PrepareRun prepares the aggregator to run, by setting up the OpenAPI spec &
// aggregated discovery document and calling the generic PrepareRun.
func (s *APIAggregator) PrepareRun() (preparedAPIAggregator, error) {
	// add post start hook before generic PrepareRun in order to be before /healthz installation
	if s.openAPIConfig != nil {
		s.GenericAPIServer.AddPostStartHookOrDie("apiservice-openapi-controller", func(context genericapiserver.PostStartHookContext) error {
			go s.openAPIAggregationController.Run(context.Done())
			return nil
		})
	}

	if s.openAPIV3Config != nil {
		s.GenericAPIServer.AddPostStartHookOrDie("apiservice-openapiv3-controller", func(context genericapiserver.PostStartHookContext) error {
			go s.openAPIV3AggregationController.Run(context.Done())
			return nil
		})
	}

	prepared := s.GenericAPIServer.PrepareRun()

	// delay OpenAPI setup until the delegate had a chance to setup their OpenAPI handlers
	if s.openAPIConfig != nil {
		specDownloader := openapiaggregator.NewDownloader()
		openAPIAggregator, err := openapiaggregator.BuildAndRegisterAggregator(
			&specDownloader,
			s.GenericAPIServer.NextDelegate(),
			s.GenericAPIServer.Handler.GoRestfulContainer.RegisteredWebServices(),
			s.openAPIConfig,
			s.GenericAPIServer.Handler.NonGoRestfulMux)
		if err != nil {
			return preparedAPIAggregator{}, err
		}
		s.openAPIAggregationController = openapicontroller.NewAggregationController(&specDownloader, openAPIAggregator)
	}

	if s.openAPIV3Config != nil {
		specDownloaderV3 := openapiv3aggregator.NewDownloader()
		openAPIV3Aggregator, err := openapiv3aggregator.BuildAndRegisterAggregator(
			specDownloaderV3,
			s.GenericAPIServer.NextDelegate(),
			s.GenericAPIServer.Handler.GoRestfulContainer,
			s.openAPIV3Config,
			s.GenericAPIServer.Handler.NonGoRestfulMux)
		if err != nil {
			return preparedAPIAggregator{}, err
		}
		s.openAPIV3AggregationController = openapiv3controller.NewAggregationController(openAPIV3Aggregator)
	}

	return preparedAPIAggregator{APIAggregator: s, runnable: prepared}, nil
}
```

#### 注册 OpenAPI 聚合控制器的钩子

```go
	// add post start hook before generic PrepareRun in order to be before /healthz installation
	if s.openAPIConfig != nil {
		s.GenericAPIServer.AddPostStartHookOrDie("apiservice-openapi-controller", func(context genericapiserver.PostStartHookContext) error {
			go s.openAPIAggregationController.Run(context.Done())
			return nil
		})
	}

	if s.openAPIV3Config != nil {
		s.GenericAPIServer.AddPostStartHookOrDie("apiservice-openapiv3-controller", func(context genericapiserver.PostStartHookContext) error {
			go s.openAPIV3AggregationController.Run(context.Done())
			return nil
		})
	}
```

* `apiservice-openapi-controller` → 服务启动后，运行 `openAPIAggregationController.Run()`，持续从各 APIService 拉取并聚合 OpenAPI v2 规范
* `apiservice-openapiv3-controller` → 同上，但针对 OpenAPI v3。

#### GenericServer PrepareRun

```go
prepared := s.GenericAPIServer.PrepareRun()
```

* `delegationTarget.PrepareRun()` → 递归调用委托链上所有 server 的 PrepareRun
* `InstallV2() `→ 注册 /openapi/v2 端点
* `InstallV3()` → 注册 /openapi/v3 端点
* `installHealthz()` → 注册 /healthz 端点
* `installLivez()` → 注册 /livez 端点
* `installReadyz()` → 注册 /readyz 端点
* `statusz.Install()` → 注册状态页端点（如果启用了 ComponentStatusz 特性）

#### 设置 OpenAPI 端点响应器

OpenAPI 的端点 `/openapi/v3/apis` 返回一个规格说明，包含当前 API Server 所支持的所有 API 组及版本；访问端点 `/openapi/v3/apis/<组>/<版本>`（访问核心 API 时将 apis 替换为 api 并省略组信息）则会返回该组版本下所有 API 的 RESTful 服务规格说明。聚合器自身管理的只是 APIService 这一个 Kubernetes API，其它的 API 的 OpenAPI 规格说明需要从各个子 Server 中获取。如果等到请求到来时去轮询子 Server，则有获取效率的问题。设想一下每个子 Server 都去查询 ETCD 找出相应信息，结果不会快。相比核心 API Server，聚合 Server 的返回效率就更加不可控了。为缓解这一问题，聚合器采用了缓存策略：把所有 OpenAPI 规格说明文档从各个子 Server 上收集过来存于本地缓存，当有规格说明的变化时也会通过一个控制器来更新缓存，基于这些信息去响应对 OpenAPI 规格的请求。

##### 构造 OpenAPI 规格下载器

```go
specDownloaderV3 := openapiv3aggregator.NewDownloader()
```

OpenAPI 规格下载器具有从各个子 Server 的 OpenAPI 端点上下载其规格说明的能力。

##### 构造并注册 OpenAPI 端点响应器

访问 `/openapi/v3` 端点的 HTTP 请求会被专有响应器处理。所有子 Server 中 API 的 OpenAPI 规格说明就是缓存于该响应器内部，规格说明的下载也发生于此。函数 `BuildAndRegisterAggregator()` 完成了相关设置，代码如下：

```go
// BuildAndRegisterAggregator registered OpenAPI aggregator handler. This function is not thread safe as it only being called on startup.
func BuildAndRegisterAggregator(downloader Downloader, delegationTarget server.DelegationTarget, aggregatorService *restful.Container, openAPIConfig *common.OpenAPIV3Config, pathHandler common.PathHandlerByGroupVersion) (SpecProxier, error) {
    //要点①
	s := &specProxier{
		apiServiceInfo: map[string]*openAPIV3APIServiceInfo{},
		downloader:     downloader,
	}

	if aggregatorService != nil && openAPIConfig != nil { 
		// Make native types exposed by aggregator available to the aggregated
		// OpenAPI (normal handle is disabled by skipOpenAPIInstallation option)
		aggregatorLocalServiceName := "k8s_internal_local_kube_aggregator_types"
		v3Mux := mux.NewPathRecorderMux(aggregatorLocalServiceName)
		_ = routes.OpenAPI{
			V3Config: openAPIConfig,
		}.InstallV3(aggregatorService, v3Mux)

		s.AddUpdateAPIService(v3Mux, &v1.APIService{
			ObjectMeta: metav1.ObjectMeta{
				Name: aggregatorLocalServiceName,
			},
		})
		s.UpdateAPIServiceSpec(aggregatorLocalServiceName)
	}

	i := 1
	for delegate := delegationTarget; delegate != nil; delegate = delegate.NextDelegate() { //要点④
		handler := delegate.UnprotectedHandler()
		if handler == nil {
			continue
		}

		apiServiceName := fmt.Sprintf(localDelegateChainNamePattern, i)
		localAPIService := v1.APIService{}
		localAPIService.Name = apiServiceName
		s.AddUpdateAPIService(handler, &localAPIService)
		s.UpdateAPIServiceSpec(apiServiceName) //要点③
		i++
	}

	handler := handler3.NewOpenAPIService()
	s.openAPIV2ConverterHandler = handler
	openAPIV2ConverterMux := mux.NewPathRecorderMux(openAPIV2Converter)
	s.openAPIV2ConverterHandler.RegisterOpenAPIV3VersionedService("/openapi/v3", openAPIV2ConverterMux)
	openAPIV2ConverterAPIService := v1.APIService{}
	openAPIV2ConverterAPIService.Name = openAPIV2Converter
	s.AddUpdateAPIService(openAPIV2ConverterMux, &openAPIV2ConverterAPIService)
	s.register(pathHandler) //要点②

	return s, nil
}
```

上述代要点①处声明的结构体 specProxier 实例 s 将被设置为处理针对 `/openapi/v3` 或以 `/openapi/v3/` 开头的 HTTP 请求，这是在要点②`s.register(pathHandler)`完成的。

```go
// 代码: staging\src\k8s.io\kube-aggregator\pkg\controllers\openapiv3\aggregator\aggregator.go#L297-L319
// Register registers the OpenAPI V3 Discovery and GroupVersion handlers
func (s *specProxier) register(handler common.PathHandlerByGroupVersion) {
	handler.Handle("/openapi/v3", metrics.InstrumentHandlerFunc("GET",
		/* group = */ "",
		/* version = */ "",
		/* resource = */ "",
		/* subresource = */ "openapi/v3",
		/* scope = */ "",
		/* component = */ "",
		/* deprecated */ false,
		/* removedRelease */ "",
		http.HandlerFunc(s.handleDiscovery)))
	handler.HandlePrefix("/openapi/v3/", metrics.InstrumentHandlerFunc("GET",
		/* group = */ "",
		/* version = */ "",
		/* resource = */ "",
		/* subresource = */ "openapi/v3/",
		/* scope = */ "",
		/* component = */ "",
		/* deprecated */ false,
		/* removedRelease */ "",
		http.HandlerFunc(s.handleGroupVersion)))
}
```

specProxier 结构体的使用上述的两个方法分别处理上述两类端点上的请求。

该 specProxier 实例的字段 apiServiceInfo 上缓存了全部 OpenAPI 规格说明信息，规格说明的数据结构为 `openAPIV3APIServiceInfo` 结构体。apiServiceInfo 是一个 map，它的 key 类型为 string，value 类型为结构体 `openAPIV3APIServiceInfo`。两者关系如下：

![image-20251215143301843](http://images.liangning7.cn/typora/202512151433046.png)

结构体 openAPIV3APIServiceInfo 主要字段意义为：

* `apiService` 字段：代表一个 APIService API 的实例，但并非严格如此，聚合器在处理过程中造一些虚拟的 APIService 实例。
* `handler` 字段：聚合器会触发它对 `/openapi/v3` 端点的响应，从而获取这个 APIService 实例的 OpenAPI 规格说明。当 apiService 代表一个 API 组版本时，得到的结果将是这个 API 组版本下所有 API 的规格说明。
* discovery 字段：由 handler 返回的结果将保存在这个字段中。是聚合器的发现信息直接信息源。

当聚合器的 `/openapi/v3` 端点被访问时，`specProxier.handleDiscovery()` 方法将进行响应。 这个方法将进行嵌套的双重遍历：第一层遍历字段 apiServiceInfo，第二层遍历每个 apiServiceInfo 元素的 discovery 字段，用 discovery 的信息形成对请求的响应。

当聚合器的 `/openapi/v3/apis/<组>/<版本>` 端点被访问时， `specProxier.handleGroupVersion()` 方法将进行响应，它也是通过上述双重遍历找到目标组与版本对应的 discovery 并形成响应结果。

由此可见，apiServiceInfo 是聚合器响应 OpenAPI 规格请求的核心变量。该变量的内容填写并非一步到位，方法 `BuildAndRegisterAggregator()` 会将核心 API Server 的内置 API（除了 APIService）以子 Server 为单位加载进去，这是通过创建 specProxier 时向 apiServiceInfo 添加两个虚拟 APIService 实例来做到的，见`BuildAndRegisterAggregator()` 方法要点④处的 for 循环。这两条 apiServiceInfo 记录的关键信息如下：

* key 为 `k8s_internal_local_delegation_chain_1`，value 的 handler 被设置为主 Server 的 `UnprotectedHandler`。
* key 为 `k8s_internal_local_delegation_chain_2`，value 的 handler 被设置为扩展 Server 的 `UnprotectedHandler`。

紧接着要点③触发了对 `value.handler` 的调用，于是主 Server 与扩展 Server 中内置 API 的端点信息被加载到 `value.discovery` 字段内。`BuildAndRegisterAggregator()` 方法遗留了部分 API 没有加载，包括聚合器自己的 API——APIService 和来自聚合 Server 的 API。遗留不做是有原因的。`BuildAndRegisterAggregator()` 方法为 APIService API 的加载做了一些准备工作，它将 key 为 openapiv2converter、value 为聚合器底座 Generic Server 提供的 `/openapi/v3` 处理器加入了 `apiServiceInfo` 中，但没有触发下载，因为在这段代码运行之时聚合器正在启动，还没有能力响应对端点的请求。`BuildAndRegisterAggregator()` 方法无法加载来自聚合 Server 的 API 是由于聚合 Server 的热插拔属性，聚合器不能假设在它启动时聚合 Server 已经就位。

考虑到上述待加载的信息，以及 CRD 与聚合 Server 引入客制化 API 的动态性，聚合器设立了 OpenAPI 规格说明控制器进行动态加载。

##### 制作 OpenAPI 聚合控制器

```go
s.openAPIV3AggregationController = openapiv3controller.NewAggregationController(openAPIV3Aggregator)
```

```go
// 代码: staging\src\k8s.io\kube-aggregator\pkg\controllers\openapiv3\controller.go#L56-L73
// NewAggregationController creates new OpenAPI aggregation controller.
func NewAggregationController(openAPIAggregationManager aggregator.SpecProxier) *AggregationController {
	c := &AggregationController{
		openAPIAggregationManager: openAPIAggregationManager,
		queue: workqueue.NewTypedRateLimitingQueueWithConfig(
			workqueue.NewTypedItemExponentialFailureRateLimiter[string](successfulUpdateDelay, failedUpdateMaxExpDelay),
			workqueue.TypedRateLimitingQueueConfig[string]{Name: "open_api_v3_aggregation_controller"},
		),
	}

	c.syncHandler = c.sync

	// update each service at least once, also those which are not coming from APIServices, namely local services
	for _, name := range openAPIAggregationManager.GetAPIServiceNames() {
		c.queue.AddAfter(name, time.Second)
	}

	return c
}
```

`PrepareRun()` 方法通过调用方法 `openapiv3controller.NewAggregationController()` 创建出一个 OpenAPI 聚合控制器，将其保存在聚合器基座结构体的 `openAPIV3AggregationController` 字段上。控制器的构造方法以一个 specProxier 实例作为形参，实参用的就是上文所创建的 specProxier 实例。这个控制器的控制循环只做一件事情：利用 specProxier 实例，为其工作队列中的 APIService 实例重新下载 API 组版本的 OpenAPI 规格说明，并更新 specProxier 实例上的缓存——apiServiceInfo 字段。在如下情况中会向该控制器的工作队列中添加内容：

1. 在创建该控制器时，specProxier 实例的 apiServiceInfo 内保有的 APIService 信息都会被加入到工作队列。
2. 当有新 APIService 实例变动时，聚合器会调用本控制器的 `AddAPIService()` 、 `UpdateAPIService()` 、`RemoveAPIService()`方法，将目标 APIService 加入到控制器工作队列 中。

这样，在 OpenAPI 聚合控制器的协助下，聚合器的 `/openapi/v3` 端点响应器——上述 specProxier 结构体实例将始终缓存 API Server 的 API 组版本下所有 API 的 OpenAPI 规格说明书。当客户端请求时可以直接从缓存中取出信息返回，大大提升响应效率。

#### 设置 API 信息发现响应器

在理解了 OpenAPI 端点响应器设置后，设置 API 信息发现响应器就容易多了。响应 API 信息发现请求和响应 OpenAPI 规格说明请求具有类似的难点：结果分布在整个 API Server 的各个子 Server 上，为了提速聚合器需要在本地缓存这些信息。而代码的实现思路几乎一致。不过该部分位于 `NewWithDelegate` 函数中，为什么“设置 API 信息发现响应器”可以放在 `NewWithDelegate` 中，而设置 OpenAPI 端点响应器放在 `PrepareRun` 函数中？

API 信息发现响应器用于响应 API 发现请求，让客户端（如 kubectl）知道集群中有哪些 API 资源可用。初始化时是否需要立即获取 delegate 的数据，所以可以在 `NewWithDelegate` 中进行设置，而 OpenAPI 端点响应器依赖 delegate 的数据，就必须等待 delegate 的数据准备完成后进行设置。下面来看 API 信息发现响应器是如何设置的吧。

```go
	// 代码: staging\src\k8s.io\kube-aggregator\pkg\apiserver\apiserver.go#L364-L396
	s.discoveryAggregationController = NewDiscoveryManager( //要点①
		// Use aggregator as the source name to avoid overwriting native/CRD
		// groups
		s.GenericAPIServer.AggregatedDiscoveryGroupManager.WithSource(aggregated.AggregatorSource),
	)

	// Setup discovery endpoint
	//要点②
	s.GenericAPIServer.AddPostStartHookOrDie("apiservice-discovery-controller", func(context genericapiserver.PostStartHookContext) error {
		// Discovery aggregation depends on the apiservice registration controller
		// having the full list of APIServices already synced
		select {
		case <-context.Done():
			return nil
		// Context cancelled, should abort/clean goroutines
		case <-apiServiceRegistrationControllerInitiated:
		}

		// Run discovery manager's worker to watch for new/removed/updated
		// APIServices to the discovery document can be updated at runtime
		// When discovery is ready, all APIServices will be present, with APIServices
		// that have not successfully synced discovery to be present but marked as Stale.
		discoverySyncedCh := make(chan struct{})
		go s.discoveryAggregationController.Run(context.Done(), discoverySyncedCh)

		select {
		case <-context.Done():
			return nil
		// Context cancelled, should abort/clean goroutines
		case <-discoverySyncedCh:
			// API services successfully sync
		}
		return nil
	})
```

这里的核心是制作并启动 API 发现聚合控制器。它首先在要点①处创建一个 API 发现聚合控制器，并保存在聚合器的 discoveryAggregationController 属性上，然后要点②处制作启动后钩子函数，用于启动该控制器。由要点①可见，API 发现聚合控制器是基于聚合器的 `GenericAPIServer.AggregatedDiscoveryGroupManager` 属性所创建，这个属性内部缓存了 所有 API 的发现信息，控制器的控制循环会不断更新它。

AggregatedDiscoveryGroupManager 属性很重要，因为它是 /apis 端点的请求响应器，这 是方法 `NewWithDelegate()` 在创建聚合器时做的设置，代码如下所示：

```go
// 1. 创建 /apis 端点的 handler
apisHandler := &apisHandler{
    codecs:         aggregatorscheme.Codecs,
    lister:         s.lister,
    discoveryGroup: discoveryGroup(enabledVersions),
}

// 2. 包装支持聚合发现
apisHandlerWithAggregationSupport := aggregated.WrapAggregatedDiscoveryToHandler(
    apisHandler, 
    s.GenericAPIServer.AggregatedDiscoveryGroupManager,
)

// 3. 注册到路由 - 这才是"设置响应器"
s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", apisHandlerWithAggregationSupport)
s.GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandle("/apis/", apisHandler)

```

API 发现聚合控制器基于如图所示结构体构建：

![image-20251215153348483](http://images.liangning7.cn/typora/202512151533770.png)

1. 字段 `mergedDiscoveryHandler` 保有聚合器的 `AggregatedDiscoveryGroupManager`，在控制循环中它内部信息会被更新从而一直具有各个 APIService 所代表的最新 API 信息。
2. 字段 `apiServices` 扮演了控制器的工作队列的角色，该队列内容的生产者是方法 `AddAPIService()` 和方法 `RemoveAPIService()`，当有新的 APIService 实例变动时，这两个方法会被调用让目标 APIService 入队，下一次控制循环就会针对它进行更新。
3. 方法 `syncAPIService` 是控制循环的主要逻辑，其内主要逻辑只有一个：更新 `mergedDiscoveryHandler`。

在 API 发现聚合控制器的辅助下，`/apis` 端点响应器内一直具有最新的 API 信息，可直 接应用于查询响应，大大提高了效率。

### Run

启动聚合器的下一步是执行经 PrepareRun()方法处理后的聚合器实例，该实例有 `Run()` 方法。和 `PrepareRun()` 的任务繁重完全不同，Run 方法如此之简单：

```go
// 代码: staging\src\k8s.io\kube-aggregator\pkg\apiserver\apiserver.go#L503-L505
func (s preparedAPIAggregator) Run(ctx context.Context) error {
	return s.runnable.RunWithContext(ctx)
}
```

## 聚合器代理转发 HTTP 请求

聚合器最为显著的特点是其需要转发针对 Kubernetes API 的 HTTP 请求到正确的子 Server，毕竟它自己只能处理针对 APIService 的请求。下面梳理它的转发机制。

可以用如下命令来查询某一个 API 实例的详细信息：

```bash
$ kubectl describe <API> <实例名字>
```

例如请求聚合器直接管理的名为“v1.”的 APIService 实例，命令如下：

```bash
$ kubectl describe APIService v1.
```

注意，目标 API 是由哪个子 Server 管理对客户端透明，因为命令中根本没有指定。请求发出后，kubectl 首先根据用户输入组织出要访问的 URL，格式为 `https://ip:port/apis////<实例名>`，然后向这个 URL 发起 HTTP Get 请求。聚合器会最先接收到这一 HTTP 请求，分情况进行处理：

* 针对 APIService 的，聚合器的 Generic Server 会截留处理。
* 针对其它内置 API 的，则交给它的 Delegation， 即主 Server 去处理。
* 针对来自聚合 Server 的客制化 API，则启动一个反向代理服务，利用它将请求转 发给该聚合 Server。

这个判断并不是通过写 if 语句实现的，聚合器通过给不同端点绑定不同响应器的方式 来达成。对于第一种情况，它的 Generic Server 已经为其注册了响应器；对于第二、三种情 况，聚合器把分发和响应逻辑都放在了 apiserver 包下一个名为 proxyHandler 的结构体中，

```go
// 代码: staging\src\k8s.io\kube-aggregator\pkg\apiserver\handler_proxy.go#L50-L68
// proxyHandler provides a http.Handler which will proxy traffic to locations
// specified by items implementing Redirector.
type proxyHandler struct {
	// localDelegate is used to satisfy local APIServices
	localDelegate http.Handler

	// proxyCurrentCertKeyContent holds the client cert used to identify this proxy. Backing APIServices use this to confirm the proxy's identity
	proxyCurrentCertKeyContent certKeyFunc
	proxyTransportDial         *transport.DialHolder

	// Endpoints based routing to map from cluster IP to routable IP
	serviceResolver ServiceResolver

	handlingInfo atomic.Value

	// reject to forward redirect response
	rejectForwardingRedirects bool

	// tracerProvider is used to wrap the proxy transport and handler with tracing
	tracerProvider tracing.TracerProvider
}

func (r *proxyHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ...
}

func (r *proxyHandler) setServiceAvailable() {
    ...
}

func (r *proxyHandler) updateAPIService(apiService *apiregistrationv1api.APIService) {
    ...
}
```

这个结构体实现了 `http.Hanlder`，可以作为端点响应器。每当有 APIService 实例被创建出来，聚合器就会根据该实例信息，创建一个 `proxyHandler` 结构体的实例，并调用其 `updateAPIService` 方法完成内部信息初始化，最后将它设置为端点“`/apis/<该 API 组>/<该 API 版本>`”的响应器。ProxyHandler 结构体在客户端请求分发过程中的作用如图所示。

![image-20251215154525096](http://images.liangning7.cn/typora/202512151545383.png)

`proxyHandle.updateAPIService()` 方法将创建反向代理服务时需要的信息组织到 `handlingInfo` 字段中，包括但不限于：代理服务用于和聚合 Server 建立互信的证书和私钥与 CA 证书、目标 Service 的 host 和 port 等信息。

`proxyHandler.ServeHTTP()` 方法首先会取出 `handlingInfo`，据此判断是不是由主 Server 和扩展 Server 负责的 API，是的话交由 `localDelegate` 字段所代表的主 Server 去处理，否则就需要交由聚合 Server 了。如果需要与聚合 Server 交互，则需先创建代理服务，这用到 handlingInfo 中的信息：类型为 `url.URL` 的地址信息、类型为 `http.RoundTriper` 的请求操作信息，它们被同请求内容一起交给反向代理服务提供者来获得代理服务。apimachinery 库实现了反向代理服务提供者，它包装了基础库 `net/http` 中 `httputil.ReverseProxy` 结构体所提供的能力。

针对本节开头的例子，当从 kubectl 发出的 describe 命令被转成 http 请求并到达聚合器时，根据目标端点的不同，请求将流转到不同的响应器去处理。
