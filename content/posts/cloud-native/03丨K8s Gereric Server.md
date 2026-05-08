+++
title = "K8s Generic Server"
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
series_order = 3

+++

# 1. Generic Server 的构建

Generic Server 是主 Server，扩展 Server，聚合器和聚合 Server 的底座，所以它的构建过程也是以上三类 Server 的构建过程的一部分，所以这部分十分重要。

聚焦 Generic Server 的创建过程，各个 Server 的部件如何被组装起来是重点。Generic Server 有几大核心能力：

1. 请求过滤机制。一个请求在抵达目标处理逻辑之前，要经过一组通用的处理过滤器，从而完成限流，登录鉴权等必要处理。
2. Server 链。两个 Generic Server 实例可以相互连接，上游 Server 处理不了的请求传递给下游继续处理，即形成 Server Chain 的能力。API Server 内部的 Server 链正是在各个子 Server 的基座 Generic Server 基础上构建的。
3. 装配 Kubernetes API。在 Generic Server的源码中看不到任何具体的 Kubernetes API， 这是其上层 Server 运行时注入进来的，**Generic Server 提供了将 API 注入 Server 并进行装配的接口方法**。

Server 的创建过程包含了请求过滤机制和 Server 链，而 API 的注入与装配较重要，将在后续小节进行详细讲解。

## 1.1 准备 Server 运行配置

获得 Server 运行配置（Config）是创建 Serer 实例的前提。Generic Server 不会独立里运行，而是作为各子 Server 的底座之用，所以它的**运行配置均来自使用它的各子 Server**。

源文件 `config.go` 中定义的 `NewConfig()` 函数是 Generic Server 运行配置的工厂函数，它会返回一个推荐配置实例。子 Server 会调用它来创建其底座 Generic Server 运行配置实例，进行信息填充然后用它创建出底座 Generic Server。填充所用信息来自多个源，包括用户命令行输入的参数、命令行参数默认值和子 Server 专有调整。信息流的大致方向为：由启动参数流转至运行选项（Option）、再由运行选项到 Config。

1. 从启动参数到 Option 的主要步骤为：首先用户输入命令行参数值，它们是利用 Cobra 定义出的标志（flag），然后这些标志被转化为 Option 结构体实例；接着该 Option 实例会用自己的 `Validate()` 方法校验自身内容。
2. 信息由 Option 流转至 Config 则**由各个子 Server 主导，中间会添加子 Server 需要的专有调整**，例如改变 Generic Server 的某些运行配置值和添加子 Server 专有运行配置项。

详情参考 [K8s Generic Server Options To Configs](/posts/cloud-native/appendix/01丨k8s-generic-server-options-to-configs)

## 1.2 创建 Server 实例

Generic Server 的创建代码位于源文件`staging/src/k8s.io/apiserver/pkg/server/config.go` 中。

无论上层的 Server 是主、扩展或其它，其底层的 Generic Server 都是通过两次方法调用的得到的：

* 首先调用 Config 结构体实例的 `Complete()` 方法得到 `CompletedConfig` 结构体实例；
* 然后调用 `CompletedConfig` 的 `New()` 方法得到 Generic Server。

这个两步走制作 Server 实例的过程很有代表性。 主 Server、扩展 Server 以及聚合 Server 实例的创建过程如出一辙：它们各自具有 Config、 CompletedConfig 结构体，对应结构体上也会有 `Complete()` 方法和 `New()` 方法，在获取各自实例时经历上述两步调用。当上层 Server 的 `Complete()` 和 `New()` 方法执行时，会调用其底座 Generic Server 的 `Complete()` 和 `New()` 方法。Generic Server 实例的创建过程如下图所示：

![image-20251122151322069](http://images.liangning7.cn/typora/202511221513229.png)

各步的具体工作内容：

1. Config 结构体实例是这个过程的源头。可由源文件 `config.go` 中定义的 `NewConfig()` 方法得到一个 Generic Server 的空 Config 实例，使用前要对其进行内容的填充。Config 中信息的源头是 API Server 的启动命令参数，信息由启动参数流转至运行选项（Option）、再由运行选项到 Config。

   从启动参数到 Option 这一过程主要步骤为：首先用户输入命令行参数值，它们是利用 Cobra 定义出的标志（flag），然后这些标志被转化为 Option 结构体实例； 接着该 Option 实例会用自己的 `Validate()` 方法校验自身内容。

   信息由 Option 流转至 Config 则由各个子 Server 主导，具体实施上，Option 的 `Apply()` 方法被用来将信息转移至 Config 结构体实例。

2. Config 的 `Complete()` 方法作用是对参数设定进行查漏补缺。在 Generic Server 这个层面，它检查并完善了 Server 的 IP 和端口设置，以及登录鉴权的参数。`Complete()` 方法最终制作了一个 `CompletedConfig` 结构体实例作为结果返回。

3. 而 `CompletedConfig` 结构体的 `New()` 方法将正式开始一个 Generic Server 实例的创建。这里通过讲解 Server 核心能力的构造过程来详解这个方法。`New()` 方法签名部分代码如下所示：

   ```go
   // 代码: staging\src\k8s.io\apiserver\pkg\server\config.go
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
   
       //要点①
   	handlerChainBuilder := func(handler http.Handler) http.Handler {
   		return c.BuildHandlerChainFunc(handler, c.Config)
   	}
   
   	var debugSocket *routes.DebugSocket
   	if c.DebugSocketPath != "" {
   		debugSocket = routes.NewDebugSocket(c.DebugSocketPath)
   	}
   
       //要点②
   	apiServerHandler := NewAPIServerHandler(name, c.Serializer, handlerChainBuilder, delegationTarget.UnprotectedHandler())
       ...
   }
   ```

   `New()` 方法将主要完成两个任务：首先，制作请求处理链。包含了添加请求过滤器和构造 Server 链。然后，设置 Server 的启动后和关闭前的钩子函数，在 Server 启动后和关闭前， 这些钩子函数将被逐个执行。

> 详情参考 [02丨K8s Generic Server Instance](/posts/cloud-native/appendix/02丨K8s-Generic-Server-Instance)

## 1.3 构造请求处理链

关注 `config.go` 的要点①和要点②，

1. 要点①制作了一个处理链 Builder，是一个名为handerChainBuilder 的函数，这个 Builder 实际作用是为处理链加上各种请求过滤器；
2. 要点②处利用这个 Builder 作为参数之一调用方法 `NewAPIServerHandler()`，生成了 apiServerHandler 这一变量，它将成为 Server 的请求处理者。

### 1.3.1 请求过滤机制

Kubernetes Generic Server 提供了一组请求过滤器，用于把请求交给处理器之前做预处理工作。请求到达 Server 后立即进入过滤链，这一过程甚至**早于**请求中所包含的 Kubernetes API **实例被解码为 Go 结构体实例**，因为请求过滤器不需要理解业务逻辑。下面通过分析代码讲解这条过滤器链的构建过程。

通过 `config.go` 代码可以看到，`handerChainBuilder()` 函数内部依靠 `CompletedConfig` 的 `BuildHandlerChainFunc` 字段（类型为函数）去给 `http.Handler` 类型的入参加过滤器。追溯类型为 CompletedConfig 的变量 c 的出处，最终可以找到 BuildHandlerChainFunc 的赋予地： 位于上述 config.go 文件的 `NewConfig()` 方法，代码如下所示：

```go
// 代码  staging/src/k8s.io/apiserver/pkg/server/config.go
	return &Config{
		Serializer:                     codecs,
		BuildHandlerChainFunc:          DefaultBuildHandlerChain, //要点①
		NonLongRunningRequestWaitGroup: new(utilwaitgroup.SafeWaitGroup),
		WatchRequestWaitGroup:          &utilwaitgroup.RateLimitedSafeWaitGroup{},
		LegacyAPIGroupPrefixes:         sets.NewString(DefaultLegacyAPIPrefix),
		DisabledPostStartHooks:         sets.NewString(),
		PostStartHooks:                 map[string]PostStartHookConfigEntry{},
		HealthzChecks:                  append([]healthz.HealthChecker{}, defaultHealthChecks...),
		ReadyzChecks:                   append([]healthz.HealthChecker{}, defaultHealthChecks...),
		LivezChecks:                    append([]healthz.HealthChecker{}, defaultHealthChecks...),
		EnableIndex:                    true,
		EnableDiscovery:                true,
		EnableProfiling:                true,
		DebugSocketPath:                "",
		EnableMetrics:                  true,
		MaxRequestsInFlight:            400,
		MaxMutatingRequestsInFlight:    200,
		RequestTimeout:                 time.Duration(60) * time.Second,
		MinRequestTimeout:              1800,
		StorageInitializationTimeout:   time.Minute,
		LivezGracePeriod:               time.Duration(0),
		ShutdownDelayDuration:          time.Duration(0),
		// 1.5MB is the default client request size in bytes
		// the etcd server should accept. See
		// https://github.com/etcd-io/etcd/blob/release-3.4/embed/config.go#L56.
		// A request body might be encoded in json, and is converted to
		// proto when persisted in etcd, so we allow 2x as the largest size
		// increase the "copy" operations in a json patch may cause.
		JSONPatchMaxCopyBytes: int64(3 * 1024 * 1024),
		// 1.5MB is the recommended client request size in byte
		// the etcd server should accept. See
		// https://github.com/etcd-io/etcd/blob/release-3.4/embed/config.go#L56.
		// A request body might be encoded in json, and is converted to
		// proto when persisted in etcd, so we allow 2x as the largest request
		// body size to be accepted and decoded in a write request.
		// If this constant is changed, DefaultMaxRequestSizeBytes in k8s.io/apiserver/pkg/cel/limits.go
		// should be changed to reflect the new value, if the two haven't
		// been wired together already somehow.
		MaxRequestBodyBytes: int64(3 * 1024 * 1024),

		// Default to treating watch as a long-running operation
		// Generic API servers have no inherent long-running subresources
		LongRunningFunc:                     genericfilters.BasicLongRunningRequestCheck(sets.NewString("watch"), sets.NewString()),
		lifecycleSignals:                    lifecycleSignals,
		StorageObjectCountTracker:           flowcontrolrequest.NewStorageObjectCountTracker(),
		ShutdownWatchTerminationGracePeriod: time.Duration(0),

		APIServerID:           id,
		StorageVersionManager: storageversion.NewDefaultManager(),
		TracerProvider:        tracing.NewNoopTracerProvider(),
	}
```

由上述代码要点①可见， Generic Server 默认会用自身的 `DefaultBuildHandlerChain()` 方法作为过滤器添加函数，将一组过滤器加入到 Server 的请求处理器上。在构建上层 Server （例如聚合器）时，可以通过改变 Config 的该字段值来改变这一行为，不过实践中 Generic Server 所提供的所有请求过滤器都被核心 API Server 保留使用了，它们足够底层和通用。

方法 `DefaultBuildHandlerChain()` 中为请求处理添加的过滤器见下表。

| 序号 | 添加过滤器                                            | 作用                                                         |
| ---- | ----------------------------------------------------- | ------------------------------------------------------------ |
| 1    | `genericapifilters.WithAuthorization`                 | 权限                                                         |
| 2    | `genericfilters.WithMaxInFlightLimit`                 | 流量控制                                                     |
| 3    | `genericapifilters.WithImpersonation`                 | 冒名请求                                                     |
| 4    | `genericapifilters.WithAudit`                         | 审计                                                         |
| 5    | `genericapifilters.WithFailedAuthenticationAudit`     | 审计 - 登录失败                                              |
| 6    | `genericapifilters.WithAuthentication`                | 登录                                                         |
| 7    | `genericfilters.WithCORS`                             | 跨域资源共享                                                 |
| 8    | `genericfilters.WithTimeoutForNonLongRunningRequests` | 超时时对客户端的响应                                         |
| 9    | `genericapifilters.WithRequestDeadline`               | 设置处理时限                                                 |
| 10   | `genericfilters.WithWaitGroup`                        | 将请求加入 wait group                                        |
| 11   | `genericfilters.WithWatchTerminationDuringShutdown`   | 设置了优雅关闭时长时，在系统关闭期间观察系统状态             |
| 12   | `genericfilters.WithProbabilisticGoaway`              | HTTP2 模式下适时发送 GOAWAY 请求                             |
| 13   | `genericapifilters.WithWarningRecorder`               | 向 Header 中添加 Warning                                     |
| 14   | `genericapifilters.WithCacheControl`                  | 设置 cache-control 请求头                                    |
| 15   | `genericfilters.WithHSTS`                             | 启用 HTTP 严格传输安全                                       |
| 16   | `genericfilters.WithRetryAfter`                       | 关机信号发出并过了延迟期，拒绝链接                           |
| 17   | `genericfilters.WithHTTPLogging`                      | 记录请求到日志中                                             |
| 18   | `genericapifilters.WithTracing`                       | 为支持在分布式 API Server 中追踪请求而设置                   |
| 19   | `genericapifilters.WithLatencyTrackers`               | 用于记录请求在 API Server 各个组件间的延迟                   |
| 20   | `genericapifilters.WithRequestInfo`                   | 放一个 RequestInfo 实例到 context 中                         |
| 21   | `genericapifilters.WithRequestReceivedTimestamp`      | 用于添加请求到达 API Server 的时间戳                         |
| 22   | `genericapifilters.WithMuxAndDiscoveryComplete`       | Server 还没完全启动就收到请求，在 context 中放入特殊标志从而返回特定 HTTP Status code |
| 23   | `genericfilters.WithPanicRecovery`                    | 异常发生时记录日志并试图恢复。但 `http.ErrAbortHandler` 不可恢复 |
| 24   | `genericapifilters.WithAuditInit`                     | 创建 Audit context                                           |

一个 HTTP 请求最终会交由处理器去响应，处理器类型需要实现 `http.Handler` 接口，该接口定义唯一方法，其签名为：

```go
ServeHTTP(w HttpRequestWriter, r *Request)
```

下面讲解一下上面 `WithXXX` 方法是如何把过滤器加到处理器响应之前的。通过 `WithAuthorization()` 的源码来一探究竟：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\endpoints\filters\authorization.go
func withAuthorization(handler http.Handler, a authorizer.Authorizer, s runtime.NegotiatedSerializer, metrics recordAuthorizationMetricsFunc) http.Handler {
	if a == nil {
		klog.Warning("Authorization is disabled")
		return handler
	}
	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) { //要点①
		ctx := req.Context()
		authorizationStart := time.Now()

		attributes, err := GetAuthorizerAttributes(ctx)
		if err != nil {
			responsewriters.InternalError(w, req, err)
			return
		}
		authorized, reason, err := a.Authorize(ctx, attributes)

		authorizationFinish := time.Now()
		request.TrackAuthorizationLatency(ctx, authorizationFinish.Sub(authorizationStart))
		defer func() {
			metrics(ctx, authorized, err, authorizationStart, authorizationFinish)
		}()

		// an authorizer like RBAC could encounter evaluation errors and still allow the request, so authorizer decision is checked before error here.
		if authorized == authorizer.DecisionAllow { //要点②
			audit.AddAuditAnnotations(ctx,
				decisionAnnotationKey, decisionAllow,
				reasonAnnotationKey, reason)
			handler.ServeHTTP(w, req)
			return
		}
		if err != nil {
			audit.AddAuditAnnotation(ctx, reasonAnnotationKey, reasonError)
			responsewriters.InternalError(w, req, err)
			return
		}

		klog.V(4).InfoS("Forbidden", "URI", req.RequestURI, "reason", reason)
		audit.AddAuditAnnotations(ctx,
			decisionAnnotationKey, decisionForbid,
			reasonAnnotationKey, reason)
		responsewriters.Forbidden(ctx, attributes, w, req, reason, s)
	})
}
```

`WithXXX()` 方法均会以原始请求处理器对象作为入参，`WithAuthorization()` 也一样，该参数在上述代码中名为 `handler`。所谓加鉴权过滤器，就是设法确保在 `handler` 的 `ServeHTTP()` 方法被调用前，先检查请求是否能通过权限验证，失败的话马上结束请求过程；而只有通过了才会把请求交给原始 handler，即调用它的 `ServeHTTP()` 去处理请求。

我们来看一下这个 `WithAuthorization` 的入参：

* `handler http.Handler`：鉴权通过后要继续执行的下游处理链（业务 handler）。该函数会在外层包一层鉴权逻辑，最后仍然调回这个 handler。
* `a authorizer.Authorizer`：具体的鉴权器，实现 `Authorize` 方法（比如 RBAC、Webhook 等），用来判定当前请求是否允许。
* `s runtime.NegotiatedSerializer`：序列化器，用于在拒绝请求时构造标准化的响应（比如 `responsewriters.Forbidden` 会用它生成合适的输出格式）。
* `metrics recordAuthorizationMetricsFunc`：统计/上报鉴权耗时与结果的回调，函数会在每次请求结束时通过 `defer` 调用它，以便记录指标或日志。

`WithAuthorization()` 返回了一个类型为 `http.HandlerFunc` 的实例，这个实例由要点①处的**匿名函数通过类型转换得来**，为何一定要转换为该类型呢？类型 `http.HandlerFunc` 的作用是把名称任意但形式参数为“`(w ResponseWriter, r *Request)` ”的方法“重命名”为 `ServeHTTP()` 方法，并保持形参不变，可见，经过这样的类型转换返回的对象将符合 `http.Handler` 接口，可以作为请求处理器，这意味着 `WithAuthorization()` 的返回结果可以作为下一个 `WithXXX()` 方法的 handler 型参的实参，这确保过滤链条的生成技术上可行。综上所述，`WithAuthorization()` 方法接收一个 `http.Handler` 实例，对它进行包装，最后返回包装后的结果，结果类型同样为 `http.Handler`。

上述代码要点①处的匿名函数就是包装结果，它会在原始请求处理器前进行鉴权，通过后将请求交给原始处理器。要点②处代码进行了校验权限，如果验证通过，把请求交给原始 handler；而如果验证失败，直接返回错误代码给客户端。

以上这种“包装”方法利用了设计模式中的装饰器模式。当所有这些过滤器都被装饰到原始处理器之上后，将得到一个结构如下图所示、型如洋葱的新请求处理器。

![image-20251122215835725](http://images.liangning7.cn/typora/202511222158853.png)

将焦点切回 `CompletedConfig` 的 `New()` 方法，以上看到它构造了 `handerChainBuilder` 变量，后续当以原始请求处理器为参数去调用这个 Builder 时，原始处理器会被包裹一个个请求过滤器。而这个 Builder 何时被调用呢？即 `NewAPIServerHandler()` 内完成。下面来剖析其详细的代码逻辑。

### 1.3.2 构造 Server 链

`New()` 方法的最终会为 Server 制造出一个 HTTP 请求处理器，这个处理器需要挂有上述准备的过滤器，还要能把自身无法处理的请求，转交请求委派处理器——也就是其入参 `delegationTarget`。

在核心 API Server 的构建过程中，该入参的实参是当前子 Server 的下游子 Server。正是请求的传递处理使得所有子 Server 逻辑上形成一条链，即 Server链。 它实际上是一条请求处理链，请求从一端流向另一端，直到找到可处理的 Server。

Server 链是在对 `NewAPIServerHandler()` 方法的调用中完成的，该方法是理解 Server 链的关键，将分步讲解。其代码如下所示：

```go
// 代码: staging/src/k8s.io/apiserver/pkg/server/handler.go
func NewAPIServerHandler(name string, s runtime.NegotiatedSerializer, handlerChainBuilder HandlerChainBuilderFn, notFoundHandler http.Handler) *APIServerHandler {
	nonGoRestfulMux := mux.NewPathRecorderMux(name)
	if notFoundHandler != nil {
		nonGoRestfulMux.NotFoundHandler(notFoundHandler) //要点①
	}

	gorestfulContainer := restful.NewContainer() //要点②
	gorestfulContainer.Router(restful.CurlyRouter{}) // e.g. for proxy/{kind}/{name}/{*}
	gorestfulContainer.RecoverHandler(func(panicReason interface{}, httpWriter http.ResponseWriter) {
		logStackOnRecover(s, panicReason, httpWriter)
	})
	gorestfulContainer.ServiceErrorHandler(func(serviceErr restful.ServiceError, request *restful.Request, response *restful.Response) {
		serviceErrorHandler(s, serviceErr, request, response)
	})

	director := director{ //要点③
		name:               name,
		goRestfulContainer: gorestfulContainer,
		nonGoRestfulMux:    nonGoRestfulMux,
	}

	return &APIServerHandler{
		FullHandlerChain:   handlerChainBuilder(director),
		GoRestfulContainer: gorestfulContainer,
		NonGoRestfulMux:    nonGoRestfulMux,
		Director:           director,
	}
}
```

1. **调用 `NewAPIServerHandler()` 函数使用的实际参数**

   `New()` 中对 `NewAPIServerHandler()` 函数调用时，最后一个形参 notFoundHandler 的实参是 `delegationTarget.UnprotectedHandler`：

   * Generic Server 基座结构体的 delegationTarget 字段称为请求委派处理器，代表当前 Server 无法处理一个请求时该由哪个处理器去接替处理。一个 API Server 子 Server 在构造自己的底座 Generic Server 时会予以指定，拿核心 API Server 来说，主 Server 的下一子 Server 为扩展 Server，扩展 Server 的下一个是 NotFound Server；
   * 而 delegationTraget 的 UnprotectedHandler 字段代表未加过滤两条的请求处理器。

2. **`NewAPIServerHandler()` 构造 Server 链**

   `NewAPIServerHandler()` 方法的形参 notFoundHandler 被交给 `nonGoRestfulMux` 变量。就如其名字所揭示的，该变量代表一个**请求处理分发器**，会被作为当前 Server 请求分发器的一部分。当前 Server 不含该请求的处理器时，请求会被交由 `nonGoRestfulMux` 去分发，最终转至 notFoundHandler。通过对 HTTP 请求的接力处理，各 个 Server 形成了链。

3. **构造 Kubernetes API 端点的请求分发器**

   要点②处 `NewAPIServerHandler()` 构造了一个 go-restful 中的 Container，名为 `gorestfulContainer`，用于为 Kubernetes API 对外暴露 RESTful 服务。目前这个 Container 还是空的，没有任何服务，后续会详细讲解内置 Kubernetes API 是如何被装载进去形成 go-restful 中的 WebService 的；除了 go-restful 框架内服务，Server 也会在 go-restful 体系之外提供服务，这是由之前构造的 nonGoRestfulMux 来提供。

   gorestfulContainer 和 nonGoRestfulMux 是两个请求分发器，还需要一个分发器去在它们之间进行请求分发才行。这就是代码中要点③处 director 变量的作用：gorestfulContainer 和 nonGoRestfulMux 被放入 director 结构体实例中，director 会判断一个请求到底该由谁来处理并将请求交给它。director 是一个非常重要的变量，它是不包含过滤器的 HTTP 请求处理器，所以有些场合下也被称为 UnprotectedHandler, 上文提及的 `delegationTarget.UnprotectedHandler`，实际上就是 Server 链上下一个 Server 的 director。

### 1.3.3 完成请求处理链的构建

NewAPIServerHandler 的最终返回值是一个 APIServerHandler 结构体实例，它就是 Generic Server 的请求处理链。该实例有几个重要的属性：

1. `FullHandlerChain`：这个属性的值是以 director 为实参，调用前序所制作的 `handlerChainBuilder()` 函数来获得的。director 之所以有资格作为该方法调用的实参，是因为该结构体也实现了 `http.Hander` 接口，是一个合法的 Http 请求处理器。前面已经分析过 handlerChainBuilder 内部逻辑，它将为 director 添加过滤链。
2. `GoRestfulContainer`：就是刚刚讲过的变量 gorestfulContainer，它是一个 RESTful Container，也就是请求转发器，也是一个合法的请求处理器。Kubernetes API 将被注册进 GoRestfulContainer，形成其 Web Service。同时也包含 logs 和 OpenID 相关的端点。
3. `NonGoRestfulMux`：值为刚刚讲过的 `nonGoRestfulMux`，负责分发非 go-restful 负责处理的请求。
4. `Director`：值为前面制作的 director 变量。

APIServerHandler 结构体实例就是当前 Server 对 HTTP 请求的处理器，准确地说它是一个请求分发器：虽然它实现了 http.Handler 接口，但并**不处理请求**而是**分发**给FullHandlerChain（最终交给 GoRestfulContainer）和 NonGoRestfulMux（其中一条路径是转交请求委派处理器）去处理。被返回的 APIServerHandler 结构体实例在 `New()` 方法中被存入变量 apiServerHandler，在 `New()` 方法制作的 Generic Server 实例的后续步骤中，apiServerHandler 被赋予基座结构体的 handler 字段，被用于响应 HTTP 请求。

## 1.4 添加启动和关闭钩子函数

大型服务器的启动和关闭是严肃和复杂的过程，需要遵循一定顺序并要做好状态检验，步步为营。Generic Server 是 Kubernetes 的一个通用服务器，被作为子 Server 的底座，它不可能涵盖所有上层服务器在启动后和关闭前这两个阶段的所有考量，这时就需要提供钩子机制，让上层服务器将启动和关闭逻辑注入到 Generic Server 的启动与关闭流程中。

```go
manager := c.AggregatedDiscoveryGroupManager
if manager == nil {
    manager = discoveryendpoint.NewResourceManager("apis")
}
s.AggregatedDiscoveryGroupManager = manager
s.AggregatedLegacyDiscoveryGroupManager = discoveryendpoint.NewResourceManager("api")
```

* 确保新建的 `GenericAPIServer` 在 `/apis` 与 `/api` 这两个 discovery 端点上都有可用的资源管理器：若调用方未提供 `AggregatedDiscoveryGroupManager`，就创建默认的 `ResourceManager("apis")`，同时始终初始化 legacy `/api` 的管理器。

```go
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
```

* 用 CAS 循环把全局的 JSON Patch 拷贝量限制（`jsonpatch.AccumulatedCopySizeLimit`）调整到当前配置指定的 `c.JSONPatchMaxCopyBytes`
* 防止 JSON Patch 操作中的 “copy” 指令导致过大的内存消耗，同时确保多个 apiserver 实例共享一致的上限。

在构建 Generic Server 的 `New()` 方法后半部分，来自**三个来源的钩子函数**被放入两组中，在两个不同的时点运行。三个来源包括：

* Server 链中的下一个 Server；
* Server 运行配置信息 （CompletedConfig）；
* Generic Server 自身定义。

两个运行时点指：

```go
// first add poststarthooks from delegated targets
for k, v := range delegationTarget.PostStartHooks() {
    s.postStartHooks[k] = v
}

for k, v := range delegationTarget.PreShutdownHooks() {
    s.preShutdownHooks[k] = v
}
```

* 服务器启动后运行 PostStartHooks；
* 服务器关闭前运行 PreShutdownHooks。
* 把 **委托链（delegationTarget）里已注册的生命周期钩子直接继承到当前 `GenericAPIServer`** 

Generic Server 的 PostStartHooks 包含：

* Server 链中下一个 Server 的 PostStartHooks；
* Server 运行配置信息中定义的 PostStartHooks；
* 自定的 generic-apiserver-start-informers；
* 自定义的 priority-and-fairness-config-consumer；
* 自定义的 priority-and-fairness-filter；
* 自定义的 max-in-flight-filter；
* 自定义的 storage-object-count-tracker-hook。

而 PreShutdownHooks 包含 Server 链中下一个 Server 所具有的 PreShutdownHooks。

至此，`CompletedConfig` 的 `New()` 方法制作并返回了一个 GenericeAPIServer 结构体实例，待其启动方法被调用后，它将最终将支撑起整个 API Server。

# 2. Generic Server 的启动

启动以上得到的 Generic Server 实例分两个阶段完成：准备阶段和启动阶段。

## 2.1 启动准备阶段

准备阶段做一些准备工作，例如必要的参数调整。Generic Server 的准备阶段工作是由 GenericAPIServer 结构体的 `PrepareRun()` 方法实现的，它的实现包含如下内容：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L437
// PrepareRun does post API installation setup steps. It calls recursively the same function of the delegates.
func (s *GenericAPIServer) PrepareRun() preparedGenericAPIServer {
	s.delegationTarget.PrepareRun()

	if s.openAPIConfig != nil && !s.skipOpenAPIInstallation {
		s.OpenAPIVersionedService, s.StaticOpenAPISpec = routes.OpenAPI{
			Config: s.openAPIConfig,
		}.InstallV2(s.Handler.GoRestfulContainer, s.Handler.NonGoRestfulMux)
	}

	if s.openAPIV3Config != nil && !s.skipOpenAPIInstallation {
		s.OpenAPIV3VersionedService = routes.OpenAPI{
			V3Config: s.openAPIV3Config,
		}.InstallV3(s.Handler.GoRestfulContainer, s.Handler.NonGoRestfulMux)
	}

	s.installHealthz()
	s.installLivez()

	// as soon as shutdown is initiated, readiness should start failing
	readinessStopCh := s.lifecycleSignals.ShutdownInitiated.Signaled()
	err := s.addReadyzShutdownCheck(readinessStopCh)
	if err != nil {
		klog.Errorf("Failed to install readyz shutdown check %s", err)
	}
	s.installReadyz()

	if utilfeature.DefaultFeatureGate.Enabled(zpagesfeatures.ComponentStatusz) {
		statusz.Install(s.Handler.NonGoRestfulMux, "apiserver", statusz.NewRegistry(s.EffectiveVersion, statusz.WithListedPaths(s.ListedPaths())))
	}

	return preparedGenericAPIServer{s}
}
```

1. 触发它的请求委派处理器的 `PrepareRun()`。在 API Server 的实现中，请求委派处理器也是一个 Generic Server；
2. 安装 OpenAPI 的端点；
3. 安装健康检测端点、Server 运转检测端点和 Server 就续检测端点。这三个端点同样由 Server 实例中 handler 字段代表的请求处理器处理——精确地说是该处理器中的 NonGoRestfulMux 处理器响应的。

最终 `PrepareRun()` 方法返回一个类型为 `preparedGenericAPIServer` 的新 Server 实例。该类型是小写字母开头，所以只包内可见外部不可见，它提供了一个 `Run()` 方法用于进行第二阶段：Server 的启动，接下来展开介绍 `preparedGenericAPIServer.Run()` 方法逻辑。

## 2.2 启动

像 Kubernetes API Server 这样的 Web 应用启动，绝不单单是在目标端口上开启监听这么简单。如果在开启过程中就接收到了客户端请求怎么处理？服务器证书在哪里，怎么配置给服务器？此外，虽然是在启动 Server，但 `preparedGenericAPIServer.Run()` 方法的内部实现的一大部分是在安排 Server 停机时的扫尾工作。Go 内建的 http.Server 所提供的关机钩子机制不完善，不给开发者优雅善后的机会，所以需要需要自行安排。`Run()` 方法内部做了这么几件事情：

1. 与请求过滤器配合，拒绝 Server 就续前到来的请求，返回错误代码 503 给客户端，而不是 404。
2. 安排服务器停机时善后事项。
3. 配置并启动 http 服务。把技术参数应用到服务器并启动。

> 详情参考[K8s Generic Server Start](/posts/cloud-native/appendix/03丨K8s-Generic-Server-Start)

### 2.2.1 配置并启动 HTTP 服务

如果考虑安全证书的使用和对 HTTP2 的支持，Generic Server 技术上还是有些复杂性的，http 包已提供便捷的工具构造一个 Web 服务器，那么启动时对服务器进行配置就成为关键。 HTTP 服务的启动过程如下图所示，图中 `tlsconfig.go` 和 `secure_serving.go` 是源文件， `http.Server` 是基础 Go 包，其他三个对象是类型，不难通过在工程内搜索定位它们。

![image-20251122225334441](http://images.liangning7.cn/typora/202511222253598.png)

Generic Server HTTP 服务的启动是 `preparedGenericAPIServer.Run()` 方法中如下语句触发的：

```go
// 代码: staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go
	stoppedCh, listenerStoppedCh, err := s.NonBlockingRunWithContext(stopHTTPServerCtx, shutdownTimeout)
	if err != nil {
		return err
	}
```

这里变量 s 的类型是 `preparedGenericAPIServer`，它的 `NonBlockingRun()` 方法包装了 HTTP 服务的启动逻辑，这部分比较复杂所以用单独方法把这层逻辑包起来，切割出来增加了代码的可读性。名称以 NonBlock 为前缀，标识所启动的 HTTP 服务被跑在一个单独协程中，对 `NonBlockingRun` 的调用会立刻得到返回而不会阻塞当前进程。值的注意的是，Server 启动后运行的钩子函数组 `PostStartHooks` 也是在这个方法中被触发运行的。下面展开进行讲解。

1. **TLS 证书的处理**

   Server 是由结构体 `SecureServingInfo` 创建并启动起来的，证书相关信息也保存在该结构体上的，具体来说有三个证书相关的字段，分别是：

   * `Cert`：为 HTTPS 所准备的服务器证书和私钥，是建立 HTTPS 连接时发给客户端的证书。当 API Server 启动时，可以通过参数 `tls-cert-file` 和 `tls-private-key` 指定证书文件和私钥文件的路径。

   * `SNICerts`：作用和以上 Cert 字段包含的证书一样，不过适用于单个 IP 部署了多个 Web 服务的场景。一个请求到达服务器后，服务器需要决定用哪个 Web 服务的证书进行交互，决定的依据就是 SNICerts，它是个域名和证书的映射表。当 API Server 启动时，可以通过参数 `tls-sni-cert-key` 来给定一个文本文件，该文件包含多个域名、证书和私钥的三元组， 例如该文件的一行可以是如下内容：

     ```text
     foo.crt, foo.key: *.foo.com, foo.com
     ```

   * `ClientCA`：API Server 和它的客户端之间通过 mTLS 进行交互，这个过程用一种不严谨但易理解的方式描述为：除了 HTTPS 中的客户端对服务器的验证过程外，还附加一个反向的服务器对客户端的验证。在 HTTPS 的握手过程中，客户端需要能够验证服务器发来的证书合法性从而认证服务器；那么在 mTLS 过程中，服务器也要能够认证各个客户端的证书，这就需要服务器具有客户端证书签发机构的证书，这存放在 ClientCA 中。在 API Server 启动过程中，参数 `client-ca-file` 用 来指定从哪里读取到这些证书。

   一个技术细节：`http.Server` 能消费的证书信息需要通过 `crypto` 库的 `tls` 包所提供的 Config 结构体实例提供，而以上三份证书信息包含在 `SecureServingInfo` 结构体中，需要把它们再包装，合并为一个 Config 实例交给 Server。

   如果每个到来的连接请求都需要去读取这些文件、进行必要的格式转换、进而构造 Config 实例的话，系统效率定会大大降低，所以在 Server 启动时会根据以上证书信息构造好一个 Config 实例，之后 Server 便可直接从这个实例中获取信息。看起来很美好，但为提高安全级别，API Server 中证书需要定时刷新，每次刷新都需要更新 Server 所使用的 Config 实例，这就有些繁琐了，如何破解？答案要到源码中寻找。`SecureServingInfo` 中的 `tlsconfig()` 方法集中负责证书的配置，它将揭示应对方案。

   `tlsconfig()` 方法的实现中涉及两个相互协作的控制器，它们由两个 Go 基座结构体实现：

   * `staging/k8s.io/apiserver/pkg/server/dynamiccertificates/dynamic_cafile_content.go` 中的 `DynamicFileCAContent` 

     结构体 DynamicFileCAContent 实现的控制器可以监控一个证书文件，一旦文件发生变化就会向这个控制器的处理队列中加一条记录。在它的下一个控制循环中，所有当前控制器的观察者（Listener）就会被通知变化的发生。DynamicFileCAContent 保有一个观察者队列，实际运行时，这个队列内容只有一个 DynamicServingCertificateController 结构体的实例。 所谓通知观察者，就是向观察者控制器的工作队列中插入一条记录，供它们的控制循环去消 费。

   * `staging/k8s.io/apiserver/pkg/server/dynamiccertificates/tlsconfig.go` 中 定义的 `DynamicServingCertificateController`。

     结构体 DynamicServingCertificateController 实现的控制器负责更新 Generic Server 所使用的 Config 实例，如果它的控制器队列中出现条目，就代表有证书文件被更新，需要 针对新证书重新生成 Config 实例。

   方法 `tlsconfig()` 构造了 4 个控制器实例，其中三个是 DynamicFileCAContent 控制器实例， 分别对应 `SecureServingInfo` 结构体的 `Cert`，`SNICerts` 和 `ClientCA` 字段，通过这三个控制器 实例去监控三类证书的变化；还有一个 `DynamicServingCertificateController` 控制器实例，负 责在证书变动时更新 Server 可用的证书信息。它们之间的协作如图所示：

   ![image-20251123022644748](http://images.liangning7.cn/typora/202511230226951.png)

   控制器模式不仅被用来构建 Kubernetes 的资源变更监控，也被用在如上的证书更新监控。

2. 关于 HTTP2 的设置

   HTTP2 的实现细节非必要不必了解，但需要知道在 Go 中让一个 Web Server 支持 HTTP2 只需对 `http.Server` 做额外配置就好。在 Generic Server 中，如果用户启用了 HTTP2 服务，则相关配置就会加到 Server 上，这是在 `SecureServingInfo` 的 `Serve()` 方法中完成的，代码如下所示：

   ```go
   	// 代码: staging\src\k8s.io\apiserver\pkg\server\secure_serving.go
   	if !s.DisableHTTP2 {
   		// At least 99% of serialized resources in surveyed clusters were smaller than 256kb.
   		// This should be big enough to accommodate most API POST requests in a single frame,
   		// and small enough to allow a per connection buffer of this size multiplied by `MaxConcurrentStreams`.
   		const resourceBody99Percentile = 256 * 1024
   
   		http2Options := &http2.Server{
   			IdleTimeout: 90 * time.Second, // matches http.DefaultTransport keep-alive timeout
   			// shrink the per-stream buffer and max framesize from the 1MB default while still accommodating most API POST requests in a single frame
   			MaxUploadBufferPerStream: resourceBody99Percentile,
   			MaxReadFrameSize:         resourceBody99Percentile,
   		}
   
   		// use the overridden concurrent streams setting or make the default of 250 explicit so we can size MaxUploadBufferPerConnection appropriately
   		if s.HTTP2MaxStreamsPerConnection > 0 {
   			http2Options.MaxConcurrentStreams = uint32(s.HTTP2MaxStreamsPerConnection)
   		} else {
   			// match http2.initialMaxConcurrentStreams used by clients
   			// this makes it so that a malicious client can only open 400 streams before we forcibly close the connection
   			// https://github.com/golang/net/commit/b225e7ca6dde1ef5a5ae5ce922861bda011cfabd
   			http2Options.MaxConcurrentStreams = 100
   		}
   
   		// increase the connection buffer size from the 1MB default to handle the specified number of concurrent streams
   		http2Options.MaxUploadBufferPerConnection = http2Options.MaxUploadBufferPerStream * int32(http2Options.MaxConcurrentStreams)
   		// apply settings to the server
   		if err := http2.ConfigureServer(secureServer, http2Options); err != nil {
   			return nil, nil, fmt.Errorf("error configuring http2: %v", err)
   		}
   	}
   ```

3. 非阻塞运行 HTTP 服务

   在启动时序图中我们看到 `sercure_serving.go` 中有个方法 `RunServer()` 被调用，这是最终启动 HTTP 服务的地方。代码如下所示：

   ```go
   // 代码: staging/k8s.io/apiserver/pkg/server/secure_serving.go
   func RunServer(
   	server *http.Server,
   	ln net.Listener,
   	shutDownTimeout time.Duration,
   	stopCh <-chan struct{},
   ) (<-chan struct{}, <-chan struct{}, error) {
   	if ln == nil {
   		return nil, nil, fmt.Errorf("listener must not be nil")
   	}
   
   	// Shutdown server gracefully.
   	serverShutdownCh, listenerStoppedCh := make(chan struct{}), make(chan struct{})
   	go func() {
   		defer close(serverShutdownCh)
   		<-stopCh
   		ctx, cancel := context.WithTimeout(context.Background(), shutDownTimeout)
   		defer cancel()
   		err := server.Shutdown(ctx)
   		if err != nil {
   			klog.Errorf("Failed to shutdown server: %v", err)
   		}
   	}()
   
   	go func() {
   		defer utilruntime.HandleCrash()
   		defer close(listenerStoppedCh)
   
   		var listener net.Listener
   		listener = tcpKeepAliveListener{ln}
   		if server.TLSConfig != nil {
   			listener = tls.NewListener(listener, server.TLSConfig)
   		}
   
   		err := server.Serve(listener) //要点①
   
   		msg := fmt.Sprintf("Stopped listening on %s", ln.Addr().String())
   		select {
   		case <-stopCh:
   			klog.Info(msg)
   		default:
   			panic(fmt.Sprintf("%s due to error: %v", msg, err))
   		}
   	}()
   
   	return serverShutdownCh, listenerStoppedCh, nil
   }
   ```
   
   对 `http.Server.Serve()` 方法的调用位于上述代码要点①处，该语句是被包含在外层协程中运行的，结果就是 `server.Serve` 会阻塞该协程，却不会阻塞当前进程，达到了非阻塞的效果。

### 2.2.2 Server 停机流程

> 详细解析：[Generic Server Graceful Shutdown](/posts/cloud-native/appendix/04丨K8s-Generic-Server-Graceful-Shutdown)

当停机指令发出时，无法预测服务器正处在什么微观状态。例如，它有未处理完毕的请求吗？有客户端正在通过 watch 命令观测 API 实例吗？既不知道，也不能武断猜测。Generic Server 制定了 Server 的生命周期状态，每个状态都具有一个 Go 管道（Channel），用于向外界发出状态转换信息。这些声明周期状态的存在，使得优雅管理 Generic Server 成为可能， 包括停机时能按部就班完成善后处理。

下面代码给出所有生命周期状态的定义，一共八个，前六个停机时会历经，后两个在启动时出现。

```go
// 代码 staging/k8s.io/apiserver/pkg/server/lifecycle_signals.go
type lifecycleSignals struct {
	// ShutdownInitiated event is signaled when an apiserver shutdown has been initiated.
	// It is signaled when the `stopCh` provided by the main goroutine
	// receives a KILL signal and is closed as a consequence.
	// 该事件发生代表 api server 关机信号已发出
	// 主程序的 stopCh 管道收到 KILL 信号并因此被关闭会触发这个信号
	ShutdownInitiated lifecycleSignal

	// AfterShutdownDelayDuration event is signaled as soon as ShutdownDelayDuration
	// has elapsed since the ShutdownInitiated event.
	// ShutdownDelayDuration allows the apiserver to delay shutdown for some time.
	// 该事件发生代表从收到 ShutdownInitialed 后已经过
	// ShutdownDelayDuration 这么长的时间。ShutdownDelayDuration 的存在
	// 使得 api server 可以延迟退出
	AfterShutdownDelayDuration lifecycleSignal

	// PreShutdownHooksStopped event is signaled when all registered
	// preshutdown hook(s) have finished running.
    // 该事件发生代表所有注册的关闭钩子函数均执行完毕
	PreShutdownHooksStopped lifecycleSignal

	// NotAcceptingNewRequest event is signaled when the server is no
	// longer accepting any new request, from this point on any new
	// request will receive an error.
    // 该事件发生代表 Server 不再接收任何新请求
    // 从此新请求会得到 error 作为响应结果
	NotAcceptingNewRequest lifecycleSignal

	// InFlightRequestsDrained event is signaled when the existing requests
	// in flight have completed. This is used as signal to shut down the audit backends
    // 该事件发生代表待处理的请求都已经处理完成
	// 它被用来可关闭 audit 后端的信号
	InFlightRequestsDrained lifecycleSignal

	// HTTPServerStoppedListening termination event is signaled when the
	// HTTP Server has stopped listening to the underlying socket.
    // 该事件发生代表停止监听底层 socket
	HTTPServerStoppedListening lifecycleSignal

	// HasBeenReady is signaled when the readyz endpoint succeeds for the first time.
    // 该事件发生代表 readyz 端点首次返回成功

	HasBeenReady lifecycleSignal

	// MuxAndDiscoveryComplete is signaled when all known HTTP paths have been installed.
	// It exists primarily to avoid returning a 404 response when a resource actually exists but we haven't installed the path to a handler.
	// The actual logic is implemented by an APIServer using the generic server library.
    // 该事件发生代表所有 HTTP paths 已经被安装成功
	// 它存在意义在于，避免在一个 path 安装成功前就对其访问会得到 HTTP404 的反馈
    // 其由实现 Generic Server 实现
	MuxAndDiscoveryComplete lifecycleSignal
}
```

启动状态较少，第一个是启动完成进入正常运转的标志，即状态 HasBeenReady 的达成；第二个是加载所有服务端点的过程，这一过程完成的标志是状态 `MuxAndDiscoveryComplete` 的达成。

而停机涉及到的状态转换就比较多了。体面地收场更能反映系统的强大，这么多的状态本身就反映出开发人员对该过程周密的安排。关机时系统状态转换如下图所示。状态的转换是 `preparedGenericAPIServer.Run()` 方法的重要部分。

> 假设服务器开启了所有可选开关，例如 `ShutdownSendRetryAfter` 开关、 `AuditBackend` 开关等等都打开。

![image-20251123030606406](http://images.liangning7.cn/typora/202511230306640.png)

上图显示，状态流转间善后操作穿插其中都完成了，非常优雅！总结一下这些操作包括：

1. 等待设置的秒数再停机，这是留给 Server 的“优雅退出时间”；
2. 调用 PreShutdownHooks 里设置的停机钩子函数；
3. 等待已收到的客户端请求全部处理完成；
4. 通知 http Server，关闭对端口的监听。

`preparedGenericAPIServer` 的 `Run()` 方法一经运行就不会终止，直到收到停机指令，而当指令到来时，伴随着状态的切换，上述操作开始执行。这些状态切换以及操作的执行有的串行，更多的是并行，上图将这点展示得很清楚。先解释一下技术上是如何定义生命周期状态的相互关系，进行状态切换，并执行伴随切换的操作的。每个生命周期状态都具有类型 `lifecycleSignal`，它的定义如下代码所示：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\lifecycle_signals.go
// Server 生命周期状态都基于一个管道
// lifecycleSignal encapsulates a named apiserver event
type lifecycleSignal interface {
	// Signal signals the event, indicating that the event has occurred.
	// Signal is idempotent, once signaled the event stays signaled and
	// it immediately unblocks any goroutine waiting for this event.
    // Signal 发出事件，指出这个生命周期事件已经发生了
    // Signal 具有幂等性（idempotent），当信号到来时它立即发行等待在
    // 该事件上的 go rountine
	Signal()

	// Signaled returns a channel that is closed when the underlying event
	// has been signaled. Successive calls to Signaled return the same value.
    // 返回一个管道。该管道在其等待的事件发生时被关闭。
	Signaled() <-chan struct{}

	// Name returns the name of the signal, useful for logging.
    // 生命周期状态信号的名称
	Name() string
}
```

在 Go 中实现并行推荐的方式是借助协程（go routine），`Run()` 方法就是这么做的。在它内部启动了许多的协程，它们定义了生命周期状态之间的转换关系，也实现了转换时需要执行的操作，这一过程可以简述为：当进入 A 状态后——也就是它关注的管道关闭了，协程进行此时该做的操作，然后协程关闭状态 A 持有的管道从而切换到 B 状态，等待状态 B 的协程会被激活。来看一个例子：

```go
// 代码: 利用协程定义生命周期状态的转换关系
nonLongRunningRequestDrainedCh := make(chan struct{})
go func() { //要点①
	defer close(nonLongRunningRequestDrainedCh)//要点③
	defer klog.V(1).Info("[graceful-termination] in-flight …")
	// 等待前序状态通知自己其处理已完成，进入当前状态
	<-notAcceptingNewRequestCh.Signaled()
	…
	s.NonLongRunningRequestWaitGroup.Wait() //要点②
}()
```

上例中要点①处通过 `go func() {}` 启动了一个协程，该协程先等待 `NotAcceptingNewRequest` 状态完成其内部处理转入当前状态，一旦达成便在要点②处开始做处于当前状态应作任务——清空已收到的客户端请求。结束后，要点③defer 语句被执行，关闭管道 nonLongRunningRequestDrainedCh，这会通知等待当前状态的其它协程本状态已经完成，它们可以继续处理，下面这个协程就是其中一员，代码如下所示：

```go
// 代码: 另一个协程的执行条件被触发
go func() {
	defer klog.V(1).InfoS("[graceful-termin…", "name", drainedCh.Name())
	defer drainedCh.Signal() //要点①

	<-nonLongRunningRequestDrainedCh
	<-activeWatchesDrainedCh
}()
```

这个例子来自 Generic Server 关机状态转化的真实实现，当上述这个协程也执行完毕后，系统将切换到生命周期的 `InFlightRequestsDrained` 状态，这是代码要点①处的执行结果。

# 3. API 的注入与请求响应

之前提到过，Generic Server 创建了一个 go-restful 中的 Container 实例并放入结构体 GenericAPIServer 结构体的 Handler 属性中，用于暴露 Kubernetes API 的 RESTful 服务，但目前这个 Container 还是空的，没有任何 WebService 注册其中。当然，Generic Server 自己也不会有任何的 Kubernetes API 需要注册，基于它构建的上层 Server 才会有，它需要提供一个接口给上层 Server，用于传递 API 进来填充 Container。这节讲解 API 的注入过程，以及为随之形成的端点设置响应函数的过程。注入完成后 Generic Server 内部的 go-restful 框架内将创建出如下图所示的概念实例。

![image-20251123130808857](http://images.liangning7.cn/typora/202511231308068.png)

每个 API 组版本将形成一个 go-restful 的 WebService，一个组版本下的所有 GVK 都会成为这个 WebService 的 Route。由于每个 GVK 都可能支持多个操作，如查询、创建等，所以一个 GVK 完全可能形成多个 Route。

## 3.1 注入处理流程

> 详情查看 [K8s Generic Server API Inject](/posts/cloud-native/appendix/05丨K8s-Generic-Server-API-Inject)

Generic Server 对外提供了 API 注入接口，这些接口又会调用内部方法完成注入操作。接口、方法的调用过程如下图所示。本节将讲解这一过程。

![image-20251123131332380](http://images.liangning7.cn/typora/202511231313622.png)

### 3.1.1 GenericAPIServer.InstallAPIGroups() 方法

Generic Server 提供了两个同质的接口方法：`InstallAPIGroup()` 和 `InstallAPIGroups()` 方法，前者可以注册**一个 API 组**，后者可以注册**一组 API 组**；前者调用后者来完成工作。形式参数对于接口方法来说比较重要的，`InstallAPIGroups()` 方法的签名如下：

```go
func (s *GenericAPIServer) InstallAPIGroups(apiGroupInfos …*APIGroupInfo)error
```

上层 Server 在调用该方法注册 API 时，只需要提供一个元素为 APIGroupInfo 结构体引用的数组。APIGroupInfo 结构体的定义代码如下所示：

```go
// Info about an API group.
type APIGroupInfo struct {
	PrioritizedVersions []schema.GroupVersion
	// Info about the resources in this group. It's a map from version to resource to the storage.
    // 版本，资源和存储对象的映射
	VersionedResourcesStorageMap map[string]map[string]rest.Storage
	// OptionsExternalVersion controls the APIVersion used for common objects in the
	// schema like api.Status, api.DeleteOptions, and metav1.ListOptions. Other implementors may
	// define a version "v1beta1" but want to use the Kubernetes "v1" internal objects.
	// If nil, defaults to groupMeta.GroupVersion.
	// TODO: Remove this when https://github.com/kubernetes/kubernetes/issues/19018 is fixed.
	OptionsExternalVersion *schema.GroupVersion
	// MetaGroupVersion defaults to "meta.k8s.io/v1" and is the scheme group version used to decode
	// common API implementations like ListOptions. Future changes will allow this to vary by group
	// version (for when the inevitable meta/v2 group emerges).
	MetaGroupVersion *schema.GroupVersion

	// Scheme includes all of the types used by this group and how to convert between them (or
	// to convert objects from outside of this group that are accepted in this API).
	// TODO: replace with interfaces
    // 注册表
	Scheme *runtime.Scheme
	// NegotiatedSerializer controls how this group encodes and decodes data
    // 编解码器
	NegotiatedSerializer runtime.NegotiatedSerializer
	// ParameterCodec performs conversions for query parameters passed to API calls
    // 查询参数转换器
	ParameterCodec runtime.ParameterCodec

	// StaticOpenAPISpec is the spec derived from the definitions of all resources installed together.
	// It is set during InstallAPIGroups, InstallAPIGroup, and InstallLegacyAPIGroup.
    // 调用 InstallAPIGroups, InstallAPIGroup 和 InstallLegacyAPIGroup 时
    // 生成的 OpenAPI 规格说明文档
	StaticOpenAPISpec map[string]*spec.Schema
}
```

`APIGroupInfo` 结构体的每个字段都有其作用，尤其字段 `VersionedResourcesStorageMap`。 这是一个 Map，它把各个 API 组的版本映射到 `rest.Storage` 接口类型的实例，这种实例同时实现了用于响应 HTTP 请求（GET，POST 等）的众多接口，这些接口也定义在 Storage 接口所在文件内。也就是说它们实际包含了请求响应逻辑。在 Kubernetes API 的注册过程中， `GenericAPIServer` 结构体的 `getAPIGroupVersion()` 方法会被调用，就是它把 `Storage` 实例从 `VersionedResourcesStorageMap` 中取出，交给注册过程去设置端点响应函数。

### 3.1.2 GenericAPIServer.installAPIResources() 方法

上述 `InstallAPIGroups()` 方法的职责是对接上层 Server，真正去触发注入的是 `installAPIResources()` 方法。这是 Generic Server 的一个私有方法，它转化接收到的参数，把上层 Server 给出的 API Group（即 APIGroupInfo 结构体实例）的各个 Version 分别注入 Generic Server 中。APIGroupInfo 结构体的 `PrioritizedVersions` 字段包含了该 Group 具有的所有 Version， 遍历调用 `endpoints` 包的注入方法 `endpoints.APIGroupVersion.InstallREST()` 即可。

`installAPIResources()` 方法根据 APIGroupInfo 实例构建出了多个 APIGroupVersion 结构体实例，Group 的每个 Version 一个，如上所述 API 的注入就是以这些 APIGroupVersion 实例为单位逐一进行的，后续方法中将大量使用这一信息。APIGroupVersion 实例的构造主要由两个方法完成，代码如下所示：

```go
// 代码 staging/k8s.io/apiserver/pkg/server/genericapiserver.go
func (s *GenericAPIServer) getAPIGroupVersion(apiGroupInfo *APIGroupInfo, groupVersion schema.GroupVersion, apiPrefix string) (*genericapi.APIGroupVersion, error) {
	storage := make(map[string]rest.Storage)
    // 要点①
	for k, v := range apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version] {
		if strings.ToLower(k) != k {
			return nil, fmt.Errorf("resource names must be lowercase only, not %q", k)
		}
		storage[k] = v
	}
	version := s.newAPIGroupVersion(apiGroupInfo, groupVersion)
	version.Root = apiPrefix
	version.Storage = storage
	return version, nil
}
func (s *GenericAPIServer) newAPIGroupVersion(apiGroupInfo *APIGroupInfo, groupVersion schema.GroupVersion) *genericapi.APIGroupVersion {

	allServedVersionsByResource := map[string][]string{}
	for version, resourcesInVersion := range apiGroupInfo.VersionedResourcesStorageMap {
		for resource := range resourcesInVersion {
			if len(groupVersion.Group) == 0 {
				allServedVersionsByResource[resource] = append(allServedVersionsByResource[resource], version)
			} else {
				allServedVersionsByResource[resource] = append(allServedVersionsByResource[resource], fmt.Sprintf("%s/%s", groupVersion.Group, version))
			}
		}
	}

	return &genericapi.APIGroupVersion{
		GroupVersion:                groupVersion,
		AllServedVersionsByResource: allServedVersionsByResource,
		MetaGroupVersion:            apiGroupInfo.MetaGroupVersion,

		ParameterCodec:        apiGroupInfo.ParameterCodec,
		Serializer:            apiGroupInfo.NegotiatedSerializer,
		Creater:               apiGroupInfo.Scheme,
		Convertor:             apiGroupInfo.Scheme,
		ConvertabilityChecker: apiGroupInfo.Scheme,
		UnsafeConvertor:       runtime.UnsafeObjectConvertor(apiGroupInfo.Scheme),
		Defaulter:             apiGroupInfo.Scheme,
		Typer:                 apiGroupInfo.Scheme,
		Namer:                 runtime.Namer(meta.NewAccessor()),

		EquivalentResourceRegistry: s.EquivalentResourceRegistry,

		Admit:             s.admissionControl, //要点②
		MinRequestTimeout: s.minRequestTimeout,
		Authorizer:        s.Authorizer,
	}
}
```

上述代码有两个值得特别关注的信息：

1. 要点①处开始从 VersionedResourcesStorageMap 属性中的 Storage 信息，上文已经提及该信息的重要性。
2. 要点②处把准入控制插件信息从 Generic Server 的 `admissionControl` 字段抽取到 `APIGroupVersion` 实例的 Admit 字段中，在生成 route 响应函数时会使用 Admit 字段， 使得针对 Kubernetes API 的 Create、Update、Delete 和 Connect 请求经过这些准入控制器处理。

###3.1.3 APIGroupVersion.InstallREST() 方法和 APIInstaller.Install() 方法

`InstallREST()` 和 `Install()` 两个方法主要起到拆解细化的作用，为一个 Group Version 生成一个 go-restful 的 WebService 对象。

`Install()` 方法会拆解一个 Group Version，得到它包含的 GVK 集合并遍历这个集合，以当前 GVK 为入参调用方法 `registerResourceHandlers()`，为它生成 go-restful 中 route 对象并绑定好 route 响应函数，最后把 route 交给 Group Version 的 WebService 实例，返回该 WebService 给调用者。`registerResourceHandlers()` 方法十分重要，下面会进行详细讲解。

`InstallREST()` 方法会调用 `Install()` 获得 WebService 实例，并把这个 WebService 变量放入 GoRestfulContainer 中，这样一个 Group Version 向 Generic Server 的注入就完成了。

## 3.2 WebService 及其 Route 生成过程

> 详情查看 [K8s Generic Server WebService](/posts/cloud-native/appendix/06丨K8s-Generic-Server-WebService)

在 go-restful 核心概念及其相互关系中，endpoints 包中结构体 APIInstaller 的 `registerResourcehandlers()` 方法是魔法所在地，是最终生成 go-restful 的 route 并为其绑定响应函数的地方。这个方法有 1000 行左右， 可见其任务之繁重。从大的步骤来看，这个方法虽长但逻辑并不复杂，重要的步骤如图所示：

![image-20251123133243277](http://images.liangning7.cn/typora/202511231332520.png)

上图中省略了大量细节，例如针对不同作用域（集群范围或命名空间范围）的 API，其 URL 路径的计算逻辑不同。

### 3.2.1 获取每个 GVK 支持的端点

`APIInstaller.Install()` 方法会为每一个 GV 生成一个 go-restful WebService 对象, 然后用 GVK 的 path 和 storage 为实参调用 `registerResourcehandlers()` 方法，把该 GV 下所有 GVK 所支持的端点注册为这个 WebService 下的 route。要达到这个目的首先要搞清楚当前 GVK 支持什么端点，包括所支持的 HTTP 方法以及各种 HTTP 方法对应的处理器是什么。答案都蕴含在 rest.Storage 接口实例中。`registerResourcehandlers()` 方法是这么从 Storage 中获取以上信息的，源码如下所示：

```go
	// 代码: staging/k8s.io/apiserver/pkg/endpoints/installer.go
	// 获取 storage 对象都支持哪些 verb，也就是 HTTP 方法
	creater, isCreater := storage.(rest.Creater)
	namedCreater, isNamedCreater := storage.(rest.NamedCreater)
	lister, isLister := storage.(rest.Lister)
	getter, isGetter := storage.(rest.Getter)
	getterWithOptions, isGetterWithOptions := storage.(rest.GetterWithOptions)
	gracefulDeleter, isGracefulDeleter := storage.(rest.GracefulDeleter)
	collectionDeleter, isCollectionDeleter := storage.(rest.CollectionDeleter)
	updater, isUpdater := storage.(rest.Updater)
	patcher, isPatcher := storage.(rest.Patcher)
	watcher, isWatcher := storage.(rest.Watcher)
	connecter, isConnecter := storage.(rest.Connecter)
	storageMeta, isMetadata := storage.(rest.StorageMetadata)
	storageVersionProvider, isStorageVersionProvider := storage.(rest.StorageVersionProvider)
	gvAcceptor, _ := storage.(rest.GroupVersionAcceptor)
	if !isMetadata {
		storageMeta = defaultStorageMetadata{}
	}

	if isNamedCreater {
		isCreater = true
	}
```

上述代码试着把当前 GVK 的变量 storage 向 rest 包下的多个接口做运行时类型转换，这些接口就是 HTTP 请求响应对象应具有的类型，包括 Creater、NameCreater、Lister、Getter、 Updater、Patcher 等等。这样既获知了当前 GVK 是否支持某 HTTP 方法，又获得了该 HTTP 方法的响应对象——经过类型转换后的 storage 实例。

### 3.2.2 获取端点参数

一个 HTTP 端点可能接受不同的URL参数，这些参数可以通过问号后跟的名值对给出， 也可以是 URL 路径的一部分，在 go-restful 中参数将成为 route 的构成信息。每个 GVK 所支持的每个 HTTP 方法都有自己所支持的参数，系统需要获取这些参数，以备制作 route 时之用。拿 GET 参数的制作为例，获取参数的代码如下所示：

```go
	// 代码 staging/k8s.io/apiserver/pkg/endpoints/installer.go
	if isGetterWithOptions {
		getOptions, getSubpath, _ = getterWithOptions.NewGetOptions()
		getOptionsInternalKinds, _, err := a.group.Typer.ObjectKinds(getOptions)
		if err != nil {
			return nil, nil, err
		}
		getOptionsInternalKind = getOptionsInternalKinds[0]
		versionedGetOptions, err = a.group.Creater.New(a.group.GroupVersion.WithKind(getOptionsInternalKind.Kind)) //要点①
		if err != nil {
			versionedGetOptions, err = a.group.Creater.New(optionsExternalVersion.WithKind(getOptionsInternalKind.Kind))
			if err != nil {
				return nil, nil, err
			}
		}
		isGetter = true
	}
```

这段代码要点①表明，上层 Server 给出的 APIGroupInfo 实例提供了获取 GET 参数的方法，因为代码中的 `a.group` 就来自于该实例。

接下来，程序把已经获得的 HTTP 响应对象和 URL 参数包装到 action 结构体实例中，形成一个 actions 数组，这么做并没有特别的目的，只是方便后续遍历它从而创建出 route 数组，逻辑更清晰一些吧。可见域为命名空间时，actions 数组构造代码如下所示：

```go
// 代码 staging/k8s.io/apiserver/pkg/endpoints/installer.go
// Handler for standard REST verbs (GET, PUT, POST and DELETE).
// Add actions at the resource path: /api/apiVersion/resource
actions = appendIf(actions, action{request.MethodList, resourcePath, resourceParams, namer, false}, isLister)
actions = appendIf(actions, action{request.MethodPost, resourcePath, resourceParams, namer, false}, isCreater)
actions = appendIf(actions, action{request.MethodDeleteCollection, resourcePath, resourceParams, namer, false}, isCollectionDeleter)
// DEPRECATED in 1.11
actions = appendIf(actions, action{request.MethodWatchList, "watch/" + resourcePath, resourceParams, namer, false}, allowWatchList)

// Add actions at the item path: /api/apiVersion/resource/{name}
actions = appendIf(actions, action{request.MethodGet, itemPath, nameParams, namer, false}, isGetter)
if getSubpath {
    actions = appendIf(actions, action{request.MethodGet, itemPath + "/{path:*}", proxyParams, namer, false}, isGetter)
}
actions = appendIf(actions, action{request.MethodPut, itemPath, nameParams, namer, false}, isUpdater)
actions = appendIf(actions, action{request.MethodPatch, itemPath, nameParams, namer, false}, isPatcher)
actions = appendIf(actions, action{request.MethodDelete, itemPath, nameParams, namer, false}, isGracefulDeleter)
// DEPRECATED in 1.11
actions = appendIf(actions, action{request.MethodWatch, "watch/" + itemPath, nameParams, namer, false}, isWatcher)
actions = appendIf(actions, action{request.MethodConnect, itemPath, nameParams, namer, false}, isConnecter)
actions = appendIf(actions, action{request.MethodConnect, itemPath + "/{path:*}", proxyParams, namer, false}, isConnecter && connectSubpath)
```

### 3.2.3 生成 go-restful Route 数组

需要为当前 GVK 的 WebService 创建 route，这项工作是基于上述 actions 数组来做的， 每个 action 都会成为一个 go-restful route 交给 WebService。一个 action 所对应的 HTTP 方法可能会不同，程序用了一个 case 语句去区分，GET 操作的 route 生成代码如下所示：

```go
		// 代码: staging/k8s.io/apiserver/pkg/endpoints/installer.go
		case request.MethodGet: // Get a resource.
			var handler restful.RouteFunction
			if isGetterWithOptions {
				handler = restfulGetResourceWithOptions(getterWithOptions, reqScope, isSubresource)
			} else {
				handler = restfulGetResource(getter, reqScope)
			}

			if needOverride {
				// need change the reported verb
				handler = metrics.InstrumentRouteFunc(verbOverrider.OverrideMetricsVerb(action.Verb), group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, handler)
			} else {
				handler = metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, handler)
			}
			handler = utilwarning.AddWarningsHandler(handler, warnings)

			doc := "read the specified " + kind
			if isSubresource {
				doc = "read " + subresource + " of the specified " + kind
			}
			route := ws.GET(action.Path).To(handler). //要点①
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed. Defaults to 'false' unless the user-agent indicates a browser or command-line HTTP tool (curl and wget).")).
				Operation("read"+namespaced+kind+cases.Title(language.English).String(subresource)+operationSuffix).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
				Returns(http.StatusOK, "OK", producedObject).
				Writes(producedObject)
			if isGetterWithOptions {
				if err := AddObjectParams(ws, route, versionedGetOptions); err != nil {
					return nil, nil, err
				}
			}
			addParams(route, action.Params)
			routes = append(routes, route) //要点②
```

上述代码在要点①处制作了一个 route ，它是一个 rest.RouteBuilder 类型的实例，其核心要素是 handler，即请求响应对象，这一信息在前面第 1 步已经获得；最后要点②处把该 route 存入 routes 数组。

### 3.2.4 将 routes 数组交给 WebService

这一步就相对简单，遍历得到的 routes 数组，调用 WebService 的方法把所有 route 加入其中，大功告成。

```go
// 代码: staging/k8s.io/apiserver/pkg/endpoints/installer.go
for _, route := range routes {
	route.Metadata(RouteMetaGVK, metav1.GroupVersionKind{
		Group:   reqScope.Kind.Group,
		Version: reqScope.Kind.Version,
		Kind:    reqScope.Kind.Kind,
	})
	route.Metadata(RouteMetaAction, strings.ToLower(action.Verb))
	ws.Route(route)
}
```

## 3.3 响应对 Kubernetes API 的 HTTP 请求

Kubernetes API 被注入到 Generic Server 中形成了 WebService 以及 Route，当有 HTTP 请求到来时，Server 根据 URL 最终调用到 Route 上的处理器去响应，本节讲解处理器的内部工作流程。

### 3.3.1 请求数据解码和准入控制

route 上的响应处理器（handler）是如何构造出来的可以揭示它将做如何工作。以处理对某 Kubernetes API 的 HTTP POST 请求为例，其 handler 设置代码如下所示：

```go
// 代码 staging/src/k8s.io/apiserver/pkg/endpoints/installer.go
case request.MethodPost: // Create a resource.
	var handler restful.RouteFunction
	if isNamedCreater {
		handler = restfulCreateNamedResource(namedCreater, reqScope, admit)
	} else {
        //要点①
		handler = restfulCreateResource(creater, reqScope, admit)
	}
	//要点②
	handler = metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, handler)
	handler = utilwarning.AddWarningsHandler(handler, warnings)
	article := GetArticleForNoun(kind, " ")
	doc := "create" + article + kind
	if isSubresource {
		doc = "create " + subresource + " of" + article + kind
	}
	route := ws.POST(action.Path).To(handler).
		Doc(doc).
		Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed. Defaults to 'false' unless the user-agent indicates a browser or command-line HTTP tool (curl and wget).")).
		Operation("create"+namespaced+kind+cases.Title(language.English).String(subresource)+operationSuffix).
		Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
		Returns(http.StatusOK, "OK", producedObject).
		// TODO: in some cases, the API may return a v1.Status instead of the versioned object
		// but currently go-restful can't handle multiple different objects being returned.
		Returns(http.StatusCreated, "Created", producedObject).
		Returns(http.StatusAccepted, "Accepted", producedObject).
		Reads(defaultVersionedObject).
		Writes(producedObject)
	if err := AddObjectParams(ws, route, versionedCreateOptions); err != nil {
		return nil, nil, err
	}
	addParams(route, action.Params)
	routes = append(routes, route)
```

如果以可见性为集群的资源创建为例，代码中要点①处的方法 `restfulCreateResource()` 构造了资源创建 handler。可见性为集群意味着建出的资源命名空间不相关，相关的情况是完全类似的。调用该方法时使用了三个输入参数：

1. `creater`：这是由当前 GVK 的 `rest.Storage` 实例向 `rest.Creater` 做动态类型转换得来， 本质上还是 GVK 的 r`est.Storage` 实例。

2. `reqScope`：提供一些辅助信息，主要来自 API 注入时使用的 APIGroupInfo。它会有一个 Serializer 字段，后续被用于把 HTTP 消息体内的信息“反序列化”为目标 API 的基座结构体实例。

3. `admit`：准入控制器集合(Admission)，准入控制机制是在进入业务逻辑前对请求做 的一些修改与校验，主要是安全方面的控制。

   > 准入控制和请求过滤机制的区别：
   >
   > * 过滤器对所有到来的 HTTP 请求有效，
   > * 准入控制只针对目标为 Kubernetes API 的创建，修改，删除和 Connect 请求；
   > * 过滤器发生在请求内容被反序列化为 Kubernetes API 的 Go 基座结构体实例之前
   > * 准入控制器发生在之后。

在代码的要点②处对该 handler 做了进一步处理，加入测量和异常处理，无关大局，而如果继续深入 `restfulCreateResource()` 方法内部，探究 handler 如何处理请求，最终会定位到 handlers 包下的 `createHandler()` 方法，其中含有 handler 的实现。该方法非常长，声明部分代码如下：

```go
// 代码 staging/k8s.io/apiserver/pkg/endpoints/handlers/create.go
func createHandler(r rest.NamedCreater, scope *RequestScope, admit admission.Interface, includeName bool) http.HandlerFunc {
	return func(w http.ResponseWriter, req *http.Request) {
		ctx := req.Context()
		// 出于追踪性能的目的
		ctx, span := tracing.Start(ctx, "Create", traceFields(req)...)
		defer span.End(500 * time.Millisecond)

		namespace, name, err := scope.Namer.Name(req)
		if err != nil {
			if includeName {
				// name was required, return
				scope.err(err, w, req)
				return
			}
			// …
		}

		namespace, err = scope.Namer.Namespace(req)
		if err != nil {
			scope.err(err, w, req)
			return
		}
	}
}
```

它直接返回了一个匿名函数，其形式参数`(w http.ResponseWriter, req *http.Request)`是不是很眼熟？这个匿名函数将来会负责接收 HTTP 请求的 request 和 response 并处理，这是魔法所在地。该匿名方法内容很多，解读两个要点。第一是请求数据解码，第二是准入控制器的调用。

* 数据解码是指把 HTTP 请求消息体提取出来，转化为目标 GVK 的 Go 基座结构体实例，这是一个由字符串到 Go 程序变量的过程。数据解码是执行业务逻辑的前提。如果以上创建 API 实例为例，客户端放在消息体内的是待创建 API 资源实例的内容，要先把它转化为 Go 变量才能进行后续的处理。在上述匿名函数中，首先根据 HTTP 请求的 MediaType 信息从当前 GVK 支持的所有序列化器中选出一个适用的；然后利用这个序列化器和 HTTP 请求所使用的 API 版本，制造一个解码器；最后利用这个解码器从 HTTP 请求体中得到 GVK 的 Go 实例和 GVK 信息。解码和获取 GVK 信息的代码如下所示：

  ```go
  decoder := scope.Serializer.DecoderToVersion(decodeSerializer, scope.HubGroupVersion)
  span.AddEvent("About to convert to expected version")
  obj, gvk, err := decoder.Decode(body, &defaultGVK, original)
  ```

* 再看如何调用准入控制器。准入控制器是外部交给本匿名函数的，直接使用就好。每个准入控制器都有两个能力：修改（mutate）HTTP 请求给出的 API 实例信息；在 ETCD 操作前，校验（validate）API 实例信息。每个 HTTP 请求处理器都会分两个阶段调用准入控制器的这两个接口方法，修改操作在前，校验操作在后。准入控制器为用户提供了一个有用的扩展点，可以把特殊的需求注入 API Server 内，例如 SideCar 模式中边车容器就可以在修改阶段注入 Pod。在资源创建场景的处理器中，准入控制器的两个能力是这么被调用的：

  ```go
  		// 代码:	staging/k8s.io/apiserver/pkg/endpoints/handlers/create.go
  		span.AddEvent("About to store object in database")
  		admissionAttributes := admission.NewAttributesRecord(obj, nil, scope.Kind, namespace, name, scope.Resource, scope.Subresource, admission.Create, options, dryrun.IsDryRun(options.DryRun), userInfo)
  		requestFunc := func() (runtime.Object, error) {
  			return r.Create(
  				ctx,
  				name,
  				obj,
  				rest.AdmissionToValidateObjectFunc(admit, admissionAttributes, scope), //要点①
  				options,
  			)
  		}
  		// Dedup owner references before updating managed fields
  		dedupOwnerReferencesAndAddWarning(obj, req.Context(), false)
  		result, err := finisher.FinishRequest(ctx, func() (runtime.Object, error) {
  			liveObj, err := scope.Creater.New(scope.Kind)
  			if err != nil {
  				return nil, fmt.Errorf("failed to create new object (Create for %v): %v", scope.Kind, err)
  			}
  			obj = scope.FieldManager.UpdateNoErrors(liveObj, obj, managerOrUserAgent(options.FieldManager, req.UserAgent()))
  			admit = fieldmanager.NewManagedFieldsValidatingAdmissionController(admit)
  
  			if mutatingAdmission, ok := admit.(admission.MutationInterface); ok && mutatingAdmission.Handles(admission.Create) {
                  //要点③
  				if err := mutatingAdmission.Admit(ctx, admissionAttributes, scope); err != nil {
  					return nil, err
  				}
  			}
  			// Dedup owner references again after mutating admission happens
  			dedupOwnerReferencesAndAddWarning(obj, req.Context(), true)
  			result, err := requestFunc()  //要点②
  			...
  ```

  代码中要点①处把准入控制的校验函数交给 `rest.Create` 方法，当 Create 被调用到时就会被启用，而 Create 是在要点被真正调用的；准入控制器的修改操作是要点②处被调用的。

数据解码和准入控制相当于运行业务逻辑之前的预处理，接下来将进入业务逻辑部分， 这部分基本都会与 ETCD 交互进行数据的存取，所以不妨称其为数据存取。

### 3.3.2 数据存取

> [K8s Generic Server Strategy](/posts/cloud-native/appendix/07丨K8s-Generic-Server-Strategy)

在前面强调了结构体 APIGroupInfo 的字段 `VersionedResourcesStorageMap` 很重要，它提供了从一个 GVK 到类型为 `rest.Storage` 接口的实例的映射，这些 `rest.Storage` 接口的实例 会最终负责完成 HTTP 请求中所要求的操作，也就是响应 C(reate)R(ead)U(date)D(elete) 以及 Watch 等 Verb，这意味着这些实例要根据自身需要，实现部分如下 Verb 相关接口：

1. `rest.Creater` 接口
2. `rest.NamedCreater` 接口
3. `rest.Lister` 接口
4. `rest.Getter` 接口
5. `rest.GetterWithOptions` 接口
6. `rest.GracefulDeleter` 接口
7. `rest.CollectionDeleter` 接口
8. `rest.Updater` 接口
9. `rest.Patcher` 接口
10. `rest.Watcher` 接口
11. `rest.Connecter` 接口
12. `rest.StorageMetadata` 接口
13. `rest.StorageVersionProvider` 接口
14. `rest.GroupVersionAcceptor` 接口

针对不同 Kubernetes API 的 HTTP 请求，处理逻辑不同，创建一个 Pod 和创建一个 Deployment 不可能一样，于是就需要针对不同 Kubernetes API 定义不同结构体去实现 `rest.Storage` 接口和以上 Verb 接口，然后用该类型实例去填充`VersionedResourcesStorageMap`。 那么，通过在源代码中查找所有`rest.Storage` 接口的实现者，就应该可以找到所有API 的 HTTP 请求处理方法了。

为了方便理解，还是以 Deployment 为例。其 rest.Storage 实例的实际类型是如下 REST 结构体：

```go
// 代码: pkg/registry/apps/deployment/storage/storage.go
// REST 为 Deployments 资源实现 RESTStorage.
type REST struct {
	*genericregistry.Store
}
```

REST 结构体基本是空的，除了匿名嵌套 `generic/registry.Store` 结构体。嵌套的结果是它 “继承”了 `generic/registry.Store` 的所有属性与方法，源文件位于 `staging/k8s.io/apiserver/pkg/registry/generic/registry/store.go`。

这样的复用极大减轻了各个 GVK 实现 HTTP 响应逻辑的压力，但问题是：如果大家都嵌套同一个结构体复用它的方法，岂不是大家的 HTTP 响应逻辑都一样了吗？

创建 Pod 和 Deployment 不可能一样。规避这个问题的方法隐藏在`generic/registry.Store` 结构体的字段中， 这些字段绝大多数都是扩展点，不同 API 通过给这些扩展点赋予不同的值来控制响应方法的内部操作，比较典型的是`*Strategy` 系列属性：`CreateStrategy`、`DeleteStrategy`、`UpdateStrategy`。

策略设计模式在软件开发中经常被用到，它的作用是抽象出一组类似逻辑中的不同部分，形成策略对象，从而统一余下的部分进行复用，不同使用场景使用不同的策略对象。`generic/registry.Store` 中的这组策略属性就是这个套路，Pod 会提供 Pod 的策略到自己的 REST 结构体实例上，而 Deployment 会有 Deployment 的策略。

最后来总结 Generic Server 如何为 Kubernetes API 提供 Verb 的响应逻辑，如图 所示。为了方便展示，图中以一些核心 API Server 的资源为例，也未考虑命名空间。

![image-20251123145524230](http://images.liangning7.cn/typora/202511231455582.png)

# 4. 准入控制机制

> [K8s Generic Server Admission](/posts/cloud-native/appendix/08丨K8s-Generic-Server-Admission)

准入控制的出现就是由安全考量出发，在 API Server 底层构建的一个可扩展安全机制。

## 4.1 什么是准入控制

在基于 Kubernetes 搭建系统，或在其上运行一个应用时，你是否曾有如下需求：

1. 怎么强制系统内所有 Pod 消耗的资源不超过约定的警戒线？
2. 怎么保证系统内所有 Pod 都不会使用某个容器镜像？
3. 如何确保某个 Deployment 具有最高的优先级？
4. 已知某个 Pod 需要的权限范围，如何确保生产环境下它不被错误地关联一个权限很大的 Service Account？

准入控制（Admission Control）是这类问题的一个理想答案：为了解决上述问题，可以在 API Server 上加准入控制器，在控制器中完成对请求的必要调整和校验，从而消除安全担忧。准入控制机制会截获所有对目标 API 的创建、修改和删除请求，交给该准入控制器预处理。

准入控制是 API Server 接收到 Verb 为创建、修改、删除和 CONNECT 的 HTTP 请求后，将资源信息存入 ETCD 前，对请求做修改和校验的过程。准入控制在请求处理过程中被触发的时点如图 5-19 所示。准入控制机制依靠准入控制器对请求做修改和检验，Kubernetes 内建近 30 个准入控制器，用户也可以通过网络钩子（Webhook）挂载自开发控制器。

![image-20251123145918478](http://images.liangning7.cn/typora/202511231459737.png)

准入控制机制因安全而被引入，但其作用却不限于安全领域，在不滥用的前提下，可以利用该机制完成如下类型的任务：

1. 安全检测：准入控制可以在整个集群或一个命名空间范围内落实安全基线。例如内建准入控制器中有一个专门用于 Pod 的配置：PodSecurityPolicy 控制器，它会禁止目标容器以 root 运行，并可以确保容器的 root 文件系统的挂载模式为只读。
2. 配置管控：准入控制还是落实各种规约的好地方。应用系统一般都会有些构建要求，例如每一个 Pod 都需要声明资源限制，都需要打好某个特定标签等等。这就像纪律，需要有机制确保纪律被遵守。在 API Server 中准入控制就常常被用来集中打标签（label）和 注解（annotation）。

值得注意的是准入控制**只针对创建，删除，修改和 CONNECT 请求**，其中对 Connect 请求的支持是最近版本才加入的；而对于读取类请求（Get，Watch，List）准入控制不加干预，对于其它客制化的 verb 也不起作用。可以这么理解：准入控制实际是“**准许信息进入 ETCD 的控制**”。这反映出该机制的局限性，它只能保证信息持久化到系统时安全规则被遵守，一旦进入，后续其被使用时将不再受准入控制的约束。这种职责范围非常清晰的设计不失为一种明智之举，在使用该机制时（特别是借助 Webhook 创建动态准入规则时），应该继承这种思想，不滥用准入控制。

整个准入控制机制分为两个阶段：修改阶段和校验阶段。

* 在修改阶段几乎可以不受限制地修改请求中包含的 API 实例，这是非常强大的存在，借助它可以加资源使用限制、修改暴露的端口、补全重要信息，也可以在 Pod 中悄悄注入容器，边车模式中的边车往往如此进入 Pod，就像 ISTIO 项目中注入网络代理边车那样。
* 校验阶段无权修改目标资源，而是从信息一致性、完备性角度去验证请求内容的准确性，如果校验的结果是失败，则立即返回错误给客户端，不再执行后续校验。这两个阶段先后执行，修改在前，校验在后。修改阶段的准入控制器串行执行，但系统不保证执行顺序；校验阶段的控制器并行执行。

API Server 把冗长的请求处理划分为三个部分：

1. 首先过滤阶段：请求首先进入过滤阶段，目的是高效地做全局性访问控制，典型操作如：登录、吞吐量控制。本阶段不对请求体包含的信息进行深入解析，因为它是普适的，不依赖业务信息。同时每一个到来的请求都需要经过过滤，无论 Verb 是什么。集群管理员在启动 API Server 时可以利用命令行标志启用或关闭特定过滤器。
2. 然后准入控制阶段：创建、修改、删除和 Connect 类请求特有，目的是确保进入系统的信息是合规的，安全方面是主要考量。准入控制的主要输入是目标 API 实例，通过解码请求内容得到。管理员在启动 API Server 时可以利用命令行标志启用或关闭特定准入控制器。
3. 最后持久化阶段：信息存入 ETCD。

请求过滤与准入控制分处前两个阶段，输入不同，目标也不同。此外，准入控制机制为用户提供了扩展机制——Webhook 控制器，而过滤器机制并没有这种可能。

## 4.2 准入控制器

准入控制的核心是准入控制器，每个控制器都具有独到的作用，所有控制器共同支撑起准入控制机制。从外部看，准入控制机制和控制器是一体的，然而从内部看，控制器独立存在，它们以插件形式加入到整个准入控制机制中，插件化使得引入新的控制器变得容易。在 API Server 启动时，管理员可以利用命令行标志指出启用哪些准入插件而禁用哪些。例如，如下命令启动 API Server 的同时开启两个控制器：

```bash
kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger …
```

而如下标志则禁用两个：

```bash
kube-apiserver --disable-admission-plugins=PodNodeSelector, AlwaysDeny …
```

Kubernetes 已经提供了 34 个开箱即用的准入控制器，上面两个命令用到的 NamespaceLifecycl、LimitRanger、PodNodeSelector 和 AlwaysDeny 均来自这组控制器。

技术上说，一个准入控制器插件主要涉及如下图所示的三个接口：

![image-20251123152045229](http://images.liangning7.cn/typora/202511231520497.png)

1. `admission.Interface`：控制器插件必须实现的接口。
2. `admission.MutationInterface`：参与修改阶段必须实现的接口，它使得本插件成为修改准入控制器。
3. `admission.ValidationInterface`：参与校验阶段必须实现的接口，它使得本插件成为校验准入控制器。

以上每个接口都很简洁，各有一个接口方法。`admission.Interface` 的 `Handles()` 方法会接收 Operation 类型的参数，它是一个字符串，值可以是 CREATE、UPDATE、DELETE 和 CONNECT。如果 `Handles()` 返回 true，则这个插件可以处理该类 HTTP 请求。`Handles()` 方法会被准入控制机制调用以确认该控制器插件是否需要参与当前请求的处理。而接口 `admission.MutationInterface` 和 `admission.ValidationInterface` 各自有一个方法去修改或去校验请求内容，这些方法入参中的一个类型为 Attributes，该入参会提供目标资源的基本信息，通过它也可以直接获取到请求中的资源，在大多数场景下基于这些信息足够完成准入控制的工作。另一个类型为 ObjectInterfaces 的入参可以从请求中获取 Defaulter、Converter 这种不太常用的信息，在处理 CRD 时有可能需要它们。

开发一个内建准入控制器并不难，只要以一个 Go 结构体为基座制作一个插件，实现以上三个接口，并注入到准入控制机制中就可以了。

作为 Kubernetes 的普通用户并没有定义内建准入控制器的机会，需要通过**动态准入控制器**（Webhooks）注入自己的控制逻辑。但在自开发聚合 Server 的场景中，准入控制器的开发是完全可行并且比较重要的工作。

## 4.2 动态准入控制

在内建的准入控制器中，有两个特殊的控制器——MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook。它们本质上各自代表了一系列由用户自己开发出来的准入控制器，是 Kubernetes 为客户提供的一种扩展机制。本小节着重介绍其结构和工作原理。

### 4.2.1 工作原理

一般的准入控制器代码是随 API Server 源码一起编译打包的，属于 API Server 可执行文件的一部分，用户不可能去把自己的控制逻辑以这种方式放入 API Server，而动态准入控制的两个准入控制器为用户扩展开了口子：用户只需将自己的准入控制逻辑编写为**独立运行的网络服务**并部署在集群内或集群外，通过配置告知上述动态准入控制器如何使用这些服务，这样就完成了准入机制的扩展。准入控制除了做安全控制，技术上也可以做任何其它控制，这一点通过观察内建准入控制器也可以体会得到，由此可见，动态准入控制实际上为扩展 Kubernetes 开辟了一条重要的通道。

动态准入控制机制涉及一些 Kubernetes API，整个体系并不复杂，如下图所示。

![image-20251123154218005](http://images.liangning7.cn/typora/202511231542313.png)

上图中 Webhook Server 指代用户自开发的包含准入控制逻辑的 Web 服务。

动态准入控制器被 API Server 调用时，它会把待校验、修改的 HTTP 请求内容包装成 一个 AdmissionReview 实例发往 Webhook Server。AdmissionReview 是一个瞬时的 Kubernetes API，内部包含了目标请求中的关键信息，供 Webhook Server 中的客户代码做判断之用。而 Webhook Server 也会以 AdmissionReview 的格式给出判断结果。拿到结果后，动态准入控制 器会把用户意志反应到针对 HTTP 请求的响应中，准入控制过程结束。

动态准入机制能调用 Webhook Server 的前提是：

* 知道它的存在，包括在哪里、如何调用。
* 知道它能参与修改和校验的哪个阶段。

一般的准入控制器插件会实现 `admission.Interface` 接口，其中的 `Handles()` 方法可以告知准入控制机制当前这个插件是否可以处理一个到来的请求，这样准入机制可以预先决定要不要在修改和校验阶段调用它。但这一套不能直接套用到动态准入控制器上，因为动态准入控制器只是起中转作用。

这些问题都由 Webhook Server 的配置信息回答。有两个 Kubernetes API 专门用于为动态准入控制器提供配置，它们是：`ValidatingWebhookConfiguration` 和 `MutatingWebhookConfiguration`。它们含有如下信息：

1. 哪些 Webhook Server 存在，名称是什么。

2. Webhook Server 关注的 HTTP 请求匹配规则。

   规则可以基于 Kubernetes API 的属性——例如所在组，也可以是灵活的通用表达语句（CEL）

3. Webhook Server 的访问方式。地址可通过 URL 或集群内的 Service 资源来指定，还可能包含必要的认证设置。

Webhook Server 的部署可以采用集群内部署，也可放到集群外。如果部署到集群内，可以将该服务包装成 Deployment 资源，然后通过一个 Service 资源在集群内暴露它。

### 4.2.2 构造动态准入器插件

两类动态准入控制器也是以插件形式存在的，这部分以 `MutatingAdmissionWebhook` 为例讲解其插件的代码实现。其它内建准入器插件都是以非常类似的过程开发，自然包括 `Validating Webhook`。关键部分代码如下所示。

```go
// 代码: staging/k8s.io/apiserver/pkg/admission/plugin/webhook/mutating/plugin.go
const (
	// PluginName indicates the name of admission plug-in
	PluginName = "MutatingAdmissionWebhook"
)

// Register registers a plugin
func Register(plugins *admission.Plugins) {
	plugins.Register(PluginName, func(configFile io.Reader) (admission.Interface, error) {
		plugin, err := NewMutatingWebhook(configFile)
		if err != nil {
			return nil, err
		}

		return plugin, nil
	})
}

// Plugin is an implementation of admission.Interface.
type Plugin struct {
	*generic.Webhook
}

var _ admission.MutationInterface = &Plugin{}

// NewMutatingWebhook returns a generic admission webhook plugin.
func NewMutatingWebhook(configFile io.Reader) (*Plugin, error) {
	handler := admission.NewHandler(admission.Connect, admission.Create, admission.Delete, admission.Update)
	p := &Plugin{}
	var err error
	p.Webhook, err = generic.NewWebhook(handler, configFile, configuration.NewMutatingWebhookConfigurationManager, newMutatingDispatcher(p))
	if err != nil {
		return nil, err
	}

	return p, nil
}

// ValidateInitialization implements the InitializationValidator interface.
func (a *Plugin) ValidateInitialization() error {
	if err := a.Webhook.ValidateInitialization(); err != nil {
		return err
	}
	return nil
}

// Admit makes an admission decision based on the request attributes.
func (a *Plugin) Admit(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces) error {
	return a.Webhook.Dispatch(ctx, attr, o)
}
```

由名字就可以看出，结构体 Plugin 将用于代表 MutatingAdmissionWebhook，确实如此，它通过嵌套结构体 `generic.Webhook` 获得了后者的众多方法，其中就包括 `admission.MutaionInterface` 要求的 `Handles()` 方法，这使得 Plugin 结构体有资格成为准入控制插件。

最上面的 Register 方法是留给外部调用的接口，准入控制机制会调用它将当前插件加入到插件库。从该方法中可以看到，方法 `NewMutatingWebhook()` 会负责制作该插件的实例，从要点①处开始，该方法首先做一个 handler 变量，指明 MutatingAdmissionWebhook 可以处理 Create、Update、Delete 和 Connect 类 HTTP 请求；然后利用 `generic.NewWebhook` 方法， 基于刚才的 handler 和配置信息——也就是 API Server 中的 MutatingWebhookConfiguration API 实例，以及 `newMutatingDispather()` 方法的返回值制作一个 webhook 存入插件的 Webhook 字段。

要点②处对 `newMutatingDispatcher()` 方法的调用特别重要：它为 Webhook Server 做了一 个分发器 Dispatcher。查看要点③处的 Admit 方法逻辑：当有目标 HTTP 请求到来时，准入控制机制会调用它进行修改操作，而它直接让插件 Webhook 字段把它分发出去，这里的分发便是利用了 Dispatcher。

分发器的代码位于 `staging/k8s.io/apiserver/pkg/admission/plugin/webhook/mutating/dispatcher.go`，复杂程度由其代码长度就能看出。由其源码中可见，分发操作主要发生在 `Dispatch()` 和 `callAttrMutationHook()` 方法

# 5. 一个 HTTP 请求的处理过程

一个 Web Server 所有的服务都是通过处理 HTTP 请求来实现的，如果从 HTTP 请求出发，观察它在 Server 内部经历哪些流转和处理过程、各触及哪些方法，则能更好理解 Generic Server。这一过程如下图所示。

![image-20251123155750186](http://images.liangning7.cn/typora/202511231557559.png)

Generic Server 提供的端点有多种，最主要的是针对 Kubernetes API 的，它们的相对地址以 apis/（针对具有组的 API）或 api/（针对核心 API）开头；其次 Server 还提供辅助性的端点供外界查询，典型的有健康状况检查端点 readyz、livez 和 healthz，不过 healthz 端点在新版本中被前面两个替代。

经过简单转发后，请求被逐个交给过滤链中的过滤器，完成各项基本检测，包括登录、 鉴权、审计、CORS 防护等等。值得留意的是登录和鉴权，它们的触发地是在过滤器中，后续会在 Master Server 中进行讲解。

请求顺利通过过滤器后来到一个关键的分流时刻：handlers 包的 director 结构体会判断当前请求是针对 Kubernetes API 的，还是其它。如果是前者，请求将被转交 go-restful 的转 发器——GoRestfulContainer；如果是后者，请求将交给普通转发器——NonGoRestfulMux。

对于针对系统健康状况查询端点的请求，director 会将其流转给 healthz 包的 handleRootHealth 方法去处理，请求过程随之结束。

对于请求 Kubernetes API 的请求，由 handlers 包下的各个资源处理器去处理。处理过程包含三步：

1. 除了 Get 和 List 请求，先解析出请求体中传递过来的 GVK 信息，形成对应的 Kubernetes API 的 Go 基座结构体实例。这是借助序列化器进行的，Generic Server 提供了三 种解析器应对三种格式：Json 解析器，Yaml 解析器和 Protobuf 解析器。
2. 对解析出的 Kubernetes API 实例，逐个调用准入控制器插件的修改逻辑，之后再 逐个调用校验逻辑，完成准入控制的所有处理。
3. 利用 GVK 的 `rest.Storage` 接口实例，针对请求的 Verb 去进行 CRUD 等操作，结 束后请求响应也随之结束。Generic Server 在 `generic/registry` 包中提供了 Store 结构体，它实现了 `rest.Getter`，`rest.Updater` 等接口，供各个 GVK 在自己的 Storage 结构体上去匿名嵌套从而复用这些实现。
