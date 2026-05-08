+++
title = "K8s Generic Server Admission"
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

series_order = 11

+++

# K8s Generic Server Admission

准入控制器会在请求经过认证、授权等前置检查后，对象被真正持久化前进行最后的拦截（实际上准入控制的实现位置是在 RESTStorage 存储层），对请求的资源对象进行存储前的变更（Mutating）或验证（Validating），以确保其符合准入控制要求。准入控制器以插件的形式运行，支持通过命令行参数动态启用：

```bash
$ kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger …
```

或禁用：

```bash
$ kube-apiserver --disable-admission-plugins=PodNodeSelector, AlwaysDeny …
```

指定的准入控制器插件。

同时，Kubernetes 支持自定义扩展。当同时启用多个准入控制器插件时，将按照内置既定的顺序依次执行，与配置参数中的准入控制器插件顺序无关。**准入控制器限制创建、删除、修改对象的请求**，也可以阻止自定义操作（如通过 API 服务器代理连接 Pod 的请求），**但准入控制器不会也不能阻止读取对象的请求**（如 get、list、watch）。

## Admission 接口

1. 基础接口：控制器插件必须实现的接口。

   ```go
   // Interface is an abstract, pluggable interface for Admission Control decisions.
   type Interface interface {
   	// Handles returns true if this admission controller can handle the given operation
   	// where operation can be one of CREATE, UPDATE, DELETE, or CONNECT
       // 判断插件是否处理特定操作（CREATE, UPDATE, DELETE, CONNECT）
   	Handles(operation Operation) bool
   }
   ```

2. 变更接口：参与修改阶段必须实现的接口，它使得本插件成为修改准入控制器。

   ```go
   type MutationInterface interface {
   	Interface
   
   	// Admit makes an admission decision based on the request attributes.
   	// Context is used only for timeout/deadline/cancellation and tracing information.
       // 执行准入决策，可以修改资源对象
   	Admit(ctx context.Context, a Attributes, o ObjectInterfaces) (err error)
   }
   ```

3. 验证接口：参与校验阶段必须实现的接口，它使得本插件成为校验准入控制器。

   ```go
   // ValidationInterface is an abstract, pluggable interface for Admission Control decisions.
   type ValidationInterface interface {
   	Interface
   
   	// Validate makes an admission decision based on the request attributes.  It is NOT allowed to mutate
   	// Context is used only for timeout/deadline/cancellation and tracing information.
       // 执行准入决策，只能验证不能修改
   	Validate(ctx context.Context, a Attributes, o ObjectInterfaces) (err error)
   }
   ```

## 内置插件

Kubernetes 官方提供了多达 34 种准入控制器插件，将其存放在 `plugin\pkg\admission` 目录下。你可以进入该目录进行查看。

该目录下的插件不会全部开启，**默认启用的准入控制器：**

* **核心生命周期与完整性：** `NamespaceLifecycle`、`ServiceAccount`、`NodeRestriction`、`TaintNodesByCondition`
* **扩展性与 Webhook：** `MutatingAdmissionWebhook`、`ValidatingAdmissionWebhook`
* **资源管理与默认值：** `LimitRanger`、`ResourceQuota`、`Priority`、`DefaultTolerationSeconds`、`DefaultStorageClass`
* **存储与保护：** `StorageObjectInUseProtection`、`PersistentVolumeClaimResize`
* **安全规范：** `PodSecurity` (注意：`PodSecurityPolicy` 已被移除)

## 内部插件初始化流程

以 `namespace.lifecycle` 插件为例，讲解该插件的初始化流程，而函数的具体逻辑在后续进行详细讲解，这里主要先梳理架构。

### 1. 定义插件承载体

```go
// Lifecycle is an implementation of admission.Interface.
// It enforces life-cycle constraints around a Namespace depending on its Phase
type Lifecycle struct { 
    // 嵌入 Handler - 提供 Handles() 方法的默认实现，并内嵌 ReadyFunc，即InformerFactory产品1: 同步状态检查函数
	*admission.Handler  
	client             kubernetes.Interface // 备用数据源：直接 API 调用（兜底
	immortalNamespaces sets.String 			//不可删除的 Namespace 列表
	namespaceLister    corelisters.NamespaceLister // InformerFactory 产品2. 主要数据源：本地缓存（高性能）
	// forceLiveLookupCache holds a list of entries for namespaces that we have a strong reason to believe are stale in our local cache.
	// if a namespace is in this cache, then we will ignore our local state and always fetch latest from api server.
	forceLiveLookupCache *utilcache.LRUExpireCache
}
```

### 2. 实现插件业务逻辑

目的：

* 阻止在「正在删除」的 Namespace 中创建新资源
* 阻止删除系统关键 Namespace（default、kube-system、kube-public）
* 确保资源操作的目标 Namespace 存在

```go
func (l *Lifecycle) Admit(ctx context.Context, a admission.Attributes, o admission.ObjectInterfaces) error {
    // ========== 第一阶段：快速放行 ==========
    if a.GetKind().GroupKind() == v1.SchemeGroupVersion.WithKind("Namespace").GroupKind() {
        // 如果是删除 Namespace，加入强制实时查询缓存
        if a.GetOperation() == admission.Delete {
            l.forceLiveLookupCache.Add(a.GetName(), true, forceLiveLookupTTL)
        }
        // 对 Namespace 资源本身的所有操作直接放行
        return nil
    }
    // ========== 第二阶段 ==========
    // 1. 阻止删除不可删除的命名空间
    if a.GetOperation() == admission.Delete && 
       a.GetKind().GroupKind() == v1.SchemeGroupVersion.WithKind("Namespace").GroupKind() && 
       l.immortalNamespaces.Has(a.GetName()) {
        return errors.NewForbidden(a.GetResource().GroupResource(), a.GetName(), 
            fmt.Errorf("this namespace may not be deleted"))
    }

    // 2. 集群级资源（非 Namespace）直接放行
    if len(a.GetNamespace()) == 0 && 
       a.GetKind().GroupKind() != v1.SchemeGroupVersion.WithKind("Namespace").GroupKind() {
        return nil
    }

    // 3. 对 Namespace 资源本身的操作直接放行（上面讲的"漏洞"处理）
    if a.GetKind().GroupKind() == v1.SchemeGroupVersion.WithKind("Namespace").GroupKind() {
        if a.GetOperation() == admission.Delete {
            l.forceLiveLookupCache.Add(a.GetName(), true, forceLiveLookupTTL)
        }
        return nil
    }

    // 4. 删除其他资源直接放行（允许清理 Terminating 中的资源）
    if a.GetOperation() == admission.Delete {
        return nil
    }

    // 5. AccessReview 请求直接放行（避免信息泄露）
    if isAccessReview(a) {
        return nil
    }

    // ========== 第三阶段：检查 Namespace 状态 ==========
    
    // 6. 等待 Informer 缓存就绪
    if !l.WaitForReady() {
        return admission.NewForbidden(a, fmt.Errorf("not yet ready to handle request"))
    }

    // 7. 从缓存获取 Namespace
    namespace, err := l.namespaceLister.Get(a.GetNamespace())
    // ... 错误处理 ...

    // 8. 如果不存在且是 Create 操作，等待 50ms 后重试
    //    （处理"创建 namespace 后立即创建资源"的竞态条件）
    if !exists && a.GetOperation() == admission.Create {
        time.Sleep(missingNamespaceWait) // 50ms
        namespace, err = l.namespaceLister.Get(a.GetNamespace())
        // ...
    }

    // 9. 判断是否需要强制实时查询
    forceLiveLookup := false
    if _, ok := l.forceLiveLookupCache.Get(a.GetNamespace()); ok {
        // 缓存说 namespace 存在且 Active，但我们怀疑它正在被删除
        forceLiveLookup = exists && namespace.Status.Phase == v1.NamespaceActive
    }

    // 10. 如果缓存不可信，直接查询 API Server
    if !exists || forceLiveLookup {
        namespace, err = l.client.CoreV1().Namespaces().Get(ctx, a.GetNamespace(), metav1.GetOptions{})
        // ...
    }

    // ========== 第四阶段：最终判断 ==========
    
    // 11. 核心逻辑：禁止在 Terminating 状态的 Namespace 中创建资源
    if a.GetOperation() == admission.Create {
        if namespace.Status.Phase != v1.NamespaceTerminating {
            return nil  // Active 状态，放行
        }
        
        // Terminating 状态，拒绝创建
        err := admission.NewForbidden(a, 
            fmt.Errorf("unable to create new content in namespace %s because it is being terminated", 
                a.GetNamespace()))
        // 添加详细的错误原因
        if apierr, ok := err.(*errors.StatusError); ok {
            apierr.ErrStatus.Details.Causes = append(apierr.ErrStatus.Details.Causes, 
                metav1.StatusCause{
                    Type:    v1.NamespaceTerminatingCause,
                    Message: fmt.Sprintf("namespace %s is being terminated", a.GetNamespace()),
                    Field:   "metadata.namespace",
                })
        }
        return err
    }

    return nil
}
```

这里逻辑不是重点。

### 3. 设计注入依赖机制

```go
// Register registers a plugin
func Register(plugins *admission.Plugins) {
	plugins.Register(PluginName, func(config io.Reader) (admission.Interface, error) {
        // 此时只有 config 参数
    	// ❌ 没有 client
    	// ❌ 没有 InformerFactory，而 namespaceLister 由 InformerFactory 得到
        // 只能创建一个"半成品"
		return NewLifecycle(sets.NewString(metav1.NamespaceDefault, metav1.NamespaceSystem, metav1.NamespacePublic))
	})
}

// NewLifecycle creates a new namespace Lifecycle admission control handler
func NewLifecycle(immortalNamespaces sets.String) (*Lifecycle, error) {
	return newLifecycleWithClock(immortalNamespaces, clock.RealClock{})
}

func newLifecycleWithClock(immortalNamespaces sets.String, clock utilcache.Clock) (*Lifecycle, error) {
	forceLiveLookupCache := utilcache.NewLRUExpireCacheWithClock(100, clock)
	return &Lifecycle{
		Handler:              admission.NewHandler(admission.Create, admission.Update, admission.Delete),
		immortalNamespaces:   immortalNamespaces,
		forceLiveLookupCache: forceLiveLookupCache,
        /*
        // 以下全是 nil，等待注入
        client:          nil,  // ❌ 需要注入
        namespaceLister: nil,  // ❌ 需要注入
        informerSynced:  nil,  // ❌ 需要注入
		*/
	}, nil
}
```

这是**最关键的架构问题**：Lifecycle 需要 namespaceLister，但创建时还没有 InformerFactory，也没有 client 时序如下：

```markdown
时间线
──────────────────────────────────────────────────────────────────►

阶段1: 插件注册                阶段2: 基础设施创建              阶段3: 依赖注入
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│ plugins.Register│          │ 创建 Client     │          │ 遍历所有插件    │
│ 创建 Lifecycle  │          │ 创建 Informer   │          │ 注入依赖        │
│ (空壳)          │          │ Factory         │          │                 │
└─────────────────┘          └─────────────────┘          └─────────────────┘
        │                            │                            │
        │                            │                            │
        ▼                            ▼                            ▼
   Lifecycle 存在              Client 存在                  Lifecycle 完整
   但 client=nil              InformerFactory 存在          可以工作
   lister=nil

```

解决方式：Setter 注入模式，以 InformerFactory 为例：

```go
// 这个接口声明：实现者需要被注入 InformerFactory
type WantsExternalKubeInformerFactory interface {
    SetExternalKubeInformerFactory(informers.SharedInformerFactory)
    admission.InitializationValidator  // 验证注入是否成功
}

// InitializationValidator holds ValidateInitialization functions, which are responsible for validation of initialized
// shared resources and should be implemented on admission plugins
type InitializationValidator interface {
	ValidateInitialization() error
}
```

然后让 `Lifecycle` 实现该结构体 Lifecycle 实现这个接口：

```go
var _ = initializer.WantsExternalKubeInformerFactory(&Lifecycle{}) // 确保实现

// SetExternalKubeInformerFactory 实现依赖注入接口
func (l *Lifecycle) SetExternalKubeInformerFactory(f informers.SharedInformerFactory) {
    // 从 InformerFactory 中提取我们需要的组件
    namespaceInformer := f.Core().V1().Namespaces()
    
    // 获取 Lister（本地缓存访问器）
    l.namespaceLister = namespaceInformer.Lister()
    
    // 获取同步状态检查函数
    l.informerSynced = namespaceInformer.Informer().HasSynced
    
    // 设置就绪检查（Handler 提供的能力）
    l.SetReadyFunc(l.informerSynced)
}

// ValidateInitialization 验证依赖是否正确注入
func (l *Lifecycle) ValidateInitialization() error {
    if l.namespaceLister == nil {
        return fmt.Errorf("missing namespaceLister")
    }
    return nil
}
```

### 4. Initializer 的工作原理

API Server 中有一个 PluginInitializer，它负责给所有插件注入依赖：

```go
// ============================================================================
// PluginInitializer - 依赖注入器
// ============================================================================

type pluginInitializer struct {
    client          kubernetes.Interface
    informerFactory informers.SharedInformerFactory
    // ... 其他可注入的依赖
}

// Initialize - 检查插件需要什么，然后注入
func (i pluginInitializer) Initialize(plugin admission.Interface) {
    // 检查：这个插件需要 Client 吗？
    if wants, ok := plugin.(WantsExternalKubeClientSet); ok {
        wants.SetExternalKubeClientSet(i.client)
    }
    
    // 检查：这个插件需要 InformerFactory 吗？
    if wants, ok := plugin.(WantsExternalKubeInformerFactory); ok {
        wants.SetExternalKubeInformerFactory(i.informerFactory)
    }
    
    // ... 检查其他依赖接口
}

var _ admission.PluginInitializer = pluginInitializer{}
```

通过实现 `admission.PluginInitializer` 接口来进行解耦，调用方只依赖接口，不关心具体实现。

### 5. 创建依赖注入器

```go
// 代码: staging\src\k8s.io\apiserver\pkg\admission\initializer\initializer.go#L42-L60
// New creates an instance of admission plugins initializer.
// This constructor is public with a long param list so that callers immediately know that new information can be expected
// during compilation when they update a level.
func New(
	extClientset kubernetes.Interface,
	dynamicClient dynamic.Interface,
	extInformers informers.SharedInformerFactory,
	authz authorizer.Authorizer,
	featureGates featuregate.FeatureGate,
	stopCh <-chan struct{},
	restMapper meta.RESTMapper,
) pluginInitializer {
	return pluginInitializer{
		externalClient:    extClientset,
		dynamicClient:     dynamicClient,
		externalInformers: extInformers,
		authorizer:        authz,
		featureGates:      featureGates,
		stopCh:            stopCh,
		restMapper:        restMapper,
	}
}
```

把 API Server 启动时创建的各种基础设施（Client、Informer、Authorizer 等）打包成一个注入器，后续遍历所有插件时，根据插件实现的接口，按需分发这些依赖。

### 6. 流程总结

#### 1. 插件注册

```markdown
┌─────────────────────────────────────────────────────────────────────────────┐
│  cmd/kube-apiserver/app/server.go                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  NewAPIServerCommand()                                                      │
│       │                                                                     │
│       └──► s := options.NewServerRunOptions() ──────────────────────────┐   │
│                                                                         │   │
│            返回: *ServerRunOptions                                       │   │
│                                                                         │   │
└─────────────────────────────────────────────────────────────────────────┼───┘
                                                                          │
┌─────────────────────────────────────────────────────────────────────────┼───┐
│  cmd/kube-apiserver/app/options/options.go                              │   │
├─────────────────────────────────────────────────────────────────────────┼───┤
│                                                                         │   │
│  NewServerRunOptions() ◄────────────────────────────────────────────────┘   │
│       │                                                                     │
│       └──► s := ServerRunOptions{                                           │
│                Options: controlplaneapiserver.NewOptions(), ────────────┐   │
│                ...                                                      │   │
│            }                                                            │   │
│                                                                         │   │
│            返回: *ServerRunOptions                                       │   │
│                                                                         │   │
└─────────────────────────────────────────────────────────────────────────┼───┘
                                                                          │
┌─────────────────────────────────────────────────────────────────────────┼───┐
│  pkg/controlplane/apiserver/options/options.go                          │   │
├─────────────────────────────────────────────────────────────────────────┼───┤
│                                                                         │   │
│  NewOptions() ◄─────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       └──► s := Options{                                                    │
│                GenericServerRunOptions: ...,                                │
│                Etcd: ...,                                                   │
│                Admission: kubeoptions.NewAdmissionOptions(), ───────────┐   │
│                ...                                                      │   │
│            }                                                            │   │
│                                                                         │   │
│            返回: *Options                                                │   │
│                                                                         │   │
└─────────────────────────────────────────────────────────────────────────┼───┘
                                                                          │
┌─────────────────────────────────────────────────────────────────────────┼───┐
│  pkg/kubeapiserver/options/admission.go                                 │   │
├─────────────────────────────────────────────────────────────────────────┼───┤
│                                                                         │   │
│  NewAdmissionOptions() ◄────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ├──► options := genericoptions.NewAdmissionOptions() ─────────────┐   │
│       │                                                                 │   │
│       ├──► RegisterAllAdmissionPlugins(options.Plugins) ────────────────┼─┐ │
│       │                                                                 │ │ │
│       ├──► options.RecommendedPluginOrder = AllOrderedPlugins           │ │ │
│       │                                                                 │ │ │
│       └──► options.DefaultOffPlugins = DefaultOffAdmissionPlugins()     │ │ │
│                                                                         │ │ │
│            返回: *AdmissionOptions{GenericAdmission: options}           │ │ │
│                                                                         │ │ │
└─────────────────────────────────────────────────────────────────────────┼─┼─┘
                                                                          │ │
┌─────────────────────────────────────────────────────────────────────────┼─┼─┐
│  staging/src/k8s.io/apiserver/pkg/server/options/admission.go           │ │ │
├─────────────────────────────────────────────────────────────────────────┼─┼─┤
│                                                                         │ │ │
│  NewAdmissionOptions() ◄────────────────────────────────────────────────┘ │ │
│       │                                                                   │ │
│       ├──► options := &AdmissionOptions{                                  │ │
│       │        Plugins: admission.NewPlugins(),  // 空的插件注册中心       │ │
│       │        ...                                                        │ │
│       │    }                                                              │ │
│       │                                                                   │ │
│       └──► server.RegisterAllAdmissionPlugins(options.Plugins)            │ │
│            // 注册 apiserver 通用插件 (lifecycle, webhook 等)              │ │
│                                                                           │ │
│            返回: *AdmissionOptions                                        │ │
│                                                                           │ │
└───────────────────────────────────────────────────────────────────────────┼─┘
                                                                            │
┌───────────────────────────────────────────────────────────────────────────┼─┐
│  pkg/kubeapiserver/options/plugins.go                                     │ │
├───────────────────────────────────────────────────────────────────────────┼─┤
│                                                                           │ │
│  RegisterAllAdmissionPlugins(plugins *admission.Plugins) ◄────────────────┘ │
│       │                                                                     │
│       ├──► admit.Register(plugins)                                          │
│       ├──► alwayspullimages.Register(plugins)                               │
│       ├──► limitranger.Register(plugins)                                    │
│       ├──► autoprovision.Register(plugins)                                  │
│       ├──► exists.Register(plugins)                                         │
│       ├──► resourcequota.Register(plugins)                                  │
│       ├──► serviceaccount.Register(plugins)                                 │
│       └──► ... (更多 kube-apiserver 特有插件)                                │
│                                                                             │
│       // 注意：lifecycle.Register 在 genericoptions 中已调用                 │
│       // 这里注册的是 kube-apiserver 特有的插件                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 每个 Register 函数
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  各插件的 Register 函数 (以 lifecycle 为例)                                  │
│  staging/src/k8s.io/apiserver/pkg/admission/plugin/namespace/lifecycle/     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  func Register(plugins *admission.Plugins) {                                │
│      plugins.Register(PluginName, func(config io.Reader) (Interface, error) │
│          return NewLifecycle(...)  // 工厂函数，此时不调用                   │
│      })                                                                     │
│  }                                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  staging/src/k8s.io/apiserver/pkg/admission/plugins.go                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  func (ps *Plugins) Register(name string, plugin Factory) {                 │
│      ps.lock.Lock()                                                         │
│      defer ps.lock.Unlock()                                                 │
│                                                                             │
│      // 只是存储工厂函数，不调用！                                            │
│      ps.registry[name] = plugin                                             │
│                                                                             │
│      klog.V(1).InfoS("Registered admission plugin", "plugin", name)         │
│  }                                                                          │
│                                                                             │
│  // 最终状态:                                                                │
│  // ps.registry = {                                                         │
│  //     "NamespaceLifecycle": factory,                                      │
│  //     "LimitRanger": factory,                                             │
│  //     "ServiceAccount": factory,                                          │
│  //     ...                                                                 │
│  // }                                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

注意，`func Register(plugins *admission.Plugins) `函数返回的工厂函数，并不进行调用，而是采用延迟调用，因为需要等待其他资源准备就绪。而 `plugins.Register` 函数源码如下：

```go
// Register registers a plugin Factory by name. This
// is expected to happen during app startup.
func (ps *Plugins) Register(name string, plugin Factory) {
	ps.lock.Lock()
	defer ps.lock.Unlock()
	if ps.registry != nil {
		_, found := ps.registry[name]
		if found {
			klog.Fatalf("Admission plugin %q was registered twice", name)
		}
	} else {
		ps.registry = map[string]Factory{}
	}

	klog.V(1).InfoS("Registered admission plugin", "plugin", name)
    // 只是把工厂函数存到 map 里，还没有调用！
	ps.registry[name] = plugin
}
```

此阶段结束时：

```go
Plugins.registry = {
    "NamespaceLifecycle": func(config) → NewLifecycle(),  // 工厂函数，还没调用
    "ValidatingWebhook":  func(config) → ...,
    "MutatingWebhook":    func(config) → ...,
    // ...
}

此时：
- 插件实例还不存在！
- 只是注册了"如何创建插件"的工厂函数
```

#### 2. 创建基础设施 + PluginInitializer

```markdown
┌─────────────────────────────────────────────────────────────────────────────┐
│  cmd/kube-apiserver/app/server.go                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  NewAPIServerCommand()                                                      │
│       │                                                                     │
│       └──► s := options.NewServerRunOptions()  ← Phase1: 插件注册           │
│                                                                             │
│  cmd.RunE = func() {                                                        │
│       │                                                                     │
│       ├──► completedOptions, _ := s.Complete(ctx)                           │
│       │                                                                     │
│       └──► Run(ctx, completedOptions) ──────────────────────────────────┐   │
│  }                                                                      │   │
│                                                                         │   │
└─────────────────────────────────────────────────────────────────────────┼───┘
                                                                          │
┌─────────────────────────────────────────────────────────────────────────┼───┐
│  cmd/kube-apiserver/app/server.go                                       │   │
├─────────────────────────────────────────────────────────────────────────┼───┤
│                                                                         │   │
│  Run(ctx, opts CompletedOptions) ◄──────────────────────────────────────┘   │
│       │                                                                     │
│       ├──► config, _ := NewConfig(opts) ────────────────────────────────┐   │
│       │                                                                 │   │
│       ├──► completed, _ := config.Complete()                            │   │
│       │                                                                 │   │
│       ├──► server, _ := CreateServerChain(completed)                    │   │
│       │                                                                 │   │
│       └──► prepared.Run(ctx)                                            │   │
│                                                                         │   │
└─────────────────────────────────────────────────────────────────────────┼───┘
                                                                          │
┌─────────────────────────────────────────────────────────────────────────┼───┐
│  cmd/kube-apiserver/app/config.go                                       │   │
├─────────────────────────────────────────────────────────────────────────┼───┤
│                                                                         │   │
│  NewConfig(opts CompletedOptions) ◄─────────────────────────────────────┘   │
│       │                                                                     │
│       ├──► genericConfig, versionedInformers, storageFactory, _ :=          │
│       │        controlplaneapiserver.BuildGenericConfig(...) ───────────┐   │
│       │                                                                 │   │
│       └──► kubeAPIs, serviceResolver, pluginInitializer, _ :=           │   │
│                CreateKubeAPIServerConfig(opts, genericConfig,           │   │
│                    versionedInformers, storageFactory) ─────────────────┼─┐ │
│                                                                         │ │ │
│            返回: *Config                                                │ │ │
│                                                                         │ │ │
└─────────────────────────────────────────────────────────────────────────┼─┼─┘
                                                                          │ │
┌─────────────────────────────────────────────────────────────────────────┼─┼─┐
│  pkg/controlplane/apiserver/config.go                                   │ │ │
├─────────────────────────────────────────────────────────────────────────┼─┼─┤
│                                                                         │ │ │
│  BuildGenericConfig(s CompletedOptions, ...) ◄──────────────────────────┘ │ │
│       │                                                                   │ │
│       ├──► genericConfig = genericapiserver.NewConfig(...)                │ │
│       │                                                                   │ │
│       ├──► kubeClientConfig := genericConfig.LoopbackClientConfig         │ │
│       │                                                                   │ │
│       ├──► clientgoExternalClient, _ := clientgoclientset.NewForConfig()  │ │
│       │    // 创建 Kubernetes Client ✓                                    │ │
│       │                                                                   │ │
│       ├──► versionedInformers = clientgoinformers.NewSharedInformerFactory│ │
│       │    // 创建 InformerFactory ✓                                      │ │
│       │                                                                   │ │
│       └──► 返回: genericConfig, versionedInformers, storageFactory        │ │
│                                                                           │ │
└───────────────────────────────────────────────────────────────────────────┼─┘
                                                                            │
┌───────────────────────────────────────────────────────────────────────────┼─┐
│  cmd/kube-apiserver/app/server.go                                         │ │
├───────────────────────────────────────────────────────────────────────────┼─┤
│                                                                           │ │
│  CreateKubeAPIServerConfig(opts, genericConfig, versionedInformers, ...) ◄┘ │
│       │                                                                     │
│       └──► controlplaneConfig, admissionInitializers, _ :=                  │
│                controlplaneapiserver.CreateConfig(                          │
│                    opts.CompletedOptions,                                   │
│                    genericConfig,                                           │
│                    versionedInformers,  // InformerFactory 传入             │
│                    storageFactory,                                          │
│                    serviceResolver,                                         │
│                    kubeInitializers,                                        │
│                ) ───────────────────────────────────────────────────────┐   │
│                                                                         │   │
└─────────────────────────────────────────────────────────────────────────┼───┘
                                                                          │
┌─────────────────────────────────────────────────────────────────────────┼───┐
│  pkg/controlplane/apiserver/config.go                                   │   │
├─────────────────────────────────────────────────────────────────────────┼───┤
│                                                                         │   │
│  CreateConfig(opts, genericConfig, versionedInformers, ...) ◄───────────┘   │
│       │                                                                     │
│       │  // 创建 Client                                                     │
│       ├──► clientgoExternalClient, _ := clientgoclientset.NewForConfig(     │
│       │        genericConfig.LoopbackClientConfig)                          │
│       │                                                                     │
│       │  // 创建 DynamicClient                                              │
│       ├──► dynamicExternalClient, _ := dynamic.NewForConfig(                │
│       │        genericConfig.LoopbackClientConfig)                          │
│       │                                                                     │
│       │  // ★★★ 关键调用：ApplyTo ★★★                                       │
│       └──► err = opts.Admission.ApplyTo(                                    │
│                genericConfig,           // server config                    │
│                versionedInformers,      // InformerFactory ✓                │
│                clientgoExternalClient,  // Client ✓                         │
│                dynamicExternalClient,   // DynamicClient ✓                  │
│                utilfeature.DefaultFeatureGate,                              │
│                append(genericInitializers, additionalInitializers...)...,   │
│            ) ───────────────────────────────────────────────────────────┐   │
│                                                                         │   │
└─────────────────────────────────────────────────────────────────────────┼───┘
                                                                          │
┌─────────────────────────────────────────────────────────────────────────┼───┐
│  pkg/kubeapiserver/options/admission.go                                 │   │
├─────────────────────────────────────────────────────────────────────────┼───┤
│                                                                         │   │
│  (a *AdmissionOptions) ApplyTo(...) ◄───────────────────────────────────┘   │
│       │                                                                     │
│       └──► return a.GenericAdmission.ApplyTo(                               │
│                c, informers, kubeClient, dynamicClient,                     │
│                features, pluginInitializers...,                             │
│            ) ───────────────────────────────────────────────────────────┐   │
│                                                                         │   │
└─────────────────────────────────────────────────────────────────────────┼───┘
                                                                          │
┌─────────────────────────────────────────────────────────────────────────┼───┐
│  staging/src/k8s.io/apiserver/pkg/server/options/admission.go           │   │
├─────────────────────────────────────────────────────────────────────────┼───┤
│                                                                         │   │
│  (a *AdmissionOptions) ApplyTo(c, informers, kubeClient, ...) ◄─────────┘   │
│       │                                                                     │
│       │  // 创建 PluginInitializer（依赖注入器）                             │
│       ├──► genericInitializer := initializer.New(                           │
│       │        kubeClient,       // Client                                  │
│       │        dynamicClient,    // DynamicClient                           │
│       │        informers,        // InformerFactory                         │
│       │        c.Authorization.Authorizer,                                  │
│       │        features,                                                    │
│       │        c.DrainedNotify(),                                           │
│       │        discoveryRESTMapper,                                         │
│       │    )                                                                │
│       │                                                                     │
│       │  // 组装 Initializer 链                                             │
│       ├──► initializersChain := admission.PluginInitializers{               │
│       │        genericInitializer}                                          │
│       │    initializersChain = append(initializersChain, pluginInitializers)│
│       │                                                                     │
│       │  // ★★★ 创建并初始化所有插件 ★★★                                     │
│       └──► admissionChain, _ := a.Plugins.NewFromPlugins(                   │
│                pluginNames,          // 插件名称列表                         │
│                pluginsConfigProvider,                                       │
│                initializersChain,    // 依赖注入器                           │
│                a.Decorators,                                                │
│            ) ───────────────────────────────────────────────────────────┐   │
│                                                                         │   │
│       c.AdmissionControl = admissionChain  // 设置到 server config      │   │
│                                                                         │   │
└─────────────────────────────────────────────────────────────────────────┼───┘
                                                                          │
                                                                          │
                              ┌────────────────────────────────────────────┘
                              │
                              ▼
                    Phase 3: 插件实例化 + 依赖注入
                    (NewFromPlugins → InitPlugin → Initialize)

```

#### 3. 插件实例化 + 依赖注入

```go
// NewFromPlugins returns an admission.Interface that will enforce admission control decisions of all
// the given plugins.
func (ps *Plugins) NewFromPlugins(pluginNames []string, configProvider ConfigProvider, pluginInitializer PluginInitializer, decorator Decorator) (Interface, error) {
	handlers := []Interface{}
	mutationPlugins := []string{}
	validationPlugins := []string{}
	for _, pluginName := range pluginNames {
         // 获取插件配置
		pluginConfig, err := configProvider.ConfigFor(pluginName)
		if err != nil {
			return nil, err
		}

        // 关键，初始化单个插件
		plugin, err := ps.InitPlugin(pluginName, pluginConfig, pluginInitializer)
		if err != nil {
			return nil, err
		}
		if plugin != nil {
			if decorator != nil {
				handlers = append(handlers, decorator.Decorate(plugin, pluginName))
			} else {
				handlers = append(handlers, plugin)
			}

			if _, ok := plugin.(MutationInterface); ok {
				mutationPlugins = append(mutationPlugins, pluginName)
			}
			if _, ok := plugin.(ValidationInterface); ok {
				validationPlugins = append(validationPlugins, pluginName)
			}
		}
	}
	if len(mutationPlugins) != 0 {
		klog.Infof("Loaded %d mutating admission controller(s) successfully in the following order: %s.", len(mutationPlugins), strings.Join(mutationPlugins, ","))
	}
	if len(validationPlugins) != 0 {
		klog.Infof("Loaded %d validating admission controller(s) successfully in the following order: %s.", len(validationPlugins), strings.Join(validationPlugins, ","))
	}
	return newReinvocationHandler(chainAdmissionHandler(handlers)), nil
}
```

```go
// InitPlugin creates an instance of the named interface.
func (ps *Plugins) InitPlugin(name string, config io.Reader, pluginInitializer PluginInitializer) (Interface, error) {
	if name == "" {
		klog.Info("No admission plugin specified.")
		return nil, nil
	}
    
    // Step 1: 调用工厂函数，创建插件实例
	plugin, found, err := ps.getPlugin(name, config)
	if err != nil {
		return nil, fmt.Errorf("couldn't init admission plugin %q: %v", name, err)
	}
	if !found {
		return nil, fmt.Errorf("unknown admission plugin: %s", name)
	}

    // Step 2: 依赖注入！
	pluginInitializer.Initialize(plugin)
	// ensure that plugins have been properly initialized
    // Step 3: 验证初始化是否成功
	if err := ValidateInitialization(plugin); err != nil {
		return nil, fmt.Errorf("failed to initialize admission plugin %q: %v", name, err)
	}

	return plugin, nil
}
```

在 Step 1. 中，即调用 流程1 结束后保存在 `Plugins.registry` 中的工厂函数，得到插件空壳，然后在 `Step2` 中，调用 `pluginInitializer` 结构体的 `Initialize` 方法进行依赖注入，通过验证后，即可得到完整的 `plugin` 了。

## 内部实现原理

准入控制器的执行时机是在资源对象持久化之前，在代码实现上位于 RESTStorage 存储层。在 InstallAPIGroup 注册 APIGroup 阶段，InstallREST 在注册资源操作相关 Handler 时，会自动嵌入准入控制逻辑。以 KubeAPIServer InstallAPIGroups 为例，按照  `InstallAPIGroups`→`s.installAPIResources`→`apiGroupVersion.InstallREST`→`installer.Install`→`a.registerResourceHandlers` 的调用链进行追踪，准入控制器 Handler 注册的代码示例如下。

其 `registerResourceHandlers` 的详细讲解见：[K8s Generic Server WebService](/posts/cloud-native/appendix/06丨K8s-Generic-Server-WebService)，不过 WebService 主要以注入 API 为核心，也只讲解了没有准入控制器控制的读取对象的请求【GET】，这里来详细介绍准入控制器。

```go
func (a *APIInstaller) registerResourceHandlers() () {
	admit := a.group.Admit
	...
	restfulUpdateResource(updater, reqScope, admit)
	restfulPatchResource(patcher, reqScope, admit, supportedTypes)
	restfulCreateNamedResource(namedCreater, reqScope, admit)
	restfulCreateResource(creater, reqScope, admit)
	restfulDeleteResource(gracefulDeleter, isGracefulDeleter, reqScope, admit)
	restfulDeleteCollection(collectionDeleter, isCollectionDeleter, reqScope, admit)
	restfulConnectResource(connecter, reqScope, admit, path, isSubresource)
	...
} 
```

在对资源进行上述操作（Update、Patch、Create、Delete、Connect）时，会先执行 Admission 准入控制逻辑，即执行 Mutation 和（或）Validation 逻辑，通过后才真正操作 etcd 存储资源。

这再次印证了，准入控制器只拦截创建、删除、修改对象的请求，以及阻止自定义操作（如通过 API 服务器代理连接 Pod 的请求），但不会也不能阻止读取对象的请求（如 Get、List、Watch）。

以 `restfulCreateResource` 资源创建为例，准入控制器的执行逻辑代码示例如下。

```go
// 代码: staging\src\k8s.io\apiserver\pkg\endpoints\installer.go
case request.MethodPost: // Create a resource.
    var handler restful.RouteFunction
    if isNamedCreater {
        handler = restfulCreateNamedResource(namedCreater, reqScope, admit) 
    } else {
        handler = restfulCreateResource(creater, reqScope, admit) // #L925
    }
```

Create 的 handler 的生成依赖 `restfulCreateResource` / `restfulCreateNamedResource`，以 `restfulCreateResource` 为例来进行查看，

```go
// 代码: staging\src\k8s.io\apiserver\pkg\endpoints\installer.go#L1304-L1308
func restfulCreateResource(r rest.Creater, scope handlers.RequestScope, admit admission.Interface) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.CreateResource(r, &scope, admit)(res.ResponseWriter, req.Request)
	}
}
```

`restfulCreateResource` 实际上就是一个适配器，之前提到过，可以学习一下。其调用 `handlers.CreateResource(r, &scope, admit)` 来生成标准的 `HTTP` 请求。再来看 `handlers.CreateResource` 函数：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\endpoints\handlers\create.go#L245-L247
// CreateResource returns a function that will handle a resource creation.
func CreateResource(r rest.Creater, scope *RequestScope, admission admission.Interface) http.HandlerFunc {
	return createHandler(&namedCreaterAdapter{r}, scope, admission, false)
}
```

`CreateResource` 直接返回 `createHandler(&namedCreaterAdapter{r}, scope, admission, false)`，再看 `createHandler` 函数，不过该函数很长，我们只看和 admit 相关的部分：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\endpoints\handlers\create.go#L53-L237
func createHandler(r rest.NamedCreater, scope *RequestScope, admit admission.Interface, includeName bool) http.HandlerFunc {
    return func(w http.ResponseWriter, req *http.Request) {
        // ...
        
        requestFunc := func() (runtime.Object, error) { //要点①
            return r.Create(
                ctx,
                name,
                obj,
                rest.AdmissionToValidateObjectFunc(admit, admissionAttributes, scope),
                options,
            )
        }
        
        // ...

        if mutatingAdmission, ok := admit.(admission.MutationInterface); ok && mutatingAdmission.Handles(admission.Create) { //要点②
            mutatingAdmission.Admit(ctx, admissionAttributes, scope) //要点③
            // ...
        }

        // ...
        
        result, err := requestFunc() // 要点④
        
        // ...
    }
}
```

```go
// 代码: staging\src\k8s.io\apiserver\pkg\registry\rest\create.go#L191-L223
func AdmissionToValidateObjectFunc(admit admission.Interface, staticAttributes admission.Attributes, o admission.ObjectInterfaces) ValidateObjectFunc {
    validatingAdmission, ok := admit.(admission.ValidationInterface)
    // ...
    if !validatingAdmission.Handles(finalAttributes.GetOperation()) {
        return nil
    }

    return validatingAdmission.Validate(ctx, finalAttributes, o)
}
```

1. 首先通过要点①处，准备好验证资源对象的函数，即 `requestFunc`；
2. 在资源对象注册 Handler 的过程中，先通过要点②的类型断言 `admit.(admission.MutationInterface)`判断准入控制器是否支持变更资源对象，并且进一步通过 `mutatingAdmission.Handles(admission.Create)` 来判断是否支持Create 类型的准入控制；
3. 如果都支持，则调用 `mutatingAdmission.Admit` 执行准入控制操作，变更资源对象。
4. 接着执行要点④，即要点①准备的验证函数，在执行数据持久化前，通过类型断言 `admit.(admission.ValidationInterface)` 判断准入控制器是否支持验证资源对象，并且进一步判断是否支持 `finalAttributes.GetOperation` 函数返回的操作类型，如果支持，则调用 `validatingAdmission.Validate` 执行准入控制验证操作。
5. 当变更准入控制器和验证准入控制器都通过后，资源对象才会被真正持久化到 etcd 底层存储。

`admission.MutationInterface` 和 `admission.ValidationInterface` 定义了 Admission 的变更准入控制器和 验证准入控制器需要实现的接口。

由上述处理逻辑可以看出，准入控制过程可以划分为两个阶段：

* 第一阶段，执行变更操作；
* 第二阶段，执行验证操作。

在实现上，某些准入控制器既是变更准入控制器，又是验证准入控制器。在**执行顺序**上，先执行**变更准入控制器**，**再执行验证准入控制器**。如果两个阶段中的任何一个准入控制器拒绝了请求，则请求被拒绝，并且向最终用户返回错误。准入控制器的执行流程如图所示。

![image-20251208213428274](http://images.liangning7.cn/typora/202512082134540.png)

在请求经过认证、授权等前置验证，到达准入控制器后，首先执行 Admit 准入控制，调用支持变更操作的入控制器对资源对象进行变更，以符合存储要求。经过修改的资源对象会继续被执行 Validate 准入控制，调用支持验证操作的准入控制器对资源对象的合法性进行校验，以符合存储要求。只有通过所有准入控制器的修改和验证检查的资源对象，才能真正被持久化到 etcd 存储。

在组织形式上，kube-apiserver 将所有已启用的准入控制器组织成链式结构，通过遍历的方式依次调用，代码示例如下。

```go
// 代码: staging\src\k8s.io\apiserver\pkg\admission\chain.go#L30-L60
// Admit performs an admission control check using a chain of handlers, and returns immediately on first error
func (admissionHandler chainAdmissionHandler) Admit(ctx context.Context, a Attributes, o ObjectInterfaces) error {
	for _, handler := range admissionHandler {
		if !handler.Handles(a.GetOperation()) {
			continue
		}
		if mutator, ok := handler.(MutationInterface); ok {
			err := mutator.Admit(ctx, a, o)
			if err != nil {
				return err
			}
		}
	}
	return nil
}

// Validate performs an admission control check using a chain of handlers, and returns immediately on first error
func (admissionHandler chainAdmissionHandler) Validate(ctx context.Context, a Attributes, o ObjectInterfaces) error {
	for _, handler := range admissionHandler {
		if !handler.Handles(a.GetOperation()) {
			continue
		}
		if validator, ok := handler.(ValidationInterface); ok {
			err := validator.Validate(ctx, a, o)
			if err != nil {
				return err
			}
		}
	}
	return nil
}
```

`chainAdmissionHandler` 以数组的形式组织已经启用的准入控制器（遵循内置的推荐执行顺序排列）

在调用 Admit 和 Validate 函数时，采用 range 循环，依次判断当前准入控制器是否具备处理条件，如果具备，则调用其准入控制处理逻辑。

内置的推荐执行顺序定义如下：

```go
// 代码: pkg\kubeapiserver\options\plugins.go#L69-L110
// AllOrderedPlugins is the list of all the plugins in order.
var AllOrderedPlugins = []string{
	admit.PluginName,                        // AlwaysAdmit
	autoprovision.PluginName,                // NamespaceAutoProvision
	lifecycle.PluginName,                    // NamespaceLifecycle
	exists.PluginName,                       // NamespaceExists
	antiaffinity.PluginName,                 // LimitPodHardAntiAffinityTopology
	limitranger.PluginName,                  // LimitRanger
	serviceaccount.PluginName,               // ServiceAccount
	noderestriction.PluginName,              // NodeRestriction
	nodetaint.PluginName,                    // TaintNodesByCondition
	alwayspullimages.PluginName,             // AlwaysPullImages
	imagepolicy.PluginName,                  // ImagePolicyWebhook
	podsecurity.PluginName,                  // PodSecurity
	podnodeselector.PluginName,              // PodNodeSelector
	podpriority.PluginName,                  // Priority
	defaulttolerationseconds.PluginName,     // DefaultTolerationSeconds
	podtolerationrestriction.PluginName,     // PodTolerationRestriction
	eventratelimit.PluginName,               // EventRateLimit
	extendedresourcetoleration.PluginName,   // ExtendedResourceToleration
	setdefault.PluginName,                   // DefaultStorageClass
	storageobjectinuseprotection.PluginName, // StorageObjectInUseProtection
	gc.PluginName,                           // OwnerReferencesPermissionEnforcement
	resize.PluginName,                       // PersistentVolumeClaimResize
	runtimeclass.PluginName,                 // RuntimeClass
	certapproval.PluginName,                 // CertificateApproval
	certsigning.PluginName,                  // CertificateSigning
	ctbattest.PluginName,                    // ClusterTrustBundleAttest
	certsubjectrestriction.PluginName,       // CertificateSubjectRestriction
	defaultingressclass.PluginName,          // DefaultIngressClass
	denyserviceexternalips.PluginName,       // DenyServiceExternalIPs
	podtopologylabels.PluginName,            // PodTopologyLabels

	// new admission plugins should generally be inserted above here
	// webhook, resourcequota, and deny plugins must go at the end

	mutatingadmissionpolicy.PluginName,   // MutatingAdmissionPolicy
	mutatingwebhook.PluginName,           // MutatingAdmissionWebhook
	validatingadmissionpolicy.PluginName, // ValidatingAdmissionPolicy
	validatingwebhook.PluginName,         // ValidatingAdmissionWebhook
	resourcequota.PluginName,             // ResourceQuota
	deny.PluginName,                      // AlwaysDeny
}
```

## 调用链

在上层 Server 将 Admit 准备完毕后，交给 Generic Server 后，该数据流程如下：

1. `GenericAPIServer` 结构体中有一个 `admissionControl` 字段（类型为 `admission.Interface`）

   ```go
   // 代码: staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go#L128
   // admissionControl is used to build the RESTStorage that backs an API Group.
   admissionControl admission.Interface
   ```

2. 在 `GenericAPIServer` 初始化时，`admissionControl` 从 `Config.AdmissionControl` 赋值。

   ```go
   // 代码: staging/src/k8s.io/apiserver/pkg/server/config.go#L808
   s := &GenericAPIServer{
       discoveryAddresses:             c.DiscoveryAddresses,
       LoopbackClientConfig:           c.LoopbackClientConfig,
       legacyAPIGroupPrefixes:         c.LegacyAPIGroupPrefixes,
       admissionControl:               c.AdmissionControl, // 赋值
   ```

3. 当安装 API 资源时，GenericAPIServer 创建 APIGroupVersion 对象，并将 admissionControl 传递给它

   ```go
   // 代码: staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go#L996
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
   
       Admit:             s.admissionControl, // 传递
       MinRequestTimeout: s.minRequestTimeout,
       Authorizer:        s.Authorizer,
   }
   ```

4. APIInstaller 从 APIGroupVersion 获取 Admit 字段

   ```go
   // 代码: staging/src/k8s.io/apiserver/pkg/endpoints/installer.go#L286
   func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
   	admit := a.group.Admit // 获得 admit
   ```

5. 最后，这个 admit 被传递给各种 handler 函数（如 restfulCreateNamedResource）用于准入控制

所以整个流程是：`GenericAPIServer.admissionControl` → `APIGroupVersion.Admit` → `APIInstaller` 中的 `admit` 变量 → 各种 `REST handler`。

## 动态准入控制器

除了内置的已经固化处理逻辑的准入控制器，为了提升可扩展性，kube-apiserver 提供了两种动态准入控制器，同时分别提供了两种资源类型专门服务于这两种准入控制器的配置，分别如下。

* `MutatingAdmissionWebhook`：通过 Webhook 扩展的变更准入控制器，可以通过 Webhook 自定义变更处理逻辑，该准入控制器使用 `MutatingWebhookConfiguration` 资源对象进行配置。
* `ValidatingAdmissionWebhook`：通过Webhook 扩展的验证准入控制器，可以通过Webhook 自定义验证处理逻辑，该准入控制器使用 `ValidatingWebhookConfiguration` 资源对象进行配置。

通过安装 `MutatingWebhookConfiguration` 和（或）`ValidatingWebhookConfiguration` 资源对象，kube-apiserver 就能通过监听机制自动发现和动态载入通过 Webhook 扩展的准入控制器，根据配置中定义的对某些特定资源的某些特定操作进行拦截，通过远端 Webhook 服务器实现对资源进行某些变更或验证逻辑，并且基于返回结果选择接受或拒绝客户端的资源操作请求，从而达到动态扩展准入控制器功能的目的。

在执行顺序上，默认的内置类型的准入控制器一般具有更高的优先级，通过 Webhook 扩展的准入控制器在处理链条的末端，仅在 ResourceQuota 和 AlwaysDeny 之前。

在相对顺序上， 先执行变更类型的准入控制器，再执行验证类型的准入控制器。相同类型的准入控制器，如`MutatingAdmissionWebhook`，默认按照名字字典顺序执行。通过 Webhook 扩展的准入控制器的执行位置如下图所示。

![image-20251209144604533](http://images.liangning7.cn/typora/202512091446848.png)

### MutatingAdmissionWebhook

`MutatingAdmissionWebhook` 是一种插件式变更准入控制器，能够在不改变 kube-apiserver 源码的情况下，扩展准入控制器功能，允许用户定制 Webhook Admission Server 服务，通过 MutatingWebhookConfiguration 动态配置拦截规则，实现自定义的准入控制处理逻辑，对传入的匹配规则的资源对象在真正持久化前通过 Webhook 执行变更操作。

`MutatingAdmissionWebhook` 准入控制器的运行原理如图所示。

![image-20251209144733313](http://images.liangning7.cn/typora/202512091447546.png)

`MutatingAdmissionWebhook` 准入控制器根据逻辑可以划分为两个主要部分。

* MutatingAdmissionWebhook 配置发现。即上图的左侧所示，MutatingAdmissionWebhook 准入控制器在初始化阶段，会注册事件监听，通过 Informer 监听集群中的 MutatingWebhookConfiguration 资源对象事件，基于事件触发 MutatingWebhookConfiguration-Manager 同步更新内部维护的 WebhookAccessor 列表，使其始终处于最新状态。WebhookAccessor 列表保存了 MutatingAdmissionWebhook 准入控制器执行 Admit 处理时依赖的必要配置，是 MutatingAdmissionWebhook 配置发现阶段的产物，也是 Admit 准入控制执行阶段消费的资料。
* Admit 准入控制执行：如上图的右侧所示，在初始化阶段，MutatingAdmissionWebhook 的 Handler 已经注册到 kube-apiserver 的处理链条。当收到客户端发起的资源请求时，kube-apiserver 按照顺序执行处理链条上的 Handler（如认证、鉴权等），最终触发 MutatingAdmissionWebhook 准入控制器执行。在执行 Admit 准入控制时，MutatingAdmissionWebhook 准入控制器会通过 Webhooks 函数调用获取当前系统配置的 WebhookAccessor 列表，结合传入的资源对象属性，判断是否需要发起 Webhook 调用。如果需要，则构建 AdmissionReview 请求对象，向远端 Webhook Admission Server 发起 HTTP 调用，根据返回结果进行后续的操作。

MutatingAdmissionWebhook 判断是否需要对资源对象执行 Admit 准入控制，依赖两个关键输入：**规则配置**（MutatingWebhookConfiguration）和**资源对象属性**（`admission.Attributes`）。

一个典型的 `MutatingWebhookConfiguration` 配置示例如下。

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: mutating-webhook-example # 配置的名称，用于唯一标识这个 webhook 配置
webhooks:
  - name: mutate.example.com # Webhook 的名称，必须是唯一的
    clientConfig: # 配置 webhook 服务的连接信息
      service:
        name: mutating-webhook-service # Webhook 服务的名称
        namespace: default # Webhook 服务所在的命名空间
        path: "/mutate" # Webhook 服务的路径
      # caBundle 这里应该是 Base64 编码的 CA 证书，用于校验 webhook 服务的 TLS 证书
      caBundle: "YOUR_CA_BUNDLE"
    rules: #  定义 webhook 应用的规则
      - operations: ["CREATE", "UPDATE"] # Webhook 触发的操作类型，如 CREATE、UPDATE。还有一个 DELETE 操作类型
        apiGroups: [""] # 目标资源所在的 API 组
        apiVersions: ["v1"] # 目标资源的 API 版本
        resources: ["pods"] # 目标资源类型，如 pods
    failurePolicy: Fail # Webhook 服务失败时的策略，Fail 或 Ignore
    matchPolicy: Exact # 资源匹配策略，Exact 或 Equivalent
    namespaceSelector: # 基于命名空间标签选择应用于哪些命名空间
      matchLabels:
        webhook: enabled
    objectSelector: # 基于资源标签选择应用于哪些资源
      matchLabels:
        apply-webhook: "true"
    sideEffects: None # Webhook 的副作用，None、Some 或 Unknown
    timeoutSeconds: 10 # Webhook 请求的超时时间（秒）
    admissionReviewVersions: ["v1", "v1beta1"] # 支持的 AdmissionReview 版本
    reinvocationPolicy: Never # 指定是否允许重复调用准入控制器
```

`Attributes` 接口的定义如下。

```go
type Attributes interface {
	GetName() string
	GetNamespace() string
	GetResource() schema.GroupVersionResource
	GetSubresource() string
	GetOperation() Operation
	GetOperationOptions() runtime.Object
	IsDryRun() bool
	GetObject() runtime.Object
	GetOldObject() runtime.Object
	GetKind() schema.GroupVersionKind
	GetUserInfo() user.Info
	AddAnnotation(key, value string) error
	AddAnnotationWithLevel(key, value string, level auditinternal.Level) error
	GetReinvocationContext() ReinvocationContext
} 
```

`Attributes` 接口定义了一系列读取资源请求信息的接口，用来判断是否满足 `MutatingWebhookConfiguration` 的匹配要求。

MutatingAdmissionWebhook 插件的初始化及 Admit 函数的代码示例如下。

```go
// 代码: staging\src\k8s.io\apiserver\pkg\admission\plugin\webhook\mutating\plugin.go#L53-L63
// NewMutatingWebhook returns a generic admission webhook plugin.
func NewMutatingWebhook(configFile io.Reader) (*Plugin, error) {
	handler := admission.NewHandler(admission.Connect, admission.Create, admission.Delete, admission.Update)
	p := &Plugin{}
	var err error
	p.Webhook, err = generic.NewWebhook(handler, configFile, configuration.NewMutatingWebhookConfigurationManager, newMutatingDispatcher(p)) // 要点①
	if err != nil {
		return nil, err
	}

	return p, nil
}

// 代码: staging\src\k8s.io\apiserver\pkg\admission\plugin\webhook\mutating\plugin.go#L74-L76
// Admit makes an admission decision based on the request attributes.
func (a *Plugin) Admit(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces) error { // 要点②
	return a.Webhook.Dispatch(ctx, attr, o) //要点③
}
```

MutatingAdmissionWebhook 底层基于 `generic.Webhook` 实现，`generic.Webhook` 是 Webhook 的通用基础结构，是 `Mutating Webhook` 和 `Validating Webhook` 的共同基类。MutatingAdmissionWebhook 的 `Admit` 函数直接调用了 `generic.Webhook` 的 `Dispatch` 函数。

```go
// Dispatch is called by the downstream Validate or Admit methods.
func (a *Webhook) Dispatch(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces) error {
	if rules.IsExemptAdmissionConfigurationResource(attr) { // 要点①
		return nil
	}
	if !a.WaitForReady() { // 要点②
		return admission.NewForbidden(attr, fmt.Errorf("not yet ready to handle request"))
	}
	hooks := a.hookSource.Webhooks() // 要点③
	return a.dispatcher.Dispatch(ctx, attr, o, hooks) // 要点④
}
```

1. `generic.Webhook` 的 Dispatch 函数的第一步就是检查请求的资源类型是否是 `admissionregistration.k8s.io` 资源组下的 ValidatingWebhookConfiguration 或 MutatingWebhookConfiguration，如果是，则直接跳过准入控制器执行。

2. 然后检查准入控制器是否就绪，即其 Informer 是否处于 Synced 同步成功状态，如果未就绪，则返回失败错误。完整的启动流程如下：

   ```go
   // API Server 启动
   //   ↓
   // 代码: staging/src/k8s.io/apiserver/pkg/server/options/admission.go:132
   AdmissionOptions.ApplyTo(c, informers, kubeClient, dynamicClient, features, pluginInitializers...)
   //  ↓
   // 代码: staging/src/k8s.io/apiserver/pkg/server/options/admission.go:157-160
   // 创建 genericInitializer
   genericInitializer = initializer.New(kubeClient, dynamicClient, informers, ...)
   initializersChain := admission.PluginInitializers{genericInitializer}
   initializersChain = append(initializersChain, pluginInitializers...)
   //   ↓
   // 代码: staging/src/k8s.io/apiserver/pkg/server/options/admission.go:173
   admissionChain, err := a.Plugins.NewFromPlugins(pluginNames, pluginsConfigProvider, initializersChain, a.Decorators)
     ↓
   // 代码: staging/src/k8s.io/apiserver/pkg/admission/plugins.go:137
   plugin, err := ps.InitPlugin(pluginName, pluginConfig, pluginInitializer)
   //   ↓
   // 代码: staging/src/k8s.io/apiserver/pkg/admission/plugins.go:180
   pluginInitializer.Initialize(plugin)
     ↓
   // 代码: staging/src/k8s.io/apiserver/pkg/admission/plugins.go:203-207
   func (pp PluginInitializers) Initialize(plugin Interface) {
       for _, p := range pp {
           p.Initialize(plugin)  // 遍历所有初始化器
       }
   }
   //   ↓
   // 代码: staging/src/k8s.io/apiserver/pkg/admission/initializer/initializer.go:66-93
   func (i pluginInitializer) Initialize(plugin admission.Interface) {
       // ... 其他初始化 ...
       
       if wants, ok := plugin.(WantsExternalKubeInformerFactory); ok {
           wants.SetExternalKubeInformerFactory(i.externalInformers)  // 第85行
       }
   }
   //   ↓
   // 代码: staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/generic/webhook.go:130-136
   func (a *Webhook) SetExternalKubeInformerFactory(f informers.SharedInformerFactory) {
       namespaceInformer := f.Core().V1().Namespaces()
       a.namespaceMatcher.NamespaceLister = namespaceInformer.Lister()
       a.hookSource = a.sourceFactory(f)
       a.SetReadyFunc(func() bool {
           return namespaceInformer.Informer().HasSynced() && a.hookSource.HasSynced()
       })
   }
   //   ↓
   // 代码: staging/src/k8s.io/apiserver/pkg/admission/handler.go:59-61
   func (h *Handler) SetReadyFunc(readyFunc ReadyFunc) {
       h.readyFunc = readyFunc
   }
   ```

3. 最后通过 `a.hookSource.Webhooks` 函数读取已经发现并同步 的所有 Webhook 列表，并且调用 `a.dispatcher.Dispatch` 执行 MutatingAdmissionWebhook 插件自定义的 Dispatch 函数。

在这里，`a.hookSource.Webhooks` 函数读取的是 MutatingWebhookConfigurationManager 生产的 WebhookAccessor 列表，基于 MutatingWebhookConfiguration 的 `meta.name` 按字典顺序排序，因此在执行阶段也会默认按照该顺序执行（在不考虑 reinvocation 的情况下），排序代码示例如下。

```go
// 代码: staging\src\k8s.io\apiserver\pkg\admission\configuration\validating_webhook_manager.go#L151-L155
type ValidatingWebhookConfigurationSorter []*v1.ValidatingWebhookConfiguration

func (a ValidatingWebhookConfigurationSorter) ByName(i, j int) bool {
	return a[i].Name < a[j].Name
}
```

MutatingAdmissionWebhook 插件的 Dispatch 函数是真正执行变更准入控制器的主体，其代码示例如下。

```go
// 代码: staging\src\k8s.io\apiserver\pkg\admission\plugin\webhook\mutating\dispatcher.go#L105-L240
func (a *mutatingDispatcher) Dispatch(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces, hooks []webhook.WebhookAccessor) error {
	// ...
	for i, hook := range hooks {
		// ...
		invocation, statusErr := a.plugin.ShouldCallHook(hook, attrForCheck, o) //要点①
		// ...
		if invocation == nil {
			continue
		}

		changed, err := a.callAttrMutatingHook(ctx, hook, invocation, ...) //要点②
		// ...

		if err == nil {
			continue
		}

		if callErr, ok := err.(*webhookutil.ErrCallingWebhook); ok { //要点③
			if ignoreClientCallFailures {
				// ...
				continue
			}
			return apierrors.NewInternalError(err) //要点④
		}

		if rejectionErr, ok := err.(*webhookutil.ErrWebhookRejection); ok {
			return rejectionErr.Status //要点⑤
		}
		return err
	}
	// ...
	return nil
}
```

在上述代码中，MutatingAdmissionWebhook 执行准入控制器的逻辑为：

* 首先遍历所有的 WebhookAccessor，首先通过要点①处的 ShouldCallHook 函数检查当前资源对象是否匹配 Webhook 定义的匹配规则，如果不匹配，则跳过该 hook 的执行。如果匹配，则通过要点②处的 callAttrMutatingHook 函数发起远程 Webhook 调用，底层通过客户端向目标服务发起 Post 请求，请求体为 AdmissionReview 对象。
* 然后根据请求返回的结果，如果调用正确完成且被许可，则应用返回的 Patch 对 attr 的 VersionedObject 进行变更，之后继续执行下一个 hook，否则检查失败类型。 当失败类型为要点③处的  ErrCallingWebhook 时，通过要点④检查该 Webhook 设置的失败策略是否为 Ignore，如果是， 则跳过该 hook ，继续执行下一个 hook ，否则返回调用失败错误；当失败类型为 ErrWebhookRejection 时，通过要点⑤返回 Status 拒绝原因。

ShouldCallHook 函数完成请求和 Webhook 规则的匹配，代码示例如下。

```go
// 代码: staging\src\k8s.io\apiserver\pkg\admission\plugin\webhook\generic\webhook.go#L156-L245
// ShouldCallHook returns invocation details if the webhook should be called, nil if the webhook should not be called,
// or an error if an error was encountered during evaluation.
func (a *Webhook) ShouldCallHook(ctx context.Context, h webhook.WebhookAccessor, attr admission.Attributes, o admission.ObjectInterfaces, v VersionedAttributeAccessor) (*WebhookInvocation, *apierrors.StatusError) {
	matches, matchNsErr := a.namespaceMatcher.MatchNamespaceSelector(h, attr) // 要点①
	// Should not return an error here for webhooks which do not apply to the request, even if err is an unexpected scenario.
	if !matches && matchNsErr == nil {
		return nil, nil
	}

	// Should not return an error here for webhooks which do not apply to the request, even if err is an unexpected scenario.
	matches, matchObjErr := a.objectMatcher.MatchObjectSelector(h, attr) // 要点②
	if !matches && matchObjErr == nil {
		return nil, nil
	}

	var invocation *WebhookInvocation
	for _, r := range h.GetRules() {
		m := rules.Matcher{Rule: r, Attr: attr}
		if m.Matches() { // 要点③
			invocation = &WebhookInvocation{
				Webhook:     h,
				Resource:    attr.GetResource(),
				Subresource: attr.GetSubresource(),
				Kind:        attr.GetKind(),
			}
			break
		}
	}
	if invocation == nil && h.GetMatchPolicy() != nil && *h.GetMatchPolicy() == v1.Equivalent {
	...
	OuterLoop:
		for _, r := range h.GetRules() {
			// see if the rule matches any of the equivalent resources
			for _, equivalent := range equivalents {
				...
				attrWithOverride.resource = equivalent // 要点④
				m := rules.Matcher{Rule: r, Attr: attrWithOverride}
				if m.Matches() { // 要点⑤
					...
					invocation = &WebhookInvocation{
						Webhook:     h,
						Resource:    equivalent,
						Subresource: attr.GetSubresource(),
						Kind:        kind,
					}
					break OuterLoop
				}
			}
		}
	}

	if invocation == nil {
		return nil, nil
	}
	if matchNsErr != nil {
		return nil, matchNsErr
	}
	if matchObjErr != nil {
		return nil, matchObjErr
	}
	...
	return invocation, nil
}
```

* 首先通过要点① `namespaceMatcher.MatchNamespaceSelector` 检查命名空间是否匹配；
* 然后，通过要点② `objectMatcher.MatchObjectSelector` 检查资源对象标签是否匹配；
* 最后，通过要点③检查资源 group/version/resource 是否匹配。
* 如果使用了 Equivalent 匹配策略【要点④】，则匹配与其等价的资源类型【要点⑤】。

### ValidatingAdmissionWebhook

ValidatingAdmissionWebhook 与 MutatingAdmissionWebhook 类似，是一种插件式验证准入控制器，能够在不改变 kube-apiserver 源码的情况下，扩展准入控制器的功能，允许用户定制 Webhook Admission Server 服务，基于 ValidatingWebhookConfiguration 动态配置拦截规则，实现自定义的准入控制处理逻辑，对传入的匹配规则的资源对象在真正持久化前通过 Webhook 执行验证操作。

ValidatingAdmissionWebhook 准入控制器的运行原理如图所示。

![image-20251209162106385](http://images.liangning7.cn/typora/202512091621722.png)

ValidatingAdmissionWebhook 准入控制器根据逻辑可以划分为两个主要部分。

* ValidatingAdmissionWebhook 配置发现：即上图左侧所示，ValidatingAdmissionWebhook 准入控制器在初始化阶段，会注册事件监听，通过 Informer 监听集群中的 ValidatingWebhookConfiguration 资源对象事件，基于事件触发 ValidatingWebhookConfigurationManager 同步更新内部维护的 WebhookAccessor 列表，使其始终处于最新状态。WebhookAccessor 列表保存了 ValidatingAdmissionWebhook 准入控制器执行 Validate 处理时依赖的必要配置，是 ValidatingAdmissionWebhook 配置发现阶段的产物，也是 Validating 准入控制执行阶段消费的资料。
* ValidatingAdmissionWebhook 准入控制执行：如上图右侧所示，在初始化阶段，ValidatingAdmissionWebhook 的 Handler 已经注册到 kube-apiserver 的处理链条。当收到客户端发起的资源请求时，kube-apiserver 按照顺序执行处理链条上的 Handler（如认证、鉴权等），最终触发 ValidatingAdmissionWebhook 准入控制器执行。在执行 Validate 准入控制时，ValidatingAdmissionWebhook 准入控制器会通过 Webhooks 函数调用读取当前系统配置的 WebhookAccessor 列表，结合传入的资源对象属性，判断是否需要发起 Webhook 调用。如果需要，则构建 AdmissionReview 请求对象，向远端 Webhook Admission Server 发起 HTTP 调用，根据返回结果进行后续的操作。

与 MutatingAdmissionWebhook 类似，ValidatingAdmissionWebhook 判断是否需要对资源对象执行 Validate 准入控制依赖规则配置（ValidatingWebhookConfiguration）和资源对象属性 （`admission.Attributes`）。

一个典型的 ValidatingWebhookConfiguration 配置示例如下。

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: validating-webhook-example #  配置的名称，用于唯一标识这个 webhook 配置
webhooks:
  - name: validate.example.com # Webhook 的名称，必须是唯一的
    clientConfig: # 配置 webhook 服务的连接信息
      service:
        name: validating-webhook-service # Webhook 服务的名称
        namespace: default # Webhook 服务所在的命名空间
        path: "/validate" # Webhook 服务的路径
      # caBundle 这里应该是 Base64 编码的 CA 证书，用于校验 webhook 服务的 TLS 证书
      caBundle: "YOUR_CA_BUNDLE"
    rules: # 定义 webhook 应用的规则
      - operations: ["CREATE", "UPDATE"] # Webhook 触发的操作类型，如 CREATE、UPDATE。还有一个类型：DELETE
        apiGroups: [""] # 目标资源所在的 API 组
        apiVersions: ["v1"] # 目标资源的 API 版本
        resources: ["pods"] # 目标资源类型，如 pods
    failurePolicy: Fail #  Webhook 服务失败时的策略，Fail 或 Ignore
    matchPolicy: Equivalent # 资源匹配策略，Exact 或 Equivalent
    namespaceSelector: #  基于命名空间标签选择应用于哪些命名空间
      matchLabels:
        webhook: enabled 
    objectSelector: # 基于资源标签选择应用于哪些资源
      matchLabels:
        apply-webhook: "true"
    sideEffects: None # Webhook 的副作用，None、Some 或 Unknown
    timeoutSeconds: 5 # Webhook 请求的超时时间（秒）
    admissionReviewVersions: ["v1", "v1beta1"] # 支持的 AdmissionReview 版本
```

ValidatingAdmissionWebhook 执行匹配使用的 Attributes 接口和匹配方式与 MutatingAdmissionWebhook 使用的相同，故不再赘述。

ValidatingAdmissionWebhook 插件的初始化及验证准入控制 Validate 函数如下。

```go
// 代码: staging\src\k8s.io\apiserver\pkg\admission\plugin\webhook\validating\plugin.go#L53-L67
// NewValidatingAdmissionWebhook returns a generic admission webhook plugin.
func NewValidatingAdmissionWebhook(configFile io.Reader) (*Plugin, error) {
	handler := admission.NewHandler(admission.Connect, admission.Create, admission.Delete, admission.Update)
	p := &Plugin{}
	var err error
	p.Webhook, err = generic.NewWebhook(handler, configFile, configuration.NewValidatingWebhookConfigurationManager, newValidatingDispatcher(p))
	if err != nil {
		return nil, err
	}
	return p, nil
}

// Validate makes an admission decision based on the request attributes.
func (a *Plugin) Validate(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces) error {
	return a.Webhook.Dispatch(ctx, attr, o)
}
```

ValidatingAdmissionWebhook 底层同样基于 `generic.Webhook` 实现，其准入控制器 Validate 函数直接调用了 `generic.Webhook` 的 Dispatch 函数，最终调用到 ValidatingAdmissionWebhook 插件自身的 Dispatch 函数。

```go
// 代码: staging\src\k8s.io\apiserver\pkg\admission\plugin\webhook\validating\dispatcher.go#L88-L246
func (d *validatingDispatcher) Dispatch(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces, hooks []webhook.WebhookAccessor) error {
	...
	for _, hook := range hooks {
		invocation, statusError := d.plugin.ShouldCallHook(ctx, hook, attr, o, versionedAttrAccessor) // 要点①
		if statusError != nil {
			return statusError
		}
		if invocation == nil {
			continue
		}

		relevantHooks = append(relevantHooks, invocation) // 要点②
		...
	}

	if len(relevantHooks) == 0 {
		// no matching hooks
		return nil
	}

	// Check if the request has already timed out before spawning remote calls
	select {
	case <-ctx.Done():
		// parent context is canceled or timed out, no point in continuing
		return apierrors.NewTimeoutError("request did not complete within requested timeout", 0)
	default:
	}

	wg := sync.WaitGroup{}
	errCh := make(chan error, 2*len(relevantHooks)) // double the length to handle extra errors for panics in the gofunc
	wg.Add(len(relevantHooks)) // 要点③
	for i := range relevantHooks {
		go func(invocation *generic.WebhookInvocation, idx int) {
			...
			err := d.callHook(ctx, hook, invocation, versionedAttr) // 要点④
			...
			if callErr, ok := err.(*webhookutil.ErrCallingWebhook); ok {
				if ignoreClientCallFailures {
					...
					return
				}
				...
				errCh <- apierrors.NewInternalError(err) // 要点⑤
				return
			}

			if rejectionErr, ok := err.(*webhookutil.ErrWebhookRejection); ok {
				err = rejectionErr.Status
			}
			klog.Warningf("rejected by webhook %q: %#v", hook.Name, err)
			errCh <- err // 要点⑥
		}(relevantHooks[i], i)
	}
	wg.Wait() // 要点⑦
	close(errCh)

	var errs []error
	for e := range errCh { // 要点⑧
		errs = append(errs, e)
	}
	if len(errs) == 0 {
		return nil
	}
	if len(errs) > 1 {
		for i := 1; i < len(errs); i++ {
			// TODO: merge status errors; until then, just return the first one.
			utilruntime.HandleError(errs[i])
		}
	}
	return errs[0]
}
```

与 MutatingAdmissionWebhook 的顺序执行不同，ValidatingAdmissionWebhook 在执行阶段是并行执行的。这是因为在验证阶段不会对资源对象进行任何修改，最终的验证结果与执行顺序无关。

1. 首先，遍历 WebhookAccessor 列表，通过要点①的 ShouldCallHook 函数检查当前资源对象是否匹配 Webhook 定义的匹配规则，如果不匹配，则跳过该 hook 的执行。如果匹配，则通过要点②追加到待执行 hook 列表。
2. 然后，通过要点③ WaitGroup 发起针对每个 hook 的并行调用，检查每个 hook 的执行结果，检查失败类型。当类型为 ErrCallingWebhook 且 Webhook 设置的失败策略为 Fail 或返回 的失败类型为 ErrWebhookRejection 时，将 err 记录到 errCh。【要点④⑤⑥】
3. 最后，根据要点⑧处的 errCh 内容是否为空， 判断最终的准入控制器的执行结果，决定拒绝或允许该资源请求。

ShouldCallHook 的匹配方式与 MutatingAdmissionWebhook 实现一致，故不再赘述。

