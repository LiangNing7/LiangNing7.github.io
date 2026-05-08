+++
title = "K8s Extensions Server Implementation"
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

series_order = 15

+++

# K8s Extensions Server Implementation

与主 Server 相同，扩展 Server 也是以 Generic Server 为底座，二者关系如下图所示。

![image-20251211172916897](http://images.liangning7.cn/typora/202512111729995.png)

Generic Server 提供可复用的基础能力。准入控制、过滤器链、基于 go-restful 的 RESTful 设施和登录鉴权机制是扩展 Server 特别依赖的。有坚实的基础，扩展 Server 只需将自身业务相关的内容注入其中便可。API Server 中万事皆 API，所以它的内容就是扩展业务相关的 API，这又包含两部分：第一是 CRD，第二是 CRD 实例定义出的客制化 API。

## 准备 Server 运行配置

构造扩展 Server 前要先得到其配置信息，来从下面的数据流程看看 Extensions Server 的数据流程是怎样的。

```markdown
┌─────────────────────────────────────────────────────────────────────────────┐
│                  起点：Run() 函数                                             │
│                  cmd/kube-apiserver/app/server.go:154                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  func Run(ctx context.Context, opts options.CompletedOptions) error {      │
│      // 🔥 第一步：创建配置                                                  │
│      config, err := NewConfig(opts)                                        │
│      //                  👆                                                │
│      //                  传入命令行选项                                     │
│                                                                             │
│      completed, err := config.Complete()                                   │
│      server, err := CreateServerChain(completed)                           │
│      prepared, err := server.PrepareRun()                                  │
│      return prepared.Run(ctx)                                              │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                  第一步：NewConfig() 创建总配置                               │
│                  cmd/kube-apiserver/app/config.go:74                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  func NewConfig(opts options.CompletedOptions) (*Config, error) {          │
│      c := &Config{                                                         │
│          Options: opts,  // 保存命令行选项                                  │
│      }                                                                      │
│                                                                             │
│      // 🔥 步骤 1.1：构建 Generic Config（通用配置）                         │
│      genericConfig, versionedInformers, storageFactory, err :=             │
│          controlplaneapiserver.BuildGenericConfig(                         │
│              opts.CompletedOptions,                                        │
│              []*runtime.Scheme{                                            │
│                  legacyscheme.Scheme,           // Kube 核心资源            │
│                  apiextensionsapiserver.Scheme, // CRD 资源                │
│                  aggregatorscheme.Scheme,       // AA 资源                 │
│              },                                                            │
│              controlplane.DefaultAPIResourceConfigSource(),                │
│              generatedopenapi.GetOpenAPIDefinitions,                       │
│          )                                                                 │
│      // 返回：                                                              │
│      //   - genericConfig: 通用 API Server 配置（认证、授权、审计等）        │
│      //   - versionedInformers: 共享 Informer 工厂                         │
│      //   - storageFactory: 存储工厂（etcd 配置）                           │
│                                                                             │
│      // 继续下一步...                                                       │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                  第二步：CreateKubeAPIServerConfig()                         │
│                  cmd/kube-apiserver/app/config.go:89                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  func NewConfig(opts options.CompletedOptions) (*Config, error) {          │
│      // ... 接上一步                                                        │
│                                                                             │
│      // 🔥 步骤 1.2：创建 Kube API Server 配置                              │
│      kubeAPIs, serviceResolver, pluginInitializer, err :=                  │
│          CreateKubeAPIServerConfig(                                        │
│              opts,                  // 命令行选项                           │
│              genericConfig,         // 通用配置（步骤 1.1 的输出）           │
│              versionedInformers,    // Informer 工厂                        │
│              storageFactory,        // 存储工厂                             │
│          )                                                                 │
│      // 返回：                                                              │
│      //   - kubeAPIs: Kube API Server 配置                                 │
│      //       └─ kubeAPIs.ControlPlane.Generic: Generic Config             │
│      //       └─ kubeAPIs.ControlPlane.VersionedInformers: Informers       │
│      //       └─ kubeAPIs.ControlPlane.ProxyTransport: 代理传输             │
│      //   - serviceResolver: 服务解析器                                     │
│      //   - pluginInitializer: 准入插件初始化器                             │
│                                                                             │
│      c.KubeAPIs = kubeAPIs  // 保存到 Config                               │
│                                                                             │
│      // 继续下一步...                                                       │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                  第三步：CreateAPIExtensionsConfig() 👈 关键！                │
│                  cmd/kube-apiserver/app/config.go:95                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  func NewConfig(opts options.CompletedOptions) (*Config, error) {          │
│      // ... 接上一步                                                        │
│                                                                             │
│      // 🔥🔥🔥 步骤 1.3：创建 API Extensions Server 配置                    │
│      apiExtensions, err := controlplaneapiserver.CreateAPIExtensionsConfig(│
│          //                 👆                                             │
│          //                 调用 pkg/controlplane/apiserver/apiextensions.go│
│                                                                             │
│          // 参数 1：Generic Config（从 kubeAPIs 中提取）                    │
│          *kubeAPIs.ControlPlane.Generic,                                   │
│          //  👆 包含：认证、授权、审计、etcd、序列化器等通用配置              │
│                                                                             │
│          // 参数 2：Informer 工厂（从 kubeAPIs 中提取）                     │
│          kubeAPIs.ControlPlane.VersionedInformers,                         │
│          //  👆 共享 Informer，监听 Kube 资源变化                           │
│                                                                             │
│          // 参数 3：准入插件初始化器（从步骤 1.2 返回）                      │
│          pluginInitializer,                                                │
│          //  👆 用于初始化准入控制插件                                       │
│                                                                             │
│          // 参数 4：命令行选项（原始输入）                                   │
│          opts.CompletedOptions,                                            │
│          //  👆 包含 etcd、API 启用等配置                                   │
│                                                                             │
│          // 参数 5：Master 节点数（从 opts 中提取）                          │
│          opts.MasterCount,                                                 │
│          //  👆 用于 HA 检测                                                │
│                                                                             │
│          // 参数 6：服务解析器（从步骤 1.2 返回）                            │
│          serviceResolver,                                                  │
│          //  👆 解析 Webhook Service 名称为地址                             │
│                                                                             │
│          // 参数 7：认证解析器包装器（现场创建）                             │
│          webhook.NewDefaultAuthenticationInfoResolverWrapper(              │
│              kubeAPIs.ControlPlane.ProxyTransport,      // 代理传输         │
│              kubeAPIs.ControlPlane.Generic.EgressSelector, // 出口选择器    │
│              kubeAPIs.ControlPlane.Generic.LoopbackClientConfig, // 回环配置│
│              kubeAPIs.ControlPlane.Generic.TracerProvider, // 追踪提供者    │
│          ),                                                                │
│          //  👆 为 Webhook 提供认证信息                                     │
│      )                                                                      │
│                                                                             │
│      c.ApiExtensions = apiExtensions  // 保存到 Config                     │
│                                                                             │
│      // 继续下一步...                                                       │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                  第四步：进入 CreateAPIExtensionsConfig()                    │
│                  pkg/controlplane/apiserver/apiextensions.go:34             │
├─────────────────────────────────────────────────────────────────────────────┤
│  func CreateAPIExtensionsConfig(                                           │
│      kubeAPIServerConfig server.Config,                 // 👈 参数 1        │
│      kubeInformers informers.SharedInformerFactory,     // 👈 参数 2        │
│      pluginInitializers []admission.PluginInitializer,  // 👈 参数 3        │
│      commandOptions options.CompletedOptions,           // 👈 参数 4        │
│      masterCount int,                                   // 👈 参数 5        │
│      serviceResolver webhook.ServiceResolver,           // 👈 参数 6        │
│      authResolverWrapper webhook.AuthenticationInfoResolverWrapper, // 👈 7 │
│  ) (*apiextensionsapiserver.Config, error) {                               │
│                                                                             │
│      // 步骤 4.1：复制并清理 Generic Config                                 │
│      genericConfig := kubeAPIServerConfig                                  │
│      genericConfig.PostStartHooks = map[string]...{}                       │
│      genericConfig.RESTOptionsGetter = nil                                 │
│                                                                             │
│      // 步骤 4.2：配置 etcd 存储                                            │
│      etcdOptions := *commandOptions.Etcd                                   │
│      etcdOptions.StorageConfig.Codec = ...                                 │
│      etcdOptions.StorageConfig.EncodeVersioner = ...                       │
│      etcdOptions.ApplyTo(&genericConfig)                                   │
│                                                                             │
│      // 步骤 4.3：配置 API 启用                                             │
│      commandOptions.APIEnablement.ApplyTo(&genericConfig, ...)             │
│                                                                             │
│      // 步骤 4.4：组装最终配置                                              │
│      apiextensionsConfig := &apiextensionsapiserver.Config{                │
│          GenericConfig: &server.RecommendedConfig{                         │
│              Config:                genericConfig,                         │
│              SharedInformerFactory: kubeInformers,                         │
│          },                                                                │
│          ExtraConfig: apiextensionsapiserver.ExtraConfig{                  │
│              CRDRESTOptionsGetter: NewCRDRESTOptionsGetter(...),           │
│              MasterCount:          masterCount,                            │
│              AuthResolverWrapper:  authResolverWrapper,                    │
│              ServiceResolver:      serviceResolver,                        │
│          },                                                                │
│      }                                                                     │
│                                                                             │
│      return apiextensionsConfig, nil                                       │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                  第五步：返回到 NewConfig()                                   │
│                  cmd/kube-apiserver/app/config.go:101                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  func NewConfig(opts options.CompletedOptions) (*Config, error) {          │
│      // ... 接步骤 1.3                                                      │
│                                                                             │
│      c.ApiExtensions = apiExtensions  // 保存 API Extensions 配置          │
│                                                                             │
│      // 🔥 步骤 1.4：创建 Aggregator Server 配置                            │
│      aggregator, err := controlplaneapiserver.CreateAggregatorConfig(      │
│          *kubeAPIs.ControlPlane.Generic,                                   │
│          opts.CompletedOptions,                                            │
│          kubeAPIs.ControlPlane.VersionedInformers,                         │
│          serviceResolver,                                                  │
│          kubeAPIs.ControlPlane.ProxyTransport,                             │
│          kubeAPIs.ControlPlane.Extra.PeerProxy,                            │
│          pluginInitializer,                                                │
│      )                                                                      │
│      c.Aggregator = aggregator  // 保存 Aggregator 配置                    │
│                                                                             │
│      return c, nil  // 🎯 返回完整配置                                      │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                  最终结果：Config 结构体                                      │
│                  cmd/kube-apiserver/app/config.go:33                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  type Config struct {                                                      │
│      Options options.CompletedOptions  // 命令行选项                        │
│                                                                             │
│      Aggregator    *aggregatorapiserver.Config       // 聚合层配置          │
│      KubeAPIs      *controlplane.Config              // 核心 API 配置       │
│      ApiExtensions *apiextensionsapiserver.Config    // 扩展 API 配置 👈    │
│                                                                             │
│      ExtraConfig                                                           │
│  }                                                                         │
│                                                                             │
│  // 三个配置的关系：                                                         │
│  // 1. 都基于同一个 Generic Config（通用配置）                              │
│  // 2. 共享同一个 Informer 工厂                                             │
│  // 3. 使用相同的 serviceResolver 和 pluginInitializer                     │
│  // 4. 各自有专属的 ExtraConfig                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

除去 Generic Server 有的配置外，扩展 API Server 主要完成 CRD、客制化API，那么扩展 Server 还需要哪些配置呢，可以在 `CreateAPIExtensionsConfig` 函数中找到答案：

```go
// 代码: pkg\controlplane\apiserver\apiextensions.go#L34-L86
func CreateAPIExtensionsConfig(
	kubeAPIServerConfig server.Config,
	kubeInformers informers.SharedInformerFactory,
	pluginInitializers []admission.PluginInitializer,
	commandOptions options.CompletedOptions,
	masterCount int,
	serviceResolver webhook.ServiceResolver,
	authResolverWrapper webhook.AuthenticationInfoResolverWrapper,
) (*apiextensionsapiserver.Config, error) {
	// make a shallow copy to let us twiddle a few things
	// most of the config actually remains the same.  We only need to mess with a couple items related to the particulars of the apiextensions
    // 要点①
	genericConfig := kubeAPIServerConfig
	genericConfig.PostStartHooks = map[string]server.PostStartHookConfigEntry{}
	genericConfig.RESTOptionsGetter = nil

	// copy the etcd options so we don't mutate originals.
	// we assume that the etcd options have been completed already.  avoid messing with anything outside
	// of changes to StorageConfig as that may lead to unexpected behavior when the options are applied.
	etcdOptions := *commandOptions.Etcd
	// this is where the true decodable levels come from.
	etcdOptions.StorageConfig.Codec = apiextensionsapiserver.Codecs.LegacyCodec(v1beta1.SchemeGroupVersion, v1.SchemeGroupVersion)
	// prefer the more compact serialization (v1beta1) for storage until https://issue.k8s.io/82292 is resolved for objects whose v1 serialization is too big but whose v1beta1 serialization can be stored
	etcdOptions.StorageConfig.EncodeVersioner = runtime.NewMultiGroupVersioner(v1beta1.SchemeGroupVersion, schema.GroupKind{Group: v1beta1.GroupName})
	etcdOptions.SkipHealthEndpoints = true // avoid double wiring of health checks
	if err := etcdOptions.ApplyTo(&genericConfig); err != nil {
		return nil, err
	}

	// override MergedResourceConfig with apiextensions defaults and registry
	if err := commandOptions.APIEnablement.ApplyTo(
		&genericConfig,
		apiextensionsapiserver.DefaultAPIResourceConfigSource(),
		apiextensionsapiserver.Scheme); err != nil {
		return nil, err
	}
	apiextensionsConfig := &apiextensionsapiserver.Config{     // 要点②
		GenericConfig: &server.RecommendedConfig{	// 要点③
			Config:                genericConfig,
			SharedInformerFactory: kubeInformers,
		},
		ExtraConfig: apiextensionsapiserver.ExtraConfig{	// 要点④
			CRDRESTOptionsGetter: apiextensionsoptions.NewCRDRESTOptionsGetter(etcdOptions, genericConfig.ResourceTransformers, genericConfig.StorageObjectCountTracker),
			MasterCount:          masterCount,
			AuthResolverWrapper:  authResolverWrapper,
			ServiceResolver:      serviceResolver,
		},
	}

	// we need to clear the poststarthooks so we don't add them multiple times to all the servers (that fails)
	apiextensionsConfig.GenericConfig.PostStartHooks = map[string]server.PostStartHookConfigEntry{}

	return apiextensionsConfig, nil
}
```

1. 首先从要点①出的 kube-apiserver 的配置浅拷贝一份基础配置，因为扩展 Server 的底座也是 Generic Server；
2. 在要点②处构造扩展 Server 的配置，不仅传入了要点③的 Generic Server 配置，还传入了要点④处的额外配置。
   * `CRDRESTOptionsGetter`：为 CRD 提供 REST 存储选项，用于创建 CRD 的动态 REST 存储时使用。
     * 配置 CRD 如何存储到 etcd；
     * 处理 CRD 的序列化/反序列化（支持 CBOR 和 JSON）；
     * 管理存储转换器和对象计数追踪。
   * `MasterCount`：Master 节点数量，用于 CRD 状态控制器判断集群模式。
     * 检测集群是否为 HA（高可用）模式；
     * 如果是 HA 集群，CRD Establishing 状态会延迟 5 秒。
   * `ServiceResolver`：服务解析器，用于 CRD 的 Conversion Webhook 调用。
     * 在 CRD webhook 转换器中解析 webhook 的 Service 名称
     * 将 Service 名称转换为实际的网络地址
   * `AuthResolverWrapper`：认证信息解析器包装器，用于 CRD 的 Conversion Webhook 认证。
     * 在 CRD webhook 转换器中处理认证；
     * 为 webhook 调用提供认证信息。

## 创建扩展 Server

创建的流程图如下：

```markdown
┌─────────────────────────────────────────────────────────────────────────────┐
│                  起点：Run() 函数                                             │
│                  cmd/kube-apiserver/app/server.go:148                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  func Run(ctx context.Context, opts options.CompletedOptions) error {      │
│      ...                                                                   │
│      config, err := NewConfig(opts)                                        │
│      completed, err := config.Complete()                                   │
│      server, err := CreateServerChain(completed)  // 👈 关键                │
│      ...                                                                   │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                  CreateServerChain() - 创建服务器链                           │
│                  cmd/kube-apiserver/app/server.go:176                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  func CreateServerChain(config CompletedConfig) (*APIAggregator, error) {  │
│      notFoundHandler := notfoundhandler.New(...)                           │
│                                                                             │
│      // 🔥 调用扩展 API Server 的 New 方法                                   │
│      apiExtensionsServer, err := config.ApiExtensions.New(                 │
│          genericapiserver.NewEmptyDelegateWithCustomHandler(notFoundHandler)│
│      )                                                                      │
│      // 👆 到达目标：apiserver.go:127                                       │
│                                                                             │
│      kubeAPIServer, err := config.KubeAPIs.New(...)                        │
│      aggregatorServer, err := CreateAggregatorServer(...)                  │
│      return aggregatorServer, nil                                          │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                  目标：apiExtensions.New()                                   │
│                  staging/src/k8s.io/apiextensions-apiserver/pkg/apiserver/ │
│                  apiserver.go:127                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  func (c completedConfig) New(delegationTarget DelegationTarget)           │
│      (*CustomResourceDefinitions, error) {                                 │
│      ...                                                                   │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

`New()` 方法内容非常关键，仔细查看其源码，它主要完成了三项工作

1. 把 CRD 这个 Kubernetes API 通过 Generic Server 提供的 `InstallAPIGroup()` 方法注 入 Server 从而生成 API 端点。
2. 为通过 CRD 定义出的客制化 API 制作 HTTP 请求响应方法。 
3. 通过控制器监听 CRD 实例的创建，当有新 CRD 时，更新客制化 API 组等信息。

其间还穿插完成较琐碎事项，例如确保所有 HTTP 端点就续前客户端发来的请求将得到 HTTP 503 而不是 404。

本节剩余部分分三个小节详细讲解上述三个主要步骤。

但在这之前， 和介绍主 Server 时类似先探究一下扩展 Server 的注册表如何填充，这是创建 Server 的必备条件。

### 1. 准备工作：填充注册表

注册表在 API Server 程序中作用重要，它的填充不可忽略。扩展 Server 的注册表填充遵循了主 Server 类似的过程，其具有的内置 API 组会有 install 包去注册组内 API 信息。扩展 Server 只具有一个内置 API 组——apiextensions，所以注册过程也被简化了。

apiextensions 组的 install 包内只有一个源文件 install.go，其源代码如下：

```go
// 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\apis\apiextensions\install\install.go#L17-L33
package install

import (
	"k8s.io/apiextensions-apiserver/pkg/apis/apiextensions"
	v1 "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1"
	"k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1"
	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
)

// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
	utilruntime.Must(apiextensions.AddToScheme(scheme))
	utilruntime.Must(v1beta1.AddToScheme(scheme))
	utilruntime.Must(v1.AddToScheme(scheme))
	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion, v1beta1.SchemeGroupVersion))
}
```

它定义了一个 `Install()` 函数，但没有像主 Server 的内置 API 组那样，在 install 包的初始化方法内去调用该函数，

> 主 Server 的 install 包：
>
> ```go
> // 代码: pkg\apis\core\install\install.go#L19-L38
> package install
> 
> import (
> 	"k8s.io/apimachinery/pkg/runtime"
> 	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
> 	"k8s.io/kubernetes/pkg/api/legacyscheme"
> 	"k8s.io/kubernetes/pkg/apis/core"
> 	"k8s.io/kubernetes/pkg/apis/core/v1"
> )
> 
> func init() {
> 	Install(legacyscheme.Scheme)
> }
> 
> // Install registers the API group and adds types to a scheme
> func Install(scheme *runtime.Scheme) {
> 	utilruntime.Must(core.AddToScheme(scheme))
> 	utilruntime.Must(v1.AddToScheme(scheme))
> 	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion))
> }
> 
> ```

而是在扩展 Server 的 apiserver 包初始化函数 `init()` 中调用，代码如下所示：

```go
// 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\apiserver\apiserver.go#L54-L77
var (
	Scheme = runtime.NewScheme() //要点①
	Codecs = serializer.NewCodecFactory(Scheme)

	// if you modify this, make sure you update the crEncoder
	unversionedVersion = schema.GroupVersion{Group: "", Version: "v1"}
	unversionedTypes   = []runtime.Object{
		&metav1.Status{},
		&metav1.WatchEvent{},
		&metav1.APIVersions{},
		&metav1.APIGroupList{},
		&metav1.APIGroup{},
		&metav1.APIResourceList{},
	}
)

func init() {
	install.Install(Scheme) //要点②

	// we need to add the options to empty v1
	metav1.AddToGroupVersion(Scheme, schema.GroupVersion{Group: "", Version: "v1"})

	Scheme.AddUnversionedTypes(unversionedVersion, unversionedTypes...)
}
```

上述代码中，在要点②处调用了 Install 方法，apiserver 包也是 `New()` 方法所在的包，在包 `init()` 函数内进行调用确保 `New()` 方法被调用前注册表填充已经完成。

> NOTICE:
>
> 扩展 Server 的注册表实例是要点①处新建出的，和主 Server 并不是同一个。

### 2. CRD API 注入

注入的目的是让 Generic Server 为内置 API 暴露 RESTful 端点。老样子，注册是以 API 组为单位进行的，扩展 Server 只需把自己的 API 组包装成 `genericserver.APIGroupInfo` 结构体实例，去调用接口方法就好了。由于扩展 Server 只有一个 API 组 apiextensions，其内也只有一个 API `CustomResourceDefinition`，注入过程很简单。

> NOTE:
>
> 回忆一下 Main Server 中的 APIGroupInfo 是如何构建的？
>
> ```go
> // 1. 定义多个 RESTStorageProvider
> restStorageProviders := []RESTStorageProvider{
>     corerest.GenericConfig{},
>     authenticationrest.RESTStorageProvider{},
>     authorizationrest.RESTStorageProvider{},
>     // ... 更多 providers
> }
> 
> // 2. 遍历每个 provider，调用 NewRESTStorage 构建 APIGroupInfo
> for _, restStorageBuilder := range restStorageProviders {
>     apiGroupInfo, err := restStorageBuilder.NewRESTStorage(...)
>     // ...
> }
> 
> // 3. 安装 APIGroupInfo
> s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo)
> ```

```go
// 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\apiserver\apiserver.go#L145-163
// New returns a new instance of CustomResourceDefinitions from the given config.
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*CustomResourceDefinitions, error) {
	genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}

	// hasCRDInformerSyncedSignal is closed when the CRD informer this server uses has been fully synchronized.
	// It ensures that requests to potential custom resource endpoints while the server hasn't installed all known HTTP paths get a 503 error instead of a 404
	hasCRDInformerSyncedSignal := make(chan struct{})
	if err := genericServer.RegisterMuxAndDiscoveryCompleteSignal("CRDInformerHasNotSynced", hasCRDInformerSyncedSignal); err != nil {
		return nil, err
	}

	s := &CustomResourceDefinitions{
		GenericAPIServer: genericServer,
	}

	apiResourceConfig := c.GenericConfig.MergedResourceConfig
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apiextensions.GroupName, Scheme, metav1.ParameterCodec, Codecs)
	storage := map[string]rest.Storage{}
	// customresourcedefinitions
	if resource := "customresourcedefinitions"; apiResourceConfig.ResourceEnabled(v1.SchemeGroupVersion.WithResource(resource)) {
		customResourceDefinitionStorage, err := customresourcedefinition.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
		if err != nil {
			return nil, err
		}
		storage[resource] = customResourceDefinitionStorage
		storage[resource+"/status"] = customresourcedefinition.NewStatusREST(Scheme, customResourceDefinitionStorage)
	}
	if len(storage) > 0 {
		apiGroupInfo.VersionedResourcesStorageMap[v1.SchemeGroupVersion.Version] = storage
	}

	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
		return nil, err
	}
    ...
}
```

1. 基于通用配置创建一个名为 apiextensions-apiserver 的 GenericAPIServer 实例，delegationTarget 是委托链中的下一个 server（用于请求转发）。

   ```go
   genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
   ```

2. 创建一个 channel 作为信号量。作用是：在 CRD Informer 完全同步之前，对自定义资源端点的请求会返回 503（服务不可用）而不是 404（未找到）。这避免了客户端误以为资源不存在。

   ```go
   hasCRDInformerSyncedSignal := make(chan struct{})
   if err := genericServer.RegisterMuxAndDiscoveryCompleteSignal("CRDInformerHasNotSynced", hasCRDInformerSyncedSignal);
   ```

3. 包装 GenericAPIServer，作为 apiextensions-apiserver 的主结构体。

   ```go
   s := &CustomResourceDefinitions{
       GenericAPIServer: genericServer,
   }
   ```

4. 构建 API Group Info

   ```go
   apiResourceConfig := c.GenericConfig.MergedResourceConfig
   apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apiextensions.GroupName, Scheme, metav1.ParameterCodec, Codecs)
   ```

   * `MergedResourceConfig`: 合并后的资源配置，决定哪些资源被启用；
   * `NewDefaultAPIGroupInfo`: 为 `apiextensions.k8s.io` 这个 API Group 创建默认配置，该默认配置中缺失 rest.Storage 信息。

5. 注册 CRD 资源的 REST 存储

   ```go
   if resource := "customresourcedefinitions"; apiResourceConfig.ResourceEnabled(...) {
       customResourceDefinitionStorage, err := customresourcedefinition.NewREST(...)
       storage[resource] = customResourceDefinitionStorage
       storage[resource+"/status"] = customresourcedefinition.NewStatusREST(...)
   }
   ```

   NewREST 返回的 Storage 实现了 CRUD 操作，底层连接 etcd。为 CRD 构造两个 `rest.Storage`，其一为 CRD 自身准备，其二为 CRD 的 Status 子资源。

   CRD 的 rest.Storage 实例实际上是结构体 REST，代码如下：

   ```go
   // 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\registry\customresourcedefinition\etcd.go#L36-L65
   // rest implements a RESTStorage for API services against etcd
   type REST struct {
   	*genericregistry.Store
   }
   // NewREST returns a RESTStorage object that will work against API services.
   func NewREST(scheme *runtime.Scheme, optsGetter generic.RESTOptionsGetter) (*REST, error) {
   	strategy := NewStrategy(scheme)
   
   	store := &genericregistry.Store{
   		NewFunc:                   func() runtime.Object { return &apiextensions.CustomResourceDefinition{} },
   		NewListFunc:               func() runtime.Object { return &apiextensions.CustomResourceDefinitionList{} },
   		PredicateFunc:             MatchCustomResourceDefinition,
   		DefaultQualifiedResource:  apiextensions.Resource("customresourcedefinitions"),
   		SingularQualifiedResource: apiextensions.Resource("customresourcedefinition"),
   
   		CreateStrategy:      strategy,
   		UpdateStrategy:      strategy,
   		DeleteStrategy:      strategy,
   		ResetFieldsStrategy: strategy,
   
   		// TODO: define table converter that exposes more than name/creation timestamp
   		TableConvertor: rest.NewDefaultTableConvertor(apiextensions.Resource("customresourcedefinitions")),
   	}
   	options := &generic.StoreOptions{RESTOptions: optsGetter, AttrFunc: GetAttrs}
   	if err := store.CompleteWithOptions(options); err != nil {
   		return nil, err
   	}
   	return &REST{store}, nil
   }
   ```

   同样在这个文件中，函数 `NewREST()` 负责生成一个实例。有了 Storage 信息， apiGroupInfo 需要的信息便完善了。

6. 安装 API Group

   ```go
   apiGroupInfo.VersionedResourcesStorageMap[v1.SchemeGroupVersion.Version] = storage
   s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo)
   ```

   将 storage 映射到 v1 版本，然后调用 InstallAPIGroup 完成：

   * 注册 HTTP 路由（GET/POST/PUT/DELETE/PATCH/LIST/WATCH）；
   * 注册到 discovery 端点（让 kubectl api-resources 能发现）；
   * 设置 OpenAPI schema。

### 3. 响应对客制化资源的请求

CRD 实例定义出客制化 API，从使用者角度看，客制化 API 和 Kubernetes API 并无不同，用户同样可以通过 HTTP 请求去创建其实例。那么客制化 API 的端点如何制作出的？内置 API 均可借助 Generic Server 提供的接口完成注入和端点生成，然而客制化 API 相比内置 API 有所不同。在**内置 API** 注入 Generic Server 前，它们的**相关信息需要都已注册进了注册表**，例如 GVK、Go 结构体等，这些信息确保了端点正确生成。但**客制化 API 不具备这样的条件**，它的结构、属性所有信息都是在 CRD 资源中动态指定的，甚至没有专门的 Go 结构体去对应一个客制化 API。另外，内置 API 并不会动态地在控制面上创建和移除，例如 apps/v1 Deployment 不可能被一条命令从系统里拿掉，如果要移除一种 API 种类需要重启 API Server。但客制化 API 天然地允许动态创建和删除，Generic Server 只有 API 的注入接口，并没有移除。

既然 Generic Server 没办法为客制化 API 准备端点，那扩展 Server 只能自力更生。在 `New()` 方法中可以找到这部分逻辑。

```go
// 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\apiserver\apiserver.go#L164-L208
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*CustomResourceDefinitions, error) {
	crdClient, err := clientset.NewForConfig(s.GenericAPIServer.LoopbackClientConfig)
	if err != nil {
		// it's really bad that this is leaking here, but until we can fix the test (which I'm pretty sure isn't even testing what it wants to test),
		// we need to be able to move forward
		return nil, fmt.Errorf("failed to create clientset: %v", err)
	}
	s.Informers = externalinformers.NewSharedInformerFactory(crdClient, 5*time.Minute)

	delegateHandler := delegationTarget.UnprotectedHandler()
	if delegateHandler == nil {
		delegateHandler = http.NotFoundHandler()
	}

	versionDiscoveryHandler := &versionDiscoveryHandler{
		discovery: map[schema.GroupVersion]*discovery.APIVersionHandler{},
		delegate:  delegateHandler,
	}
	groupDiscoveryHandler := &groupDiscoveryHandler{
		discovery: map[string]*discovery.APIGroupHandler{},
		delegate:  delegateHandler,
	}
	establishingController := establish.NewEstablishingController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
	crdHandler, err := NewCustomResourceDefinitionHandler(
		versionDiscoveryHandler,
		groupDiscoveryHandler,
		s.Informers.Apiextensions().V1().CustomResourceDefinitions(),
		delegateHandler,
		c.ExtraConfig.CRDRESTOptionsGetter,
		c.GenericConfig.AdmissionControl,
		establishingController,
		c.ExtraConfig.ServiceResolver,
		c.ExtraConfig.AuthResolverWrapper,
		c.ExtraConfig.MasterCount,
		s.GenericAPIServer.Authorizer,
		c.GenericConfig.RequestTimeout,
		time.Duration(c.GenericConfig.MinRequestTimeout)*time.Second,
		apiGroupInfo.StaticOpenAPISpec,
		c.GenericConfig.MaxRequestBodyBytes,
	)
	if err != nil {
		return nil, err
	}
	s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", crdHandler)
	s.GenericAPIServer.Handler.NonGoRestfulMux.HandlePrefix("/apis/", crdHandler)
	s.GenericAPIServer.RegisterDestroyFunc(crdHandler.destroy)
```

1. 创建 CRD Client 和 Informer

   ```go
   crdClient, err := clientset.NewForConfig(s.GenericAPIServer.LoopbackClientConfig)
   s.Informers = externalinformers.NewSharedInformerFactory(crdClient, 5*time.Minute)
   ```

   * crdClient: 用于访问 CRD 资源的客户端（通过 loopback 连接自己）

   * Informers: SharedInformerFactory，用于监听 CRD 的变化，5 分钟 resync 一次

2. 获取委托 Handler

   ```go
   delegateHandler := delegationTarget.UnprotectedHandler()
   ```

   获取委托链中下一个 server 的 handler。如果当前 handler 无法处理请求，会转发给 delegateHandler（通常是 kube-aggregator 或 kube-apiserver）。

3. 创建 Discovery Handlers 

   ```go
   versionDiscoveryHandler := &versionDiscoveryHandler{
       discovery: map[schema.GroupVersion]*discovery.APIVersionHandler{},
       delegate:  delegateHandler,
   }
   groupDiscoveryHandler := &groupDiscoveryHandler{
       discovery: map[string]*discovery.APIGroupHandler{},
       delegate:  delegateHandler,
   }
   ```

   这两个 handler 负责 API 发现：当用户创建新 CRD 时，这些 handler 会动态更新，让 kubectl api-resources 能发现新资源。

   * `versionDiscoveryHandler`: 处理 `/apis/<group>/<version>` 的发现请求，返回该版本下有哪些资源

   * `groupDiscoveryHandler`: 处理 `/apis/<group>` 的发现请求，返回该 group 支持哪些版本

4. 创建 EstablishingController

   ```go
   establishingController := establish.NewEstablishingController(...)
   ```

   负责将 CRD 的状态从 Installing 变为 Established。只有 Established 状态的 CRD 才能正常处理 CR 请求。

5. 创建核心 CRD Handler

   ```go
   crdHandler, err := NewCustomResourceDefinitionHandler(
       versionDiscoveryHandler,  // 要点
       groupDiscoveryHandler,
       s.Informers.Apiextensions().V1().CustomResourceDefinitions(),
       delegateHandler,
       c.ExtraConfig.CRDRESTOptionsGetter,    // etcd 存储配置
       c.GenericConfig.AdmissionControl,       // 准入控制
       establishingController,
       c.ExtraConfig.ServiceResolver,          // webhook 服务解析
       c.ExtraConfig.AuthResolverWrapper,      // 认证解析
       c.ExtraConfig.MasterCount,
       s.GenericAPIServer.Authorizer,          // 授权
       c.GenericConfig.RequestTimeout,
       ...
   )
   ```

   这是最核心的组件，crdHandler 负责：

   * 根据请求路径动态匹配对应的 CRD
   * 为每个 CRD 动态创建 REST storage（连接 etcd）
   * 执行准入控制（Admission）、授权（Authorization）
   * 处理 CR 的 CRUD 操作

   在第 3 步定义版本发现器和组发现器，在这里**要点**处它们被作为入参去构造 `crdHandler` 变量。定义之初二者内部的字段 discovery 都是空 map，但 discovery 字段是发现器的 `ServeHTTP()` 方法执行时的信息来源，内容不能空，它们的填充是在一个称为发现控制器（`discoveryController`）的控制器中进行的【后续会详细讲解】，现在请假设两个发现器的 discovery 字段均已完全填充。

   * 组发现器的 discovery 字段中，键存放组名，而值是一个指针，指向 discovery 包【`staging\src\k8s.io\apiextensions-apiserver\pkg\apiserver\customresource_discovery.go`】内 APIGroupHandler 结构体实例。APIGroupHandler 结构体同样实现 http.Handler 接口，可以响应 HTTP 请求。当 discovery 被完全填充后，当前 Server 所支持的客制化 API 组分别与各自对应的 APIGroupHandler 实例配对出现在其中。组发现器的 `ServeHTTP()` 方法会从这个 map 中找到目标 API 组的 APIGroupHandler 实例，调用它的 `ServeHTTP()` 方法来响应请求，返回该组下所有版本。
   * 版本发现器的工作方式完全类似，只不过它的 discovery map 的健是组与版本；而值是 discovery 包的 APIVersionHandler 结构体实例。

6. 注册路由

   ```go
   s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", crdHandler)
   s.GenericAPIServer.Handler.NonGoRestfulMux.HandlePrefix("/apis/", crdHandler)
   s.GenericAPIServer.RegisterDestroyFunc(crdHandler.destroy)
   ```

   * 将 crdHandler 注册到 /apis 和 /apis/ 路径
   * 使用 NonGoRestfulMux 而不是 go-restful，因为 CR 的路由是动态的，无法预先定义
   * 注册销毁函数，server 关闭时清理资源

可以观察到，最重要的代码就是这两行：

```go
s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", crdHandler)
s.GenericAPIServer.Handler.NonGoRestfulMux.HandlePrefix("/apis/", crdHandler)
```

它把路由到 NonGoRestfulMux 的、目标端点以“/apis”或以“/apis/”为前缀的请求，全部交给变量 crdHandler 去处理。

> NOTE:
>
> NonGoRestfulMux 和 GoRestfulContainer 字段已经被反复提及，后者负责接收并转发针对 Kubernetes API 的 HTTP 请求；而前者兜底，处理所有后者不能处理的请求。两者有一个技术上的显著不同：GoRestfulContainer 在 go-restful 框架下构建，而 NonGoRestfulMux 就像它名字说的，没有用 go-restful，能接收它所分发的请求的处理器**只需实现 http.Handler 接口**，这多少提供了些灵活性。

#### 客制化 API 的 HTTP 请求响应

crdHandler 所接收到的请求要么针对 `/apis`，要么针对 `/apis/` 开头的一个端点，这背后隐藏了如下几类请求目的：

| **请求类别** | **请求目的**              | **端点路径 (Path)**           |
| ------------ | ------------------------- | ----------------------------- |
| **第一类**   | 获取所有 API 组           | `/apis`                       |
| **第二类**   | 获取某一 API 组的版本列表 | `/apis/<group>`               |
| **第三类**   | 获取某一版本下的 API 资源 | `/apis/<group>/<version>`     |
| **第四类**   | 操作具体的 API 资源       | `/apis/<group>/<version>/...` |

变量 crdHandler 的类型是个结构体，名称也是 crdHandler，它的 `ServeHTTP()` 方法给出了请求的处理逻辑，代码如下：

```go
// 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\apiserver\customresource_handler.go#L228-L366
func (r *crdHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	ctx := req.Context()
	requestInfo, ok := apirequest.RequestInfoFrom(ctx) // 要点①
	...
	if !requestInfo.IsResourceRequest { // 要点②
		pathParts := splitPath(requestInfo.Path) 
		// only match /apis/<group>/<version>
		// only registered under /apis
		if len(pathParts) == 3 { // 要点③
			r.versionDiscoveryHandler.ServeHTTP(w, req)
			return
		}
		// only match /apis/<group>
		if len(pathParts) == 2 { // 要点④
			r.groupDiscoveryHandler.ServeHTTP(w, req)
			return
		}

		r.delegate.ServeHTTP(w, req) // 要点⑤
		return
	}
	// 要点⑥
	crdName := requestInfo.Resource + "." + requestInfo.APIGroup
	...
}
```

1. 首先通过要点①处获取 `requestInfo` 。

2. 对于上表的四种请求，前三种都是非资源请求，而第四章是资源请求

   * 对于 `pathParts == 3` 则转发给版本发现器去处理。

   * 对于 `pathParts == 2` 则转发给组发现器去处理。

   * 还剩 `pathParts == 1` 则交给 `r.delegate.ServeHTTP`。

     > ```go
     > // 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\apiserver\customresource_handler.go#L190
     > // crd 的 New 函数如下:
     > func NewCustomResourceDefinitionHandler(
     > 	versionDiscoveryHandler *versionDiscoveryHandler,
     > 	groupDiscoveryHandler *groupDiscoveryHandler,
     > 	crdInformer informers.CustomResourceDefinitionInformer, // 👈
     > 	delegate http.Handler,
     > 	...
     > 	maxRequestBodyBytes int64) (*crdHandler, error) {
     > 	ret := &crdHandler{
     > 		versionDiscoveryHandler: versionDiscoveryHandler,
     > 		groupDiscoveryHandler:   groupDiscoveryHandler,
     > 		customStorage:           atomic.Value{},
     > 		crdLister:               crdInformer.Lister(), // 👈  
     > 		delegate:                delegate, 
     >     	...
     >     }
     >     ...
     > }
     > 
     > // 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\apiserver\apiserver.go#L190
     > crdHandler, err := NewCustomResourceDefinitionHandler(
     >     versionDiscoveryHandler,
     >     groupDiscoveryHandler,
     >     s.Informers.Apiextensions().V1().CustomResourceDefinitions(), 
     >     delegateHandler, // 👈 关键  
     > ```

     所以这里的 `r.delegate.ServeHTTP` 即传入的 `delegateHandler`。

   * 而第四种请求是资源请求，由 `ServeHTTP()` 方法自身处理这种情况。

#### ServeHTTP 方法

```go
// 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\apiserver\customresource_handler.go#L228-L366
func (r *crdHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ...
	crdName := requestInfo.Resource + "." + requestInfo.APIGroup
	crd, err := r.crdLister.Get(crdName)
	if apierrors.IsNotFound(err) {
		r.delegate.ServeHTTP(w, req)
		return
	}
	if err != nil {
		utilruntime.HandleError(err)
		responsewriters.ErrorNegotiated(
			apierrors.NewInternalError(fmt.Errorf("error resolving resource")),
			Codecs, schema.GroupVersion{Group: requestInfo.APIGroup, Version: requestInfo.APIVersion}, w, req,
		)
		return
	}

	// if the scope in the CRD and the scope in request differ (with exception of the verbs in possiblyAcrossAllNamespacesVerbs
	// for namespaced resources), pass request to the delegate, which is supposed to lead to a 404.
	namespacedCRD, namespacedReq := crd.Spec.Scope == apiextensionsv1.NamespaceScoped, len(requestInfo.Namespace) > 0
	if !namespacedCRD && namespacedReq {
		r.delegate.ServeHTTP(w, req)
		return
	}
	if namespacedCRD && !namespacedReq && !possiblyAcrossAllNamespacesVerbs.Has(requestInfo.Verb) {
		r.delegate.ServeHTTP(w, req)
		return
	}

	if !apiextensionshelpers.HasServedCRDVersion(crd, requestInfo.APIVersion) {
		r.delegate.ServeHTTP(w, req)
		return
	}

	// There is a small chance that a CRD is being served because NamesAccepted condition is true,
	// but it becomes "unserved" because another names update leads to a conflict
	// and EstablishingController wasn't fast enough to put the CRD into the Established condition.
	// We accept this as the problem is small and self-healing.
	if !apiextensionshelpers.IsCRDConditionTrue(crd, apiextensionsv1.NamesAccepted) &&
		!apiextensionshelpers.IsCRDConditionTrue(crd, apiextensionsv1.Established) {
		r.delegate.ServeHTTP(w, req)
		return
	}

	terminating := apiextensionshelpers.IsCRDConditionTrue(crd, apiextensionsv1.Terminating)

	crdInfo, err := r.getOrCreateServingInfoFor(crd.UID, crd.Name)
	if apierrors.IsNotFound(err) {
		r.delegate.ServeHTTP(w, req)
		return
	}
	if err != nil {
		utilruntime.HandleError(err)
		responsewriters.ErrorNegotiated(
			apierrors.NewInternalError(fmt.Errorf("error resolving resource")),
			Codecs, schema.GroupVersion{Group: requestInfo.APIGroup, Version: requestInfo.APIVersion}, w, req,
		)
		return
	}
	if !hasServedCRDVersion(crdInfo.spec, requestInfo.APIVersion) {
		r.delegate.ServeHTTP(w, req)
		return
	}

	deprecated := crdInfo.deprecated[requestInfo.APIVersion]
	for _, w := range crdInfo.warnings[requestInfo.APIVersion] {
		warning.AddWarning(req.Context(), "", w)
	}

	verb := strings.ToUpper(requestInfo.Verb)
	resource := requestInfo.Resource
	subresource := requestInfo.Subresource
	scope := metrics.CleanScope(requestInfo)
	supportedTypes := []string{
		string(types.JSONPatchType),
		string(types.MergePatchType),
		string(types.ApplyYAMLPatchType),
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.CBORServingAndStorage) {
		supportedTypes = append(supportedTypes, string(types.ApplyCBORPatchType))
	}

	var handlerFunc http.HandlerFunc
	subresources, err := apiextensionshelpers.GetSubresourcesForVersion(crd, requestInfo.APIVersion)
	if err != nil {
		utilruntime.HandleError(err)
		responsewriters.ErrorNegotiated(
			apierrors.NewInternalError(fmt.Errorf("could not properly serve the subresource")),
			Codecs, schema.GroupVersion{Group: requestInfo.APIGroup, Version: requestInfo.APIVersion}, w, req,
		)
		return
	}
	switch {
	case subresource == "status" && subresources != nil && subresources.Status != nil:
		handlerFunc = r.serveStatus(w, req, requestInfo, crdInfo, terminating, supportedTypes)
	case subresource == "scale" && subresources != nil && subresources.Scale != nil:
		handlerFunc = r.serveScale(w, req, requestInfo, crdInfo, terminating, supportedTypes)
	case len(subresource) == 0:
		handlerFunc = r.serveResource(w, req, requestInfo, crdInfo, crd, terminating, supportedTypes)
	default:
		responsewriters.ErrorNegotiated(
			apierrors.NewNotFound(schema.GroupResource{Group: requestInfo.APIGroup, Resource: requestInfo.Resource}, requestInfo.Name),
			Codecs, schema.GroupVersion{Group: requestInfo.APIGroup, Version: requestInfo.APIVersion}, w, req,
		)
	}

	if handlerFunc != nil {
		handlerFunc = metrics.InstrumentHandlerFunc(verb, requestInfo.APIGroup, requestInfo.APIVersion, resource, subresource, scope, metrics.APIServerComponent, deprecated, "", handlerFunc)
		handler := genericfilters.WithWaitGroup(handlerFunc, longRunningFilter, crdInfo.waitGroup)
		handler.ServeHTTP(w, req)
		return
	}
}
```

1. **校验请求**

   ```go
   // 1.1 构造 CRD 名称并查找
   crdName := requestInfo.Resource + "." + requestInfo.APIGroup  // 如 "databases.example.com"
   crd, err := r.crdLister.Get(crdName)
   if apierrors.IsNotFound(err) {
       r.delegate.ServeHTTP(w, req)  // CRD 不存在，交给下一个 Server 处理
       return
   }
   
   // 1.2 校验 Namespace 作用域是否匹配
   namespacedCRD, namespacedReq := crd.Spec.Scope == apiextensionsv1.NamespaceScoped, len(requestInfo.Namespace) > 0
   if !namespacedCRD && namespacedReq {  // CRD 是集群级别，但请求带了 namespace
       r.delegate.ServeHTTP(w, req)
       return
   }
   
   // 1.3 校验请求的 API 版本是否被 CRD 支持
   if !apiextensionshelpers.HasServedCRDVersion(crd, requestInfo.APIVersion) {
       r.delegate.ServeHTTP(w, req)
       return
   }
   
   // 1.4 校验 CRD 是否已建立（NamesAccepted 或 Established）
   if !apiextensionshelpers.IsCRDConditionTrue(crd, apiextensionsv1.NamesAccepted) &&
       !apiextensionshelpers.IsCRDConditionTrue(crd, apiextensionsv1.Established) {
       r.delegate.ServeHTTP(w, req)
       return
   }
   ```

2. 获取 CRD 实例

   ```go
   terminating := apiextensionshelpers.IsCRDConditionTrue(crd, apiextensionsv1.Terminating)
   ```

   检查 CRD 是否正在被删除（Terminating 状态）。

3. 获取 crdInfo

   ```go
   crdInfo, err := r.getOrCreateServingInfoFor(crd.UID, crd.Name)
   ```

   crdInfo 包含了处理该 CRD 请求所需的所有信息：

   * `storages`: 每个版本的 etcd 存储
   * `requestScopes`: 每个版本的请求作用域
   * `statusRequestScopes`: `status` 子资源的作用域
   * `scaleRequestScopes`: `scale` 子资源的作用域

4. 获取子资源信息并分发

   ```go
   subresources, err := apiextensionshelpers.GetSubresourcesForVersion(crd, requestInfo.APIVersion)
   
   var handlerFunc http.HandlerFunc
   switch {
   case subresource == "status" && subresources != nil && subresources.Status != nil:
       // 路径: /apis/<group>/<version>/<resource>/<name>/status
       handlerFunc = r.serveStatus(w, req, requestInfo, crdInfo, terminating, supportedTypes)
       
   case subresource == "scale" && subresources != nil && subresources.Scale != nil:
       // 路径: /apis/<group>/<version>/<resource>/<name>/scale
       handlerFunc = r.serveScale(w, req, requestInfo, crdInfo, terminating, supportedTypes)
       
   case len(subresource) == 0:
       // 路径: /apis/<group>/<version>/<resource> 或 /apis/<group>/<version>/<resource>/<name>
       handlerFunc = r.serveResource(w, req, requestInfo, crdInfo, crd, terminating, supportedTypes)
       
   default:
       // 不支持的子资源，返回 404
       responsewriters.ErrorNegotiated(apierrors.NewNotFound(...), ...)
   }
   ```

5. 调用响应器

   ```go
   if handlerFunc != nil {
       // 包装 metrics 监控
       handlerFunc = metrics.InstrumentHandlerFunc(verb, ...)
       // 包装 WaitGroup（用于优雅关闭）
       handler := genericfilters.WithWaitGroup(handlerFunc, longRunningFilter, crdInfo.waitGroup)
       // 执行实际的请求处理
       handler.ServeHTTP(w, req)
       return
   }
   ```

处理第四类请求的流程图如下：

```markdown
┌─────────────────────────────────────────────────────────────────────────────┐
│  收到请求，获取信息                                                           │
│  requestInfo := apirequest.RequestInfoFrom(ctx)                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  校验请求                                                                    │
│  - CRD 是否存在？                                                            │
│  - Namespace 作用域是否匹配？                                                 │
│  - API 版本是否支持？                                                         │
│  - CRD 是否已 Established？                                                  │
│  （任一校验失败 → delegate 处理）                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  获取 CRD 实例                                                               │
│  crd, err := r.crdLister.Get(crdName)                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  获取 crdInfo（调方法 GetSubresourcesForVersion）                             │
│  crdInfo := r.getOrCreateServingInfoFor(crd.UID, crd.Name)                  │
│  subresources := GetSubresourcesForVersion(crd, requestInfo.APIVersion)     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
                    ┌───────────────┼───────────────┐
                    ↓               ↓               ↓
              ../status        无子资源         ../scale
                    ↓               ↓               ↓
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ serveStatus │ │serveResource│ │ serveScale  │
│ status子资源 │ │ 客制化资源   │ │ scale子资源  │
│   响应器     │ │   响应器     │ │   响应器     │
└─────────────┘ └─────────────┘ └─────────────┘
                    └───────────────┼───────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  调用响应器                                                                  │
│  handler.ServeHTTP(w, req)                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

####  响应“组发现”和“版本发现”请求

在 `crdHandler.ServeHTTP()` 的逻辑中，当目标端点是“`/apis/<组>`”或“`/apis/<组>/<版本>`” 时，请求会被交由组发现器和版本发现器去处理。

1. 组发现：客户端想获取当前 Server 所支持的某一个 API 组下所有版本，可以向端点 `/apis/<组>` 发 Get 请求，一个组发现器会负责响应这个请求。组发现器类型为结构体 `versionDiscoveryHandler` （定义于 `New()` 方法所在的 apiserver 包），它具有 `ServeHTTP()` 方法，实现了 `http.Handler` 接口。
2. 版本发现：客户端想获取当前 Server 所支持的某一个 API 组的某一版本内所有 API 资源以及各个资源所支持的操作（`Get，Post，Watch...`），可以向端点 `/apis/<组>/<版本>` 发送 Get 请求，一个版本发现器会负责响应这个请求。 版本发现器类型为 `groupDiscoveryHandler` 结构体 （同样定义于 `New()` 方法所在的 apiserver 包），它同样具有 `ServeHTTP()` 方法，实现 `http.Handler` 接口。

如上所述，组发现器和版本发现器均在 `New()` 方法中创建，并交由 crdHandler 供其使用，`New()` 中相关代码如下所示。

### 4. 监听 CRD 实例创建

```go
// 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\apiserver\apiserver.go#L210-L274
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*CustomResourceDefinitions, error) {
    ...
	aggregatedDiscoveryManager := genericServer.AggregatedDiscoveryGroupManager
	if aggregatedDiscoveryManager != nil {
		aggregatedDiscoveryManager = aggregatedDiscoveryManager.WithSource(aggregated.CRDSource)
	}
	discoveryController := NewDiscoveryController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), versionDiscoveryHandler, groupDiscoveryHandler, aggregatedDiscoveryManager)
	namingController := status.NewNamingConditionController(klog.TODO() /* for contextual logging */, s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
	nonStructuralSchemaController := nonstructuralschema.NewConditionController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
	apiApprovalController := apiapproval.NewKubernetesAPIApprovalPolicyConformantConditionController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
	finalizingController := finalizer.NewCRDFinalizer(
		s.Informers.Apiextensions().V1().CustomResourceDefinitions(),
		crdClient.ApiextensionsV1(),
		crdHandler,
	)

	s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-informers", func(hookContext genericapiserver.PostStartHookContext) error {
		s.Informers.Start(hookContext.Done())
		return nil
	})
	s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-controllers", func(hookContext genericapiserver.PostStartHookContext) error {
		// OpenAPIVersionedService and StaticOpenAPISpec are populated in generic apiserver PrepareRun().
		// Together they serve the /openapi/v2 endpoint on a generic apiserver. A generic apiserver may
		// choose to not enable OpenAPI by having null openAPIConfig, and thus OpenAPIVersionedService
		// and StaticOpenAPISpec are both null. In that case we don't run the CRD OpenAPI controller.
		if s.GenericAPIServer.StaticOpenAPISpec != nil {
			if s.GenericAPIServer.OpenAPIVersionedService != nil {
				openapiController := openapicontroller.NewController(s.Informers.Apiextensions().V1().CustomResourceDefinitions())
				go openapiController.Run(s.GenericAPIServer.StaticOpenAPISpec, s.GenericAPIServer.OpenAPIVersionedService, hookContext.Done())
			}

			if s.GenericAPIServer.OpenAPIV3VersionedService != nil {
				openapiv3Controller := openapiv3controller.NewController(s.Informers.Apiextensions().V1().CustomResourceDefinitions())
				go openapiv3Controller.Run(s.GenericAPIServer.OpenAPIV3VersionedService, hookContext.Done())
			}
		}

		go namingController.Run(hookContext.Done())
		go establishingController.Run(hookContext.Done())
		go nonStructuralSchemaController.Run(5, hookContext.Done())
		go apiApprovalController.Run(5, hookContext.Done())
		go finalizingController.Run(5, hookContext.Done())

		discoverySyncedCh := make(chan struct{})
		go discoveryController.Run(hookContext.Done(), discoverySyncedCh)
		select {
		case <-hookContext.Done():
		case <-discoverySyncedCh:
		}

		return nil
	})
	// we don't want to report healthy until we can handle all CRDs that have already been registered.  Waiting for the informer
	// to sync makes sure that the lister will be valid before we begin.  There may still be races for CRDs added after startup,
	// but we won't go healthy until we can handle the ones already present.
	s.GenericAPIServer.AddPostStartHookOrDie("crd-informer-synced", func(ctx genericapiserver.PostStartHookContext) error {
		return wait.PollUntilContextCancel(ctx.Context, 100*time.Millisecond, true, func(ctx context.Context) (bool, error) {
			if s.Informers.Apiextensions().V1().CustomResourceDefinitions().Informer().HasSynced() {
				close(hasCRDInformerSyncedSignal)
				return true, nil
			}
			return false, nil
		})
	})

	return s, nil
}
```

在扩展 Server 的构建方法 `New()` 的后半部分，一系列控制器被构建出来，它们是：

1. 发现控制器：用于填充客制化 API 组和版本发现器的 discovery 属性，也为服务于聚合器的 resourceManager 填充信息。

   ```go
   aggregatedDiscoveryManager := genericServer.AggregatedDiscoveryGroupManager
   if aggregatedDiscoveryManager != nil {
       aggregatedDiscoveryManager = aggregatedDiscoveryManager.WithSource(aggregated.CRDSource)
   }
   discoveryController := NewDiscoveryController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), versionDiscoveryHandler, groupDiscoveryHandler, aggregatedDiscoveryManager)
   ```

   由于 API 发现信息需要统一聚合，具体如下：

   ```markdown
                       ┌─────────────────────────────────────┐
                       │   AggregatedDiscoveryGroupManager   │
                       │         (共享的发现管理器)            │
                       └─────────────────────────────────────┘
                              ↑           ↑           ↑
                              │           │           │
                 ┌────────────┘           │           └────────────┐
                 │                        │                        │
       ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
       │  kube-aggregator │    │ kube-apiserver  │    │ apiextensions   │
       │ (AggregatorSource)│    │ (BuiltinSource) │    │  (CRDSource)    │
       └─────────────────┘    └─────────────────┘    └─────────────────┘
   ```

   所以在创建扩展 Server 时，需要得到聚合 Server 的发现管理器。

2. 名称控制器：用于校验客制化 API 的命名（单数名，复数名，短名和 kind）会不会已经在同 API 组下存在了。

   ```go
   namingController := status.NewNamingConditionController(klog.TODO() /* for contextual logging */, s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
   ```

3. 非结构化规格控制器：根据 CRD 实例中定义的客制化 API 规格——`spec.Scheme` 节点所含内容，校验一个客制化资源的定义是否符合。

   ```go
   nonStructuralSchemaController := nonstructuralschema.NewConditionController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
   ```

4. API 审批控制器：如果要在命名空间 `k8s.io`，、`*.k8s.io`、`kubernetes.io` 或 `*.kubernetes.io` 内创建 CRD，需要具有名为 `api-approved.kubernetes.io` 的注解，注解的内容是一个 URL，指向该CRD 的设计描述页面；而如果该客制化API 还没有被批准，值必须是一个以 ‘unapproved’ 开头的字符串。该控制器会把这个信息反应到 CRD 实例的 status 属性上去。

   ```go
   apiApprovalController := apiapproval.NewKubernetesAPIApprovalPolicyConformantConditionController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
   ```

5. CRD 清理控制器：当一个 CRD 实例被删除时，这个控制器会删除所有它的客制 化 API 实例，从而达到彻底清理的目的。

   ```go
   finalizingController := finalizer.NewCRDFinalizer(
       s.Informers.Apiextensions().V1().CustomResourceDefinitions(),
       crdClient.ApiextensionsV1(),
       crdHandler,
   )
   ```

以上就是扩展 Server 涉及的重要控制器，后续会专门介绍它们的实现。`New()` 方法利用各个控制器的工厂方法分别创建出它们的一个实例，并在扩展 Server 启动后运行的名为 `start-apiextensions-controllers` 的钩子中去启动它们。

在一系列控制器被构建出来后，会太添加如下三个 PostStartHook：

1. `start-apiextensions-informers`：启动 `SharedInformerFactory`

   ```go
   s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-informers", func(hookContext genericapiserver.PostStartHookContext) error {
       s.Informers.Start(hookContext.Done())
       return nil
   })
   ```

   * `s.Informers.Start()` 会启动所有已注册的 `Informer`；
   * 开始 List & Watch CRD 资源；
   * `hookContext.Done()` 是一个 channel，Server 关闭时会被 close，用于通知 Informer 停止。

2. `start-apiextensions-controllers`：

   ```go
   s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-controllers", func(hookContext genericapiserver.PostStartHookContext) error {
       // 1. 启动 OpenAPI Controller（如果启用）
       if s.GenericAPIServer.StaticOpenAPISpec != nil {
           if s.GenericAPIServer.OpenAPIVersionedService != nil {
               openapiController := openapicontroller.NewController(...)
               go openapiController.Run(...)  // 动态更新 /openapi/v2
           }
           if s.GenericAPIServer.OpenAPIV3VersionedService != nil {
               openapiv3Controller := openapiv3controller.NewController(...)
               go openapiv3Controller.Run(...)  // 动态更新 /openapi/v3
           }
       }
   
       // 2. 启动各种 Controller
       go namingController.Run(hookContext.Done())           // 检查命名冲突
       go establishingController.Run(hookContext.Done())     // 建立 CRD
       go nonStructuralSchemaController.Run(5, hookContext.Done())  // 检查 schema
       go apiApprovalController.Run(5, hookContext.Done())   // 检查 API 审批
       go finalizingController.Run(5, hookContext.Done())    // 处理 CRD 删除
   
       // 3. 启动 DiscoveryController 并等待同步
       discoverySyncedCh := make(chan struct{})
       go discoveryController.Run(hookContext.Done(), discoverySyncedCh)
       select {
       case <-hookContext.Done():  // Server 关闭
       case <-discoverySyncedCh:   // Discovery 同步完成
       }
   
       return nil
   })
   ```

   * OpenAPI Controller 监听 CRD 变化，动态更新 OpenAPI spec，让 `kubectl explain <crd>` 能工作；
   * 参数 5 表示 worker 数量（并发处理能力）；
   * Hook 会阻塞等待 `discoverySyncedCh`，确保 Discovery 信息同步完成后才返回。

3. `crd-informer-synced`：等待 CRD Informer 完成首次同步

   ```go
   s.GenericAPIServer.AddPostStartHookOrDie("crd-informer-synced", func(ctx genericapiserver.PostStartHookContext) error {
       return wait.PollUntilContextCancel(ctx.Context, 100*time.Millisecond, true, func(ctx context.Context) (bool, error) {
           if s.Informers.Apiextensions().V1().CustomResourceDefinitions().Informer().HasSynced() {
               close(hasCRDInformerSyncedSignal)  // 关闭信号
               return true, nil  // 停止轮询
           }
           return false, nil  // 继续轮询
       })
   })
   ```

   * 每 100ms 检查一次 `HasSynced()`；
   * 同步完成后关闭 `hasCRDInformerSyncedSignal`；
   * 这个信号与前面的 `RegisterMuxAndDiscoveryCompleteSignal` 配合。

至此，扩展 Server 的构造过程就完成了，这个实例会在 `CreateServerChain()` 方法中被嵌 入 Server 链中，最终成为 API Server 的一部分。

## 启动扩展 Server

如前所述，扩展 Server 可以被编译为单独可执行应用程序去运行。扩展 Server 启动代码只有在其独立运行时才会执行。程序主函数秉承了 Cobra 设计风格，非常简单：创建一个命令，然后运行之。扩展 Server 的主函数代码如下所示：

```go
// 代码: staging\src\k8s.io\apiextensions-apiserver\main.go
package main

import (
	"os"

	"k8s.io/apiextensions-apiserver/pkg/cmd/server"
	genericapiserver "k8s.io/apiserver/pkg/server"
	"k8s.io/component-base/cli"
)

func main() {
	ctx := genericapiserver.SetupSignalContext()
	cmd := server.NewServerCommand(ctx, os.Stdout, os.Stderr)
	code := cli.Run(cmd)
	os.Exit(code)
}
```

上述代码中，`NewServerCommand()` 方法所制作的命令对象是关键。

```go
// 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\cmd\server\server.go#L29-L56
func NewServerCommand(ctx context.Context, out, errOut io.Writer) *cobra.Command {
	o := options.NewCustomResourceDefinitionsServerOptions(out, errOut)

	cmd := &cobra.Command{
		Short: "Launch an API extensions API server",
		Long:  "Launch an API extensions API server",
		PersistentPreRunE: func(*cobra.Command, []string) error {
			return o.ServerRunOptions.ComponentGlobalsRegistry.Set()
		},
		RunE: func(c *cobra.Command, args []string) error {
			if err := o.Complete(); err != nil {
				return err
			}
			if err := o.Validate(); err != nil {
				return err
			}
			if err := Run(c.Context(), o); err != nil {
				return err
			}
			return nil
		},
	}
	cmd.SetContext(ctx)

	fs := cmd.Flags()
	o.AddFlags(fs)
	return cmd
}
```

1. 首先为所生成的命令对象设置可用的命令行标志，以供用户提供参数。这些参数均来自底层的 Generic Server，包含三个方面：

   ```go
   // 代码: staging\src\k8s.io\apiextensions-apiserver\pkg\cmd\server\options\options.go#L67-L81
   // NewCustomResourceDefinitionsServerOptions creates default options of an apiextensions-apiserver.
   func NewCustomResourceDefinitionsServerOptions(out, errOut io.Writer) *CustomResourceDefinitionsServerOptions {
   	o := &CustomResourceDefinitionsServerOptions{
   		ServerRunOptions: genericoptions.NewServerRunOptions(),
   		RecommendedOptions: genericoptions.NewRecommendedOptions(
   			defaultEtcdPathPrefix,
   			apiserver.Codecs.LegacyCodec(v1beta1.SchemeGroupVersion, v1.SchemeGroupVersion),
   		),
   		APIEnablement: genericoptions.NewAPIEnablementOptions(),
   
   		StdOut: out,
   		StdErr: errOut,
   	}
   
   	return o
   }
   ```

   1. 通用标志，来自 `ServeRunOptions` 结构体的 `AddUniversalFlags()` 方法。
   2. 推荐标志，来自 `RecommendedOptions` 结构体的 `AddFlags()` 方法。
   3. 开启、关闭 API 的标志，来自 `APIEnablementOptions` 的 `AddFlags()` 方法。

2. 为该命令对象的 RunE 属性赋予一个匿名函数，它会在用户启动本程序时被调用， 从而将扩展 Server 运行起来。

   ```go
   RunE: func(c *cobra.Command, args []string) error {
       if err := o.Complete(); err != nil {
           return err
       }
       if err := o.Validate(); err != nil {
           return err
       }
       if err := Run(c.Context(), o); err != nil {
           return err
       }
       return nil
   },
   ```

   用户通过命令行输入的参数会被 Cobra 转交到选项结构体实例，即变量 o 中，通过变量 o 该匿名函数在执行时便得到了包含用户输入的所有参数值，在经过 `o.Complete()` 的补全和 `o.Validate()` 的校验后，以变量 o 为一个参数去执行 `Run()` 函数。`Run()` 函数的代码如下所示：

   ```go
   func Run(ctx context.Context, o *options.CustomResourceDefinitionsServerOptions) error {
   	config, err := o.Config()
   	if err != nil {
   		return err
   	}
   
   	server, err := config.Complete().New(genericapiserver.NewEmptyDelegate())
   	if err != nil {
   		return err
   	}
   	return server.GenericAPIServer.PrepareRun().RunWithContext(ctx)
   }
   ```

   上述代码中，`Run()` 方法分三步将 Server 启动起来：

   1. 由选项结构体制作 Server 运行配置结构体实例。
   2. 对运行配置结构体实例进行完善（`Complete() `方法）并由此创建扩展 Server 实例，过程就是已经讲解的 `New()` 方法。
   3. 最后启动扩展 Server 的底座 Generic Server，这会启动一个 Web Server 等待响应请求。

扩展 Server 的代码值得一看的一个重要原因是，它展示了如何基于 Generic Server 做一 个子 Server，代码非常清晰简明，开发者看得懂。
