+++
title = "K8s Aggregation Server Build"
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

series_order = 20

+++

# K8s Aggregation Server Build

聚合 Server 指基于 Generic Server 框架开发的子 API Server，用于引入客制化 API。在Kubernetes 中没有哪一项内置 API 服务由聚合 Server 提供，聚合 Server 是用户专属的扩展方式。社区提供了名为 API Server Builder 的工具辅助聚合 Server 的开发，同时在项目代码库中提供了聚合 Server 的例子，但对于如何开发聚合 Server 并没有详细的文档指导。软件开发绝不应、也不会完全成为黑盒，开发人员需要知其然并知其所以然，以便遇到问题时迅速找到解决方向。

 聚合 Server 的构建与可独立运行的扩展 Server 极为类似，这里不再赘述重复的环节，而是专注讲解其特有部分的实现。

## 聚合 Server 的结构

一个聚合 Server 在结构上与扩展 Server 等非常类似，同样以 Generic Server 为底座构建， 自动具有 Generic Server 所提供的众多能力，例如可以利用 Generic Server 的 `InstallAPIGroup()` 方法将扩展出的 API 注入并生成端点。通过前面的学习对各个 Server 了然于胸，构建聚合 Server 将易如反掌。聚合 Server 的整体架构如图所示，这几乎就是主 Server、扩展 Server 和聚合器的架构翻版。

![image-20251215163135034](http://images.liangning7.cn/typora/202512151631106.png)

核心 Server 启动过程中，聚合器的 `PrepareRun()` 和 `Run()` 方法会被执行，而主 Server 与 扩展 Server 干脆没有这两个方法这是由于聚合器提供了 Server 的基础设施，托起主 Server 和扩展 Server，除了提供各自的 API、钩子函数等配置，它们根本不需要直接面对 Web Server。但聚合 Server 则不然，通常情况下它会被作为一个 Service 单独跑在一个 Pod 里面，是一个可执行程序，它需要自备底层 Server、准备配置信息并启动它。所以当构建聚合 Server 时，开发者会效仿核心 API Server 和其聚合器的做法：运行时首先创建该 Server 的一个实例，该实例会有 `PrepareRun()` 方法，通过调用它完成准备工作，而且 `PrepareRun()` 内会触发对底层 Generic Server 的 `PrepareRun()` 的调用；最后，调用底层 Generic Server 的 `Run()` 方法启动底层 Server。当然，这一过程也与扩展 Server 作为独立应用时的运行过程一致。

细心的读者可能会有疑惑，一个发给聚合 Server 的请求岂不要经过两条请求过滤链？一条是聚合器的，一条是聚合 Server 的，是否多余了？这种冗余无法完全避免，毕竟聚合 Server 是一个独立的 Server，也需要考虑来自核心 API Server 之外的非法请求，请求过滤链中的环节会检验请求。

登录(authentication)和鉴权(authorization)部分值得特别注意。一般情况下聚合 Server 需要和核心 API Server 的处理方式保持一致。试想一下，可不可能出现核心 Server 允许一个用户做操作 API 实例而聚合 Server 不让呢？显然需要保持二者的逻辑一致性，如果出现了这种情况看起来更像不一致。聚合器以及聚合 Server 是协同完成登录和鉴权的过程如图所示：

![image-20251215163803974](http://images.liangning7.cn/typora/202512151638043.png)

## 委派登录认证

委派登录认证是 Generic Server 为聚合 Server 所准备的认证方案，它复合了三种基本的登录认证方式。下面从两种场景中引出这三种基本登录认证。

### 1. 认证转发的请求

对于由核心 API Server 代理转发过来的请求，聚合 Server 启用身份认证代理策略对其做登录认证。这是 Generic Server 内置的一种认证策略，聚合器通过反向代理转发请求时，它会在请求头添加对该请求的认证结果。有两个相关 Header：

1. X-Remote-User （名称可配置）: 聚合器认证后的用户名；
2. X-Remote-Group （名称可配置）: 聚合器认证后的用户组。

问题是聚合 Server 如何确认带有上述 Header 的请求来自聚合器，而不是非法第三方，这就需要证书来保证链接的安全了，如下图所示。进行请求转发时，所有反向代理服务可使用一张 X509 证书与聚合 Server 建立安全链接，该证书所用 CN 必须为 “aggregator”（启动时可更改）。在核心 API Server 启动时，需要使用命令行标志 `--proxy-client-cert-file` 和 `--proxy-client-key-file` 来指定这张证书及私钥；与这张证书相关的根证书则以 `--requestheader-client-ca-file` 标志指定。这些证书将会以 `ConfigMap`的形式，保存在核心 API Server 上，聚合 Server 从核心 API Server 读取它。据此，聚合 Server 校验请求所使用的证书，如果通过就完全信任请求头上的用户名和用户组信息。

![image-20251215164101100](http://images.liangning7.cn/typora/202512151641152.png)

### 2. 认证非转发的请求

对于不是从核心 API Server 来的请求，聚合 Server 可以使用 Generic Server 所提供的任何一种登录认证策略，但 Generic Server 推荐启用下面两种：

1. X509 客户证书策略。
2. TokenReview 策略（一种 webhook 登录认证）。这种策略的工作过程是：如果在请求头中有 `Authorization: Bearer` ，则通过 webhook 向核心 API Server 创建 TokenReview API 实例，核心 API Server 会立刻给予确认，这样聚合 Server 即刻获知用户的合法性。进行认证的也可以不是核心 API Server，这种情况下在启动聚合 Server 时，用参数 `--authentication-kubeconfig` 指出认证服务器的连接信息即可，当然这需要目标认证服务器能够处理 TokenReview 实例。

### 3. 委派登录认证

以上涉及了多种登录认证策略，Generic Server 推荐三种策略：身份认证代理、X509 客户证书和 TokeReview 策略。在构建聚合 Server 时如何同时启用三者呢？Generic Server 也已经准备好可复用方案，它生成复合以上三个登录认证策略的新策略，称为委派登录认证。

下面分析委派登录认证的源码。

由核心 API Server 的认证策略构建可知，一切从生成 Option 开始，Option 决定了命令行有哪些参数可供用户使用，用户的命令行输入会改变 Option 的原始默认值；程序以 Option 为基础生成 Server 运行配置（Config）；配置最终应用到生成的 Server 实例上。而登录认证策略的构建一样从 Option 开始，聚合 Server 的 Option 分两个部分：第一属于底座 Generic Server 的，第二自己特有的。登录认证策略是在 Generic Server 的 Option 上设置的，Generic Server 库通过函数 NewRecommendedOptions()推荐了 Option，其上包含推荐的登录认证策略设置。聚合 Server 只需在构建时用该方法构建 Generic Server 的 Option 即可。`NewRecommendedOptions()` 函数代码如下所示：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\options\recommended.go#L55-L81
func NewRecommendedOptions(prefix string, codec runtime.Codec) *RecommendedOptions {
	sso := NewSecureServingOptions()

	// We are composing recommended options for an aggregated api-server,
	// whose client is typically a proxy multiplexing many operations ---
	// notably including long-running ones --- into one HTTP/2 connection
	// into this server.  So allow many concurrent operations.
	sso.HTTP2MaxStreamsPerConnection = 1000

	return &RecommendedOptions{
		Etcd:           NewEtcdOptions(storagebackend.NewDefaultConfig(prefix, codec)),
		SecureServing:  sso.WithLoopback(),
		Authentication: NewDelegatingAuthenticationOptions(), //要点①
		Authorization:  NewDelegatingAuthorizationOptions(),
		Audit:          NewAuditOptions(),
		Features:       NewFeatureOptions(),
		CoreAPI:        NewCoreAPIOptions(),
		// Wired a global by default that sadly people will abuse to have different meanings in different repos.
		// Please consider creating your own FeatureGate so you can have a consistent meaning for what a variable contains
		// across different repos.  Future you will thank you.
		FeatureGate:                feature.DefaultFeatureGate,
		ExtraAdmissionInitializers: func(c *server.RecommendedConfig) ([]admission.PluginInitializer, error) { return nil, nil },
		Admission:                  NewAdmissionOptions(),
		EgressSelector:             NewEgressSelectorOptions(),
		Traces:                     NewTracingOptions(),
	}
}
```

上述代码要点①处生成了委派登录认证的 Option。

有了 Option，下一步就是生成 Server 配置。基于要点①处函数调用所生成的 Option，可以制作一个类型为 `DelegatingAuthenticatorConfig` 结构体的实例，利用它的 `New()` 方法将得到一个登录验证策略——这便是复合了身份代理认证策略、X509 客户证书策略和 TokenReview 策略的**委派登录认证**，可作为聚合 Server 的登录认证策略。

### 4. 委派权限认证

权限的信息被核心 API Server 集中管理，Kubernetes 集群常用 Role-Based-Access-Control （RBAC）来作为权限控制的方式。本文以 RBAC 为权限管理方式进行讲解，其它方式类似。在这种模式下，Role 这种 API 用于设定一个角色具有什么权限，一个 Role 实例如图所示。

![image-20251215164627378](http://images.liangning7.cn/typora/202512151646437.png)

API RoleBinding 则把 Role 实例和一个 User 或一组 User 绑定在一起，一个 RoleBinding 实例如图所示。

![image-20251215164702664](http://images.liangning7.cn/typora/202512151647738.png)

聚合 Server 需要知道请求的用户是否具有做某个操作的权限时，它需要联络核心 API Server 进行确认，这一确认请求被包装成 API SubjectReviewReview 的实例，聚合 Server 利用一个 webhook 向核心 API Server 发起创建请求，该请求最终会被核心 API Server 立即响应，响应代码如下：

```go
// 代码: pkg\registry\authorization\subjectaccessreview\rest.go#L63-L96
func (r *REST) Create(ctx context.Context, obj runtime.Object, createValidation rest.ValidateObjectFunc, options *metav1.CreateOptions) (runtime.Object, error) {
	subjectAccessReview, ok := obj.(*authorizationapi.SubjectAccessReview)
	if !ok {
		return nil, apierrors.NewBadRequest(fmt.Sprintf("not a SubjectAccessReview: %#v", obj))
	}
	// clear fields if the featuregate is disabled
	if !utilfeature.DefaultFeatureGate.Enabled(genericfeatures.AuthorizeWithSelectors) {
		if subjectAccessReview.Spec.ResourceAttributes != nil {
			subjectAccessReview.Spec.ResourceAttributes.FieldSelector = nil
			subjectAccessReview.Spec.ResourceAttributes.LabelSelector = nil
		}
	}
	if errs := authorizationvalidation.ValidateSubjectAccessReview(subjectAccessReview); len(errs) > 0 {
		return nil, apierrors.NewInvalid(authorizationapi.Kind(subjectAccessReview.Kind), "", errs)
	}

	if createValidation != nil {
		if err := createValidation(ctx, obj.DeepCopyObject()); err != nil {
			return nil, err
		}
	}

	authorizationAttributes := authorizationutil.AuthorizationAttributesFrom(subjectAccessReview.Spec)
	decision, reason, evaluationErr := r.authorizer.Authorize(ctx, authorizationAttributes)

    // 要点①
	subjectAccessReview.Status = authorizationapi.SubjectAccessReviewStatus{
		Allowed: (decision == authorizer.DecisionAllow),
		Denied:  (decision == authorizer.DecisionDeny),
		Reason:  reason,
	}
	subjectAccessReview.Status.EvaluationError = authorizationutil.BuildEvaluationError(evaluationErr, authorizationAttributes)

	return subjectAccessReview, nil
}
```

上述代码要点①处给出了鉴权结果，它利用了核心 API Server 所配置的 authorizer 属性，这样是否有权限做一个操作的决定最终是由核心 API Server 的鉴权机制做出，保证了一致性。

以上就是聚合 Server 权限鉴定的过程，下面讲解聚合 Server 如何建立委派鉴权机制。

回顾核心 API Server 的鉴权器生成过程可知：Server 实例由 Server 配置生成，Server 配置来自 Option，一切从生成 Option 开始，鉴权器也是如此。聚合 Server 鉴权器设置完全类似。Generic Server 为委派鉴权器相关的 Option 准备了一个工厂方法：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\options\authorization.go#L78-L94
func NewDelegatingAuthorizationOptions() *DelegatingAuthorizationOptions {
	return &DelegatingAuthorizationOptions{
		// very low for responsiveness, but high enough to handle storms
		AllowCacheTTL:       10 * time.Second,
		DenyCacheTTL:        10 * time.Second,
		ClientTimeout:       10 * time.Second,
		WebhookRetryBackoff: DefaultAuthWebhookRetryBackoff(),
		// This allows the kubelet to always get health and readiness without causing an authorization check.
		// This field can be cleared by callers if they don't want this behavior.
		AlwaysAllowPaths: []string{"/healthz", "/readyz", "/livez"},
		// In an authorization call delegated to a kube-apiserver (the expected common-case), system:masters has full
		// authority in a hard-coded authorizer.  This means that our default can reasonably be to skip an authorization
		// check for system:masters.
		// This field can be cleared by callers if they don't want this behavior.
		AlwaysAllowGroups: []string{"system:masters"},
	}
}
```

聚合 Server 在创建 Option 时，可以直接使用它获取鉴权器相关的 Option，作为其底层 Generic Server 的鉴权配置，这样便启用了委派鉴权器。还可以再简单点：为聚合 Server 的底层 Generic Server 生成 Option 时，直接借用 Generic Server 的推荐 Option，它默认已经使用了委派鉴权器，其工厂方法如下代码所示。大多数的聚合 Server 都会这么做，毕竟在 上述推荐 Option 的基础上进行更改调整会更省力一些。有了委派鉴权器的 Option 就有了生成鉴权器的基础，再深挖一步，看 Generic Server 是怎么在此基础上构造委派鉴权器的。相关代码如下：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\options\authorization.go#L177-L209
func (s *DelegatingAuthorizationOptions) toAuthorizer(client kubernetes.Interface) (authorizer.Authorizer, error) {
	var authorizers []authorizer.Authorizer

	if len(s.AlwaysAllowGroups) > 0 {
		authorizers = append(authorizers, authorizerfactory.NewPrivilegedGroups(s.AlwaysAllowGroups...))
	}

	if len(s.AlwaysAllowPaths) > 0 {
		a, err := path.NewAuthorizer(s.AlwaysAllowPaths)
		if err != nil {
			return nil, err
		}
		authorizers = append(authorizers, a)
	}

	if client == nil {
		klog.Warning("No authorization-kubeconfig provided, so SubjectAccessReview of authorization tokens won't work.")
	} else {
		cfg := authorizerfactory.DelegatingAuthorizerConfig{ //要点①
			SubjectAccessReviewClient: client.AuthorizationV1(),
			AllowCacheTTL:             s.AllowCacheTTL,
			DenyCacheTTL:              s.DenyCacheTTL,
			WebhookRetryBackoff:       s.WebhookRetryBackoff,
		}
		delegatedAuthorizer, err := cfg.New()
		if err != nil {
			return nil, err
		}
		authorizers = append(authorizers, delegatedAuthorizer)
	}

	return union.New(authorizers...), nil
}
```

Option 里面的 AlwaysAllowPaths 会有设置不受权限保护的路径，例如健康监测端点， `toAuthorizer()` 方法首先为它们单独制作子鉴权器；然后制作 Webhook 鉴权器：它先在要点①处从 Option 生成 Config，然后用 Config 的 `New()` 方法生成该鉴权器。最后，所有这些子 鉴权器被方法 `union.New` 组合起来，作为单独鉴权器——委派鉴权器返回。`cfg.New()` 方法代码如下：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\authorization\authorizerfactory\delegating.go#L51-L69
func (c DelegatingAuthorizerConfig) New() (authorizer.Authorizer, error) {
	if c.WebhookRetryBackoff == nil {
		return nil, errors.New("retry backoff parameters for delegating authorization webhook has not been specified")
	}
	compiler := c.Compiler
	if compiler == nil {
		compiler = authorizationcel.NewDefaultCompiler()
	}

	return webhook.NewFromInterface(
		c.SubjectAccessReviewClient,
		c.AllowCacheTTL,
		c.DenyCacheTTL,
		*c.WebhookRetryBackoff,
		authorizer.DecisionNoOpinion,
		NewDelegatingAuthorizerMetrics(),
		compiler,
	)
}
```

`New()` 方法内会调用 webhook 包的 `NewFromInterface()` 函数来生成 webhook 鉴权器，核心 API Server 处于聚合 Server 的远程，所以这里需用 webhook 来包装对核心 API Server 的 访问，`NewFromInterface()` 方法最终返回一个 WebhookAuthorizer 结构体实例，其上有 `Authorize()` 方法用于执行 SubjectAccessReview 的创建和结果检查等。

至此，Generic Server 为聚合 Server 准备的委派鉴权器构建完毕。
