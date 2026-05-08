+++
title = "K8s Generic Server Instance"
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
series_order = 5

+++
# Generic Server Instance

以 kube-apiserver 为例，来看 Generic Server 是如何从 Config `New` 出 Instance。最后看一下 New 方法的详情。

## Config To Instance

```go
// 代码: cmd\kube-apiserver\app\server.go#L148
// Run runs the specified APIServer.  This should never exit.
func Run(ctx context.Context, opts options.CompletedOptions) error {
	// To help debugging, immediately log version
	klog.Infof("Version: %+v", utilversion.Get())

	klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

	config, err := NewConfig(opts) 
	if err != nil {
		return err
	}
	completed, err := config.Complete() //要点①
	if err != nil {
		return err
	}
	server, err := CreateServerChain(completed) //要点②
	if err != nil {
		return err
	}

	prepared, err := server.PrepareRun()
	if err != nil {
		return err
	}

	return prepared.Run(ctx)
}
```

在要点①处，构建出了 `completedConfig`，然后在要点②处，使用该 Config 构建出实例：

```go
// 代码: cmd\kube-apiserver\app\server.go#L176
// CreateServerChain creates the apiservers connected via delegation.
func CreateServerChain(config CompletedConfig) (*aggregatorapiserver.APIAggregator, error) {
	notFoundHandler := notfoundhandler.New(config.KubeAPIs.ControlPlane.Generic.Serializer, genericapifilters.NoMuxAndDiscoveryIncompleteKey)
	apiExtensionsServer, err := config.ApiExtensions.New(genericapiserver.NewEmptyDelegateWithCustomHandler(notFoundHandler))
	if err != nil {
		return nil, err
	}
	crdAPIEnabled := config.ApiExtensions.GenericConfig.MergedResourceConfig.ResourceEnabled(apiextensionsv1.SchemeGroupVersion.WithResource("customresourcedefinitions"))

	kubeAPIServer, err := config.KubeAPIs.New(apiExtensionsServer.GenericAPIServer) //要点①
	if err != nil {
		return nil, err
	}

	// aggregator comes last in the chain
	aggregatorServer, err := controlplaneapiserver.CreateAggregatorServer(config.Aggregator, kubeAPIServer.ControlPlane.GenericAPIServer, apiExtensionsServer.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdAPIEnabled, apiVersionPriorities)
	if err != nil {
		// we don't need special handling for innerStopCh because the aggregator server doesn't create any go routines
		return nil, err
	}

	return aggregatorServer, nil
}
```

Generic Server 的实例构建是通过 `config.KubeAPIs.New` 来进行构建的：

```go
// 代码: pkg\controlplane\instance.go#L316
// New returns a new instance of Master from the given config.
// Certain config fields will be set to a default value if unset.
// Certain config fields must be specified, including:
// KubeletClientConfig
func (c CompletedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
	if reflect.DeepEqual(c.Extra.KubeletClientConfig, kubeletclient.KubeletClientConfig{}) {
		return nil, fmt.Errorf("Master.New() called with empty config.KubeletClientConfig")
	}

	cp, err := c.ControlPlane.New(controlplaneapiserver.KubeAPIServer, delegationTarget) //要点①
	if err != nil {
		return nil, err
	}
    ...
}
```

要点①出，构建出 `controlPlane` 的实例，在其中会构建出 Generic Server 的实例：

```go
// 代码: pkg\controlplane\apiserver\server.go#L95
// New returns a new instance of Master from the given config.
// Certain config fields will be set to a default value if unset.
// Certain config fields must be specified, including:
// KubeletClientConfig
func (c completedConfig) New(name string, delegationTarget genericapiserver.DelegationTarget) (*Server, error) {
	generic, err := c.Generic.New(name, delegationTarget)
	if err != nil {
		return nil, err
	}
	...
}
```

果然，这里就会通过 `c.Generic.New` 来进行构建。这样就走到了 Generic Server 的实例构建函数了。

## New 函数

> staging\src\k8s.io\apiserver\pkg\server\config.go#L766-L1006

下面让我们详细看看是如何构建的吧！

```go
// New creates a new server which logically combines the handling chain with the passed server.
// name is used to differentiate for logging. The handler chain in particular can be difficult as it starts delegating.
// delegationTarget may not be nil.
func (c completedConfig) New(name string, delegationTarget DelegationTarget) (*GenericAPIServer, error) {
	if c.Serializer == nil {
		return nil, fmt.Errorf("Genericapiserver.New() called with config.Serializer == nil")
	}
	allowedMediaTypes := defaultAllowedMediaTypes
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.CBORServingAndStorage) {
		allowedMediaTypes = append(allowedMediaTypes, runtime.ContentTypeCBOR)
	}
	for _, info := range c.Serializer.SupportedMediaTypes() {
		var ok bool
		for _, mt := range allowedMediaTypes {
			if info.MediaType == mt {
				ok = true
				break
			}
		}
		if !ok {
			return nil, fmt.Errorf("refusing to create new apiserver %q with support for media type %q (allowed media types are: %s)", name, info.MediaType, strings.Join(allowedMediaTypes, ", "))
		}
	}
	if c.LoopbackClientConfig == nil {
		return nil, fmt.Errorf("Genericapiserver.New() called with config.LoopbackClientConfig == nil")
	}
	if c.EquivalentResourceRegistry == nil {
		return nil, fmt.Errorf("Genericapiserver.New() called with config.EquivalentResourceRegistry == nil")
	}

	handlerChainBuilder := func(handler http.Handler) http.Handler {
		return c.BuildHandlerChainFunc(handler, c.Config)
	}

	var debugSocket *routes.DebugSocket
	if c.DebugSocketPath != "" {
		debugSocket = routes.NewDebugSocket(c.DebugSocketPath)
	}

	apiServerHandler := NewAPIServerHandler(name, c.Serializer, handlerChainBuilder, delegationTarget.UnprotectedHandler())

	s := &GenericAPIServer{
		discoveryAddresses:             c.DiscoveryAddresses,
		LoopbackClientConfig:           c.LoopbackClientConfig,
		legacyAPIGroupPrefixes:         c.LegacyAPIGroupPrefixes,
		admissionControl:               c.AdmissionControl,
		Serializer:                     c.Serializer,
		AuditBackend:                   c.AuditBackend,
		Authorizer:                     c.Authorization.Authorizer,
		delegationTarget:               delegationTarget,
		EquivalentResourceRegistry:     c.EquivalentResourceRegistry,
		NonLongRunningRequestWaitGroup: c.NonLongRunningRequestWaitGroup,
		WatchRequestWaitGroup:          c.WatchRequestWaitGroup,
		Handler:                        apiServerHandler,
		UnprotectedDebugSocket:         debugSocket,

		listedPathProvider: apiServerHandler,

		minRequestTimeout:                   time.Duration(c.MinRequestTimeout) * time.Second,
		ShutdownTimeout:                     c.RequestTimeout,
		ShutdownDelayDuration:               c.ShutdownDelayDuration,
		ShutdownWatchTerminationGracePeriod: c.ShutdownWatchTerminationGracePeriod,
		SecureServingInfo:                   c.SecureServing,
		ExternalAddress:                     c.ExternalAddress,

		openAPIConfig:           c.OpenAPIConfig,
		openAPIV3Config:         c.OpenAPIV3Config,
		skipOpenAPIInstallation: c.SkipOpenAPIInstallation,

		postStartHooks:         map[string]postStartHookEntry{},
		preShutdownHooks:       map[string]preShutdownHookEntry{},
		disabledPostStartHooks: c.DisabledPostStartHooks,

		healthzRegistry:  healthCheckRegistry{path: "/healthz", checks: c.HealthzChecks},
		livezRegistry:    healthCheckRegistry{path: "/livez", checks: c.LivezChecks, clock: clock.RealClock{}},
		readyzRegistry:   healthCheckRegistry{path: "/readyz", checks: c.ReadyzChecks},
		livezGracePeriod: c.LivezGracePeriod,

		DiscoveryGroupManager: discovery.NewRootAPIsHandler(c.DiscoveryAddresses, c.Serializer),

		maxRequestBodyBytes: c.MaxRequestBodyBytes,

		lifecycleSignals:       c.lifecycleSignals,
		ShutdownSendRetryAfter: c.ShutdownSendRetryAfter,

		APIServerID:           c.APIServerID,
		StorageReadinessHook:  NewStorageReadinessHook(c.StorageInitializationTimeout),
		StorageVersionManager: c.StorageVersionManager,

		EffectiveVersion:                        c.EffectiveVersion,
		EmulationForwardCompatible:              c.EmulationForwardCompatible,
		RuntimeConfigEmulationForwardCompatible: c.RuntimeConfigEmulationForwardCompatible,
		FeatureGate:                             c.FeatureGate,

		muxAndDiscoveryCompleteSignals: map[string]<-chan struct{}{},
	}

	manager := c.AggregatedDiscoveryGroupManager
	if manager == nil {
		manager = discoveryendpoint.NewResourceManager("apis")
	}
	s.AggregatedDiscoveryGroupManager = manager
	s.AggregatedLegacyDiscoveryGroupManager = discoveryendpoint.NewResourceManager("api")
	for {
		if c.JSONPatchMaxCopyBytes <= 0 {
			break
		}
		existing := atomic.LoadInt64(&jsonpatch.AccumulatedCopySizeLimit)
		if existing > 0 && existing < c.JSONPatchMaxCopyBytes {
			break
		}
		if atomic.CompareAndSwapInt64(&jsonpatch.AccumulatedCopySizeLimit, existing, c.JSONPatchMaxCopyBytes) {
			break
		}
	}

	// first add poststarthooks from delegated targets
	for k, v := range delegationTarget.PostStartHooks() {
		s.postStartHooks[k] = v
	}

	for k, v := range delegationTarget.PreShutdownHooks() {
		s.preShutdownHooks[k] = v
	}

	// add poststarthooks that were preconfigured.  Using the add method will give us an error if the same name has already been registered.
	for name, preconfiguredPostStartHook := range c.PostStartHooks {
		if err := s.AddPostStartHook(name, preconfiguredPostStartHook.hook); err != nil {
			return nil, err
		}
	}

	// register mux signals from the delegated server
	for k, v := range delegationTarget.MuxAndDiscoveryCompleteSignals() {
		if err := s.RegisterMuxAndDiscoveryCompleteSignal(k, v); err != nil {
			return nil, err
		}
	}

	genericApiServerHookName := "generic-apiserver-start-informers"
	if c.SharedInformerFactory != nil {
		if !s.isPostStartHookRegistered(genericApiServerHookName) {
			err := s.AddPostStartHook(genericApiServerHookName, func(hookContext PostStartHookContext) error {
				c.SharedInformerFactory.Start(hookContext.Done())
				return nil
			})
			if err != nil {
				return nil, err
			}
		}
		// TODO: Once we get rid of /healthz consider changing this to post-start-hook.
		err := s.AddReadyzChecks(healthz.NewInformerSyncHealthz(c.SharedInformerFactory))
		if err != nil {
			return nil, err
		}
	}

	const priorityAndFairnessConfigConsumerHookName = "priority-and-fairness-config-consumer"
	if s.isPostStartHookRegistered(priorityAndFairnessConfigConsumerHookName) {
	} else if c.FlowControl != nil {
		err := s.AddPostStartHook(priorityAndFairnessConfigConsumerHookName, func(hookContext PostStartHookContext) error {
			go c.FlowControl.Run(hookContext.Done())
			return nil
		})
		if err != nil {
			return nil, err
		}
		// TODO(yue9944882): plumb pre-shutdown-hook for request-management system?
	} else {
		klog.V(3).Infof("Not requested to run hook %s", priorityAndFairnessConfigConsumerHookName)
	}

	// Add PostStartHooks for maintaining the watermarks for the Priority-and-Fairness and the Max-in-Flight filters.
	if c.FlowControl != nil {
		const priorityAndFairnessFilterHookName = "priority-and-fairness-filter"
		if !s.isPostStartHookRegistered(priorityAndFairnessFilterHookName) {
			err := s.AddPostStartHook(priorityAndFairnessFilterHookName, func(hookContext PostStartHookContext) error {
				genericfilters.StartPriorityAndFairnessWatermarkMaintenance(hookContext.Done())
				return nil
			})
			if err != nil {
				return nil, err
			}
		}
	} else {
		const maxInFlightFilterHookName = "max-in-flight-filter"
		if !s.isPostStartHookRegistered(maxInFlightFilterHookName) {
			err := s.AddPostStartHook(maxInFlightFilterHookName, func(hookContext PostStartHookContext) error {
				genericfilters.StartMaxInFlightWatermarkMaintenance(hookContext.Done())
				return nil
			})
			if err != nil {
				return nil, err
			}
		}
	}

	// Add PostStartHook for maintenaing the object count tracker.
	if c.StorageObjectCountTracker != nil {
		const storageObjectCountTrackerHookName = "storage-object-count-tracker-hook"
		if !s.isPostStartHookRegistered(storageObjectCountTrackerHookName) {
			if err := s.AddPostStartHook(storageObjectCountTrackerHookName, func(hookContext PostStartHookContext) error {
				go c.StorageObjectCountTracker.RunUntil(hookContext.Done())
				return nil
			}); err != nil {
				return nil, err
			}
		}
	}

	for _, delegateCheck := range delegationTarget.HealthzChecks() {
		skip := false
		for _, existingCheck := range c.HealthzChecks {
			if existingCheck.Name() == delegateCheck.Name() {
				skip = true
				break
			}
		}
		if skip {
			continue
		}
		s.AddHealthChecks(delegateCheck)
	}
	s.RegisterDestroyFunc(func() {
		if err := c.Config.TracerProvider.Shutdown(context.Background()); err != nil {
			klog.Errorf("failed to shut down tracer provider: %v", err)
		}
	})

	s.listedPathProvider = routes.ListedPathProviders{s.listedPathProvider, delegationTarget}

	installAPI(name, s, c.Config)

	// use the UnprotectedHandler from the delegation target to ensure that we don't attempt to double authenticator, authorize,
	// or some other part of the filter chain in delegation cases.
	if delegationTarget.UnprotectedHandler() == nil && c.EnableIndex {
		s.Handler.NonGoRestfulMux.NotFoundHandler(routes.IndexLister{
			StatusCode:   http.StatusNotFound,
			PathProvider: s.listedPathProvider,
		})
	}

	return s, nil
}
```

### 1. 安全与合法性检查

> 必须携带序列化器、Loopback 客户端与等价资源注册表；

```go
if c.Serializer == nil {
    return nil, fmt.Errorf("Genericapiserver.New() called with config.Serializer == nil")
}
allowedMediaTypes := defaultAllowedMediaTypes
if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.CBORServingAndStorage) {
    allowedMediaTypes = append(allowedMediaTypes, runtime.ContentTypeCBOR)
}
for _, info := range c.Serializer.SupportedMediaTypes() {
    var ok bool
    for _, mt := range allowedMediaTypes {
        if info.MediaType == mt {
            ok = true
            break
        }
    }
    if !ok {
        return nil, fmt.Errorf("refusing to create new apiserver %q with support for media type %q (allowed media types are: %s)", name, info.MediaType, strings.Join(allowedMediaTypes, ", "))
    }
}
if c.LoopbackClientConfig == nil {
    return nil, fmt.Errorf("Genericapiserver.New() called with config.LoopbackClientConfig == nil")
}
if c.EquivalentResourceRegistry == nil {
    return nil, fmt.Errorf("Genericapiserver.New() called with config.EquivalentResourceRegistry == nil")
}
```

* ```go
  if c.Serializer == nil {
      return nil, fmt.Errorf("Genericapiserver.New() called with config.Serializer == nil")
  }
  ```

  `Serializer` 定义了 apiserver 支持的编解码方案：它掌握哪些媒体类型可用，以及如何把 HTTP 请求/响应中的对象与内部 `runtime.Object` 互相转换。

* ```go
  allowedMediaTypes := defaultAllowedMediaTypes
  if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.CBORServingAndStorage) {
      allowedMediaTypes = append(allowedMediaTypes, runtime.ContentTypeCBOR)
  }
  ```

  定义允许的媒体类型【`defaultAllowedMediaTypes`】：`JSON/YAML/Protobuf`

  如果特性门 `genericfeatures.CBORServingAndStorage` 被开启，则将 `runtime.ContentTypeCBOR` 附加进去。

* ```go
  for _, info := range c.Serializer.SupportedMediaTypes() {
      var ok bool
      for _, mt := range allowedMediaTypes {
          if info.MediaType == mt {
              ok = true
              break
          }
      }
      if !ok {
          return nil, fmt.Errorf("refusing to create new apiserver %q with support for media type %q (allowed media types are: %s)", name, info.MediaType, strings.Join(allowedMediaTypes, ", "))
      }
  }
  ```

  这个循环逐个检查 `Serializer` 声称支持的媒体类型是否都在前面准备好的 `allowedMediaTypes` 列表里，防止某个 GenericAPIServer 以未批准的编码格式对外提供服务，从而保证上游客户端与其他组件的兼容性。

* ```go
  if c.LoopbackClientConfig == nil {
      return nil, fmt.Errorf("Genericapiserver.New() called with config.LoopbackClientConfig == nil")
  }
  ```

  “Loopback” 指的是 apiserver 内部访问自己（自我调用）的专用客户端配置——`LoopbackClientConfig`。它通常携带内置的、具备最高权限的认证信息，用来让 server 自己去调用自身的 HTTP API。例如：

  * 注册/更新内置资源、webhook、优先级与公平性配置等控制面动作；
  * 启动和健康检查内部组件时，需要以 apiserver 身份访问 `/healthz`、`/readyz` 等端点；
  * 在 admission、aggregator 等子系统里，很多初始化步骤需要使用 loopback 客户端。

* ```go
  if c.EquivalentResourceRegistry == nil {
      return nil, fmt.Errorf("Genericapiserver.New() called with config.EquivalentResourceRegistry == nil")
  }
  ```

  `EquivalentResourceRegistry` 用于描述哪些资源在语义上等价（例如不同版本的同一资源）并共享同一个后端存储、权限或速率限制。GenericAPIServer 在处理缓存、优先级与公平性、限流等逻辑时需要这个映射表来决定“这两个 GroupResource 是否可以被视为一类”。

### 2. 构建 handler 链

```go
handlerChainBuilder := func(handler http.Handler) http.Handler {
    return c.BuildHandlerChainFunc(handler, c.Config)
}
```

* 通过 `c.BuildHandlerChainFunc(handler, c.Config)`，把认证、鉴权、审计、限流等中间件包装在最终 handler 外面。

* 这个 `BuildHandlerChainFunc` 作为字段写入 Config 中，在初始化 Cofig 的时候进行配置的。

  ```go
  // 代码: staging\src\k8s.io\apiserver\pkg\server\config.go#L396
  // NewConfig returns a Config struct with the default values
  func NewConfig(codecs serializer.CodecFactory) *Config {
  	defaultHealthChecks := []healthz.HealthChecker{healthz.PingHealthz, healthz.LogHealthz}
  	...
  
  	return &Config{
  		Serializer:                     codecs,
  		BuildHandlerChainFunc:          DefaultBuildHandlerChain,
          ...
      }
  }
  ```

### 3. Debug 入口

```go
var debugSocket *routes.DebugSocket
if c.DebugSocketPath != "" {
    debugSocket = routes.NewDebugSocket(c.DebugSocketPath)
}
```

如果配置了 DebugSocketPath，则创建 Unix socket 以提供运行时调试入口。

### 4. 生成 Server 的请求处理者

```go
apiServerHandler := NewAPIServerHandler(name, c.Serializer, handlerChainBuilder, delegationTarget.UnprotectedHandler())
```

用 name、序列化器、handler 链构建函数和委托目标的未受保护 handler 生成 API Server 的顶层 handler。

### 5. 实例化 Generic Server

```go
s := &GenericAPIServer{
    discoveryAddresses:             c.DiscoveryAddresses,
    LoopbackClientConfig:           c.LoopbackClientConfig,
    legacyAPIGroupPrefixes:         c.LegacyAPIGroupPrefixes,
    admissionControl:               c.AdmissionControl,
    Serializer:                     c.Serializer,
    AuditBackend:                   c.AuditBackend,
    Authorizer:                     c.Authorization.Authorizer,
    delegationTarget:               delegationTarget,
    EquivalentResourceRegistry:     c.EquivalentResourceRegistry,
    NonLongRunningRequestWaitGroup: c.NonLongRunningRequestWaitGroup,
    WatchRequestWaitGroup:          c.WatchRequestWaitGroup,
    Handler:                        apiServerHandler,
    UnprotectedDebugSocket:         debugSocket,

    listedPathProvider: apiServerHandler,

    minRequestTimeout:                   time.Duration(c.MinRequestTimeout) * time.Second,
    ShutdownTimeout:                     c.RequestTimeout,
    ShutdownDelayDuration:               c.ShutdownDelayDuration,
    ShutdownWatchTerminationGracePeriod: c.ShutdownWatchTerminationGracePeriod,
    SecureServingInfo:                   c.SecureServing,
    ExternalAddress:                     c.ExternalAddress,

    openAPIConfig:           c.OpenAPIConfig,
    openAPIV3Config:         c.OpenAPIV3Config,
    skipOpenAPIInstallation: c.SkipOpenAPIInstallation,

    postStartHooks:         map[string]postStartHookEntry{},
    preShutdownHooks:       map[string]preShutdownHookEntry{},
    disabledPostStartHooks: c.DisabledPostStartHooks,

    healthzRegistry:  healthCheckRegistry{path: "/healthz", checks: c.HealthzChecks},
    livezRegistry:    healthCheckRegistry{path: "/livez", checks: c.LivezChecks, clock: clock.RealClock{}},
    readyzRegistry:   healthCheckRegistry{path: "/readyz", checks: c.ReadyzChecks},
    livezGracePeriod: c.LivezGracePeriod,

    DiscoveryGroupManager: discovery.NewRootAPIsHandler(c.DiscoveryAddresses, c.Serializer),

    maxRequestBodyBytes: c.MaxRequestBodyBytes,

    lifecycleSignals:       c.lifecycleSignals,
    ShutdownSendRetryAfter: c.ShutdownSendRetryAfter,

    APIServerID:           c.APIServerID,
    StorageReadinessHook:  NewStorageReadinessHook(c.StorageInitializationTimeout),
    StorageVersionManager: c.StorageVersionManager,

    EffectiveVersion:                        c.EffectiveVersion,
    EmulationForwardCompatible:              c.EmulationForwardCompatible,
    RuntimeConfigEmulationForwardCompatible: c.RuntimeConfigEmulationForwardCompatible,
    FeatureGate:                             c.FeatureGate,

    muxAndDiscoveryCompleteSignals: map[string]<-chan struct{}{},
}
```

把 discovery 地址、认证授权、审计、OpenAPI、健康检查、超时/关机策略等配置全部注入。同时记录 delegationTarget，以便把请求链继续往下传，形成多层 apiserver 的委托链。

### 6. 后续

继承/合并 `delegationTarget` 已注册的 `post-start`、`pre-shutdown hook` 以及 `mux/discovery` 完成信号；再注册当前 server 自己需要的 hook（比如启动 informer、优先级与公平性控制等）。这些逻辑确保整条委托链的钩子顺序执行且不会丢失。

