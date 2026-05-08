+++
title = "K8s Generic Server WebService"
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

series_order = 9

+++

# K8s Generic Server WebService

WebService 及其 Router 生成过程，其主要代码位于：`staging\src\k8s.io\apiserver\pkg\endpoints\installer.go#L285-L1128`

下面用 Pod 资源作为案例，详细讲解从准备到生成路由的完整过程。

## 案例背景：注册 Pod 资源

假设我们要注册：

* 主资源：`pods`
* 子资源：`pods/status`

## 阶段0. 函数调用前的准备

### 调用代码

> `staging\src\k8s.io\apiserver\pkg\endpoints\installer.go#L208`
>
> ```go
> apiResource, resourceInfo, err := a.registerResourceHandlers(path, a.group.Storage[path], ws)
> ```

### 需要准备的入参

1. `path = "pods"`：资源路径名称

2. `storage = podStorage`【实现了多个接口】

   ```go
   // 代码: pkg\registry\core\pod\storage\storage.go#L52-L71
   // PodStorage includes storage for pods and all sub resources
   type PodStorage struct {
   	Pod                 *REST
   	Binding             *BindingREST
   	LegacyBinding       *LegacyBindingREST
   	Eviction            *EvictionREST
   	Status              *StatusREST
   	EphemeralContainers *EphemeralContainersREST
   	Resize              *ResizeREST
   	Log                 *podrest.LogREST
   	Proxy               *podrest.ProxyREST
   	Exec                *podrest.ExecREST
   	Attach              *podrest.AttachREST
   	PortForward         *podrest.PortForwardREST
   }
   
   // REST implements a RESTStorage for pods
   type REST struct {
   	*genericregistry.Store
   	proxyTransport http.RoundTripper
   }
   
   // 实现的接口：
   // - rest.Scoper          (NamespaceScoped() bool)
   // - rest.Getter          (Get(...) runtime.Object)
   // - rest.Lister          (List(...) runtime.Object)
   // - rest.Creater         (Create(...) runtime.Object)
   // - rest.Updater         (Update(...) runtime.Object)
   // - rest.GracefulDeleter (Delete(...) runtime.Object)
   // - rest.Watcher         (Watch(...) watch.Interface)
   // - rest.Patcher         (Patch(...) runtime.Object)
   // - rest.CollectionDeleter
   // - rest.TableConvertor
   // - rest.StorageVersionProvider
   ```

3. `ws = *restful.WebServic` ，即通过 `ws := a.NewWebService()` 创建的 `go-restful WebService` 实例，路径为 `/api/v1`，

   ```go
   // 代码: staging\src\k8s.io\apiserver\pkg\endpoints\installer.go#L193-#L220
   // Install handlers for API resources.
   func (a *APIInstaller) Install() ([]metav1.APIResource, []*storageversion.ResourceInfo, *restful.WebService, []error) {
       ...
   	ws := a.newWebService() // #L197
   	...
   	for _, path := range paths {
   		apiResource, resourceInfo, err := a.registerResourceHandlers(path, a.group.Storage[path], ws) // #L208
   	...
   	}
   	return apiResources, resourceInfos, ws, errors
   }
   ```

4. 由于该函数是 `APIInstaller` 的方法，该 APIInstaller 包含的上下文如下：

   ```go
   a.group = &APIGroupVersion{
       GroupVersion: schema.GroupVersion{Group: "", Version: "v1"},
       Serializer:   scheme.Codecs,
       Creater:      scheme.Scheme,
       Typer:       scheme.Scheme,
       Convertor:   scheme.Scheme,
       Defaulter:   scheme.Scheme,
       Admit:       admissionChain,
       Namer:       runtime.Namer,
       Storage: map[string]rest.Storage{
           "pods":        podStorage.Pod,
           "pods/status": podStorage.Status,
           "pods/log":    podStorage.Log,
           // ...
       },
   }
   ```


## 阶段1. 初始化与准备

> **Lines 286-332**

函数签名如下：

```go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error)
```

初始化与准备的代码如下：

```go
	admit := a.group.Admit

	optionsExternalVersion := a.group.GroupVersion
	if a.group.OptionsExternalVersion != nil {
		optionsExternalVersion = *a.group.OptionsExternalVersion
	}

	resource, subresource, err := splitSubresource(path)
	if err != nil {
		return nil, nil, err
	}

	group, version := a.group.GroupVersion.Group, a.group.GroupVersion.Version

	fqKindToRegister, err := GetResourceKind(a.group.GroupVersion, storage, a.group.Typer)
	if err != nil {
		return nil, nil, err
	}

	versionedPtr, err := a.group.Creater.New(fqKindToRegister)
	if err != nil {
		return nil, nil, err
	}
	defaultVersionedObject := indirectArbitraryPointer(versionedPtr)
	kind := fqKindToRegister.Kind
	isSubresource := len(subresource) > 0

	// If there is a subresource, namespace scoping is defined by the parent resource
	var namespaceScoped bool
	if isSubresource {
		parentStorage, ok := a.group.Storage[resource]
		if !ok {
			return nil, nil, fmt.Errorf("missing parent storage: %q", resource)
		}
		scoper, ok := parentStorage.(rest.Scoper)
		if !ok {
			return nil, nil, fmt.Errorf("%q must implement scoper", resource)
		}
		namespaceScoped = scoper.NamespaceScoped()

	} else {
		scoper, ok := storage.(rest.Scoper)
		if !ok {
			return nil, nil, fmt.Errorf("%q must implement scoper", resource)
		}
		namespaceScoped = scoper.NamespaceScoped()
	}
```

1. 获取准入控制器

   ```go
   admit := a.group.Admit
   // 输出: admit = admissionChain (包含 PodSecurityPolicy, ResourceQuota 等)
   ```

   这里的准入控制器在后续会进行详细讲解

2. 确定 Options 对象的版本

   ```go
   optionsExternalVersion := a.group.GroupVersion
   if a.group.OptionsExternalVersion != nil {
       optionsExternalVersion = *a.group.OptionsExternalVersion
   }
   ```

   确定 `ListOptions`、`CreateOptions` 等选项对象使用的 API 版本，默认使用当前组版本。

3. 分离主资源和子资源

   ```go
   resource, subresource, err := splitSubresource(path)
   if err != nil {
       return nil, nil, err
   }
   ```

   * 将路径拆分为主资源和子资源
   * 例如：`"pods/status"` → resource=`"pods"`, subresource=`"status"`
   * 例如：`"pods"` → resource=`"pods"`, subresource=`""`

4. 提取 Group 和 Version

   ```go
   group, version := a.group.GroupVersion.Group, a.group.GroupVersion.Version
   ```

   提取当前 API 的组名和版本号，后续用于路径构建和指标记录。

5. 获取要注册的 GVK

   ```go
   fqKindToRegister, err := GetResourceKind(a.group.GroupVersion, storage, a.group.Typer)
   if err != nil {
       return nil, nil, err
   }
   ```

   通过 Storage 对象获取该资源的完整 Kind 信息（Group+Version+Kind）

   输出：`fqKindToRegister = {Group: "", Version: "v1", Kind: "Pod"}`

6. 创建默认版本化对象

   ```go
   versionedPtr, err := a.group.Creater.New(fqKindToRegister)
   if err != nil {
       return nil, nil, err
   }
   defaultVersionedObject := indirectArbitraryPointer(versionedPtr)
   kind := fqKindToRegister.Kind
   isSubresource := len(subresource) > 0
   ```

   * 输出：`versionedPtr = &v1.Pod{}`
   * 创建一个该资源类型的实例，用于 Swagger 文档生成
   * 提取 Kind 名称
   * 判断是否为子资源

7. 确定命名空间作用域

   ```go
   // If there is a subresource, namespace scoping is defined by the parent resource
   var namespaceScoped bool
   if isSubresource {
       parentStorage, ok := a.group.Storage[resource]
       if !ok {
           return nil, nil, fmt.Errorf("missing parent storage: %q", resource)
       }
       scoper, ok := parentStorage.(rest.Scoper)
       if !ok {
           return nil, nil, fmt.Errorf("%q must implement scoper", resource)
       }
       namespaceScoped = scoper.NamespaceScoped()
   
   } else {
       scoper, ok := storage.(rest.Scoper)
       if !ok {
           return nil, nil, fmt.Errorf("%q must implement scoper", resource)
       }
       namespaceScoped = scoper.NamespaceScoped()
   }
   ```

   * **主资源**：直接查询当前 Storage 的 `NamespaceScoped()` 方法
   * **子资源**：必须从父资源的 Storage 获取作用域信息
     * 例如：`pods/status` 的命名空间作用域由 `pods` 决定

支持，输出如下：

* `resource` = `"pods"`
* `kind` = `"Pod"`
* `namespaceScoped` = `true`
* `defaultVersionedObject = v1.Pod{...}`

## 阶段2. 检测 Storage 能力

> **Lines 334-351**

输入：`storage` = `podStorage.Pod`

处理过程，类型断言：

```go
creater, isCreater := storage.(rest.Creater)
// isCreater = true, creater = podStorage.Pod

lister, isLister := storage.(rest.Lister)
// isLister = true, lister = podStorage.Pod

getter, isGetter := storage.(rest.Getter)
// isGetter = true, getter = podStorage.Pod

updater, isUpdater := storage.(rest.Updater)
// isUpdater = true, updater = podStorage.Pod

patcher, isPatcher := storage.(rest.Patcher)
// isPatcher = true, patcher = podStorage.Pod

gracefulDeleter, isGracefulDeleter := storage.(rest.GracefulDeleter)
// isGracefulDeleter = true, gracefulDeleter = podStorage.Pod

collectionDeleter, isCollectionDeleter := storage.(rest.CollectionDeleter)
// isCollectionDeleter = true, collectionDeleter = podStorage.Pod

watcher, isWatcher := storage.(rest.Watcher)
// isWatcher = true, watcher = podStorage.Pod

connecter, isConnecter := storage.(rest.Connecter)
// isConnecter = false (Pod 主资源不支持 CONNECT)

tableProvider, isTableProvider := storage.(rest.TableConvertor) //#L480
// isTableProvider = true, tableProvider = podStorage.Pod
```

输出（能力标志）

```go
{
    isCreater: true,
    isLister: true,
    isGetter: true,
    isUpdater: true,
    isPatcher: true,
    isGracefulDeleter: true,
    isCollectionDeleter: true,
    isWatcher: true,
    isConnecter: false,
    isTableProvider: true,
}
```

### Storage 类型断言的底层机制

这里来介绍 Kubernetes API 框架的一个核心设计模式，基于 Go 的接口组合（Interface Composition）和类型断言（Type Assertion）。

1. 设计理念：能力导向的接口设计

   Kubernetes 采用了小接口组合的设计模式，而不是一个大而全的接口：

   ```go
   // 基础接口 - 所有 Storage 必须实现
   type Storage interface {
       New() runtime.Object
       Destroy()
   }
   
   // 能力接口 - 按需实现
   type Creater interface {
       New() runtime.Object
       Create(ctx, obj, validation, options) (runtime.Object, error)
   }
   
   type Lister interface {
       NewList() runtime.Object
       List(ctx, options) (runtime.Object, error)
       TableConvertor
   }
   
   type Getter interface {
       Get(ctx, name, options) (runtime.Object, error)
   }
   
   type Updater interface {
       New() runtime.Object
       Update(ctx, name, objInfo, ...) (runtime.Object, bool, error)
   }
   
   type GracefulDeleter interface {
       Delete(ctx, name, validation, options) (runtime.Object, bool, error)
   }
   
   // ... 还有 Watcher, Patcher, Connecter 等
   ```

2. Pod Storage 的实现层次

   ```go
   // 第一层：Pod 的 REST 结构
   type REST struct {
       *genericregistry.Store  // 嵌入 Store，继承所有方法
       proxyTransport http.RoundTripper
   }
   ```

   ```go
   // 第二层：genericregistry.Store 实现了多个接口
   type Store struct {
       NewFunc         func() runtime.Object
       NewListFunc     func() runtime.Object
       CreateStrategy  rest.RESTCreateStrategy
       UpdateStrategy  rest.RESTUpdateStrategy
       DeleteStrategy  rest.RESTDeleteStrategy
       Storage         DryRunnableStorage  // 底层 etcd 存储
       // ...
   }
   
   // Store 显式声明实现的接口
   var _ rest.StandardStorage = &Store{}
   var _ rest.StorageWithReadiness = &Store{}
   var _ rest.TableConvertor = &Store{}
   
   // StandardStorage 是多个接口的组合
   type StandardStorage interface {
       Getter
       Lister
       CreaterUpdater
       GracefulDeleter
       CollectionDeleter
       Watcher
       Destroy()
   }
   ```

3. 类型断言的工作原理

   ```go
   // installer.go 中的类型断言
   creater, isCreater := storage.(rest.Creater)
   lister, isLister := storage.(rest.Lister)
   getter, isGetter := storage.(rest.Getter)
   // ...
   
   // 实际发生的事情：
   // 1. storage 的动态类型是 *pod.REST
   // 2. *pod.REST 嵌入了 *genericregistry.Store
   // 3. *genericregistry.Store 实现了这些接口的方法
   // 4. Go 运行时检查 *pod.REST 是否实现了 rest.Creater 接口
   // 5. 如果实现了，isCreater = true，creater 指向该对象
   ```

由于 Pod 需要 `*genericregistry.Store` 里的所有方法，即可以采用**嵌入** `Store` ，来继承所有方法。而对于 `Pod.Status` 资源，不需要那么多方法，即可以采用**字段**方式，然后手动委托方法，`StatusREST` 只实现了 4 个方法，因此只支持有限的操作：例如：

```go
// 1. Storage 接口（基础）
func (r *StatusREST) New() runtime.Object {
    return &api.Pod{}
}

func (r *StatusREST) Destroy() {
    // Given that underlying store is shared with REST,
    // we don't destroy it here explicitly.
}

// 2. Getter 接口（支持 GET）
func (r *StatusREST) Get(ctx context.Context, name string, options *metav1.GetOptions) (runtime.Object, error) {
    return r.store.Get(ctx, name, options)  // 委托给 store
}

// 3. Updater 接口（支持 PUT/PATCH）
func (r *StatusREST) Update(ctx context.Context, name string, objInfo rest.UpdatedObjectInfo, createValidation rest.ValidateObjectFunc, updateValidation rest.ValidateObjectUpdateFunc, forceAllowCreate bool, options *metav1.UpdateOptions) (runtime.Object, bool, error) {
    // 注意：forceAllowCreate 被强制设为 false
    // 子资源不允许 create on update
    return r.store.Update(ctx, name, objInfo, createValidation, updateValidation, false, options)
}

// 4. TableConvertor 接口（支持表格输出）
func (r *StatusREST) ConvertToTable(ctx context.Context, object runtime.Object, tableOptions runtime.Object) (*metav1.Table, error) {
    return r.store.ConvertToTable(ctx, object, tableOptions)
}

// 5. ResetFieldsStrategy 接口
func (r *StatusREST) GetResetFields() map[fieldpath.APIVersion]*fieldpath.Set {
    return r.store.GetResetFields()
}
```

也有不“继承”`*genericregistry.Store`的资源，直接自己实现的，例如 `ComponentStatus`，只读资源，不存储在 etcd，实时查询组件健康状态，其 REST 实现如下：

```go
type REST struct {
    GetServersToValidate func() map[string]Server  // 获取要检查的服务器列表
    rest.TableConvertor
}

// 只实现 Getter 和 Lister 接口
func (rs *REST) Get(ctx context.Context, name string, options *metav1.GetOptions) (runtime.Object, error) {
    servers := rs.GetServersToValidate()
    
    if server, ok := servers[name]; !ok {
        return nil, apierrors.NewNotFound(api.Resource("componentstatus"), name)
    } else {
        // 实时探测组件健康状态
        return rs.getComponentStatus(name, server), nil
    }
}

func (rs *REST) List(ctx context.Context, options *metainternalversion.ListOptions) (runtime.Object, error) {
    servers := rs.GetServersToValidate()
    
    // 并发探测所有组件
    wait := sync.WaitGroup{}
    wait.Add(len(servers))
    statuses := make(chan api.ComponentStatus, len(servers))
    for k, v := range servers {
        go func(name string, server Server) {
            defer wait.Done()
            status := rs.getComponentStatus(name, server)
            statuses <- *status
        }(k, v)
    }
    wait.Wait()
    
    // 返回所有组件状态
    return &api.ComponentStatusList{Items: reply}, nil
}

// 不支持 Create, Update, Delete, Watch
```

由于实现了 `rest.Getter` 、`rest.Lister` 接口，而其他接口没有实现：

```go
storage := componentStatusStorage  // *REST

isGetter := storage.(rest.Getter)              // ✓ true
isLister := storage.(rest.Lister)              // ✓ true
isCreater := storage.(rest.Creater)            // ✗ false
isUpdater := storage.(rest.Updater)            // ✗ false
isDeleter := storage.(rest.GracefulDeleter)    // ✗ false
isWatcher := storage.(rest.Watcher)            // ✗ false
```

## 阶段3. 创建版本化对象

> **Lines 357-463**

通过 List 操作来看创建版本化对象的目的：

```go
// 创建版本化的 List 对象
versionedListPtr, err := a.group.Creater.New(a.group.GroupVersion.WithKind(listGVKs[0].Kind)))
versionedList = indirectArbitraryPointer(versionedListPtr)
// 结果：v1.PodList{}

// 创建版本化的 ListOptions
versionedListOptions, err := a.group.Creater.New(optionsExternalVersion.WithKind("ListOptions"))
// 结果：&metav1.ListOptions{}

// 在路由注册时使用
route := ws.GET(action.Path).To(handler).
    Doc("list objects of kind Pod"). // 以 Pod 为例，与源码有差异
    Returns(http.StatusOK, "OK", versionedList).  // 告诉 OpenAPI：返回 PodList
    Writes(versionedList)                          // 用于生成响应 schema

// 添加查询参数到 OpenAPI 文档
if err := AddObjectParams(ws, route, versionedListOptions); err != nil {
    return nil, nil, err
}
```

1. OpenAPI/Swagger 文档生成，这是最主要的用途。这些版本化对象用于生成 API 文档，告诉客户端：
   * 每个 API 端点接受什么参数
   * 返回什么类型的对象
   * 参数的数据类型和描述
2. 这些版本化对象告诉 go-restful 框架：
   * 请求体应该是什么类型（`Reads()`）
   * 响应体应该是什么类型（`Writes()`, R`eturns()`）
3. Kubernetes 支持多个 API 版本（v1, v1beta1, v1alpha1），这些版本化对象确保使用正确的版本。

## 阶段4. Actions 数组的构建

> **Lines 465-595**

输入：

* `namespaceScoped` = `true`
* 各种能力标志 (`isLister`, `isCreater`, 等)

Actions 结构体定义

```go
type action struct {
    Verb          string                    // HTTP 方法：GET, POST, PUT, DELETE, etc.
    Path          string                    // URL 路径
    Params        []*restful.Parameter      // 路径参数列表
    Namer         handlers.ScopeNamer       // 命名器
    AllNamespaces bool                      // 是否跨命名空间
}
```

### Actions 的最后准备

```go
	// #L465-L496
	allowWatchList := isWatcher && isLister // watching on lists is allowed only for kinds that support both watch and list.
	nameParam := ws.PathParameter("name", "name of the "+kind).DataType("string")
	pathParam := ws.PathParameter("path", "path to the resource").DataType("string")

	params := []*restful.Parameter{}
	actions := []action{}

	var resourceKind string
	kindProvider, ok := storage.(rest.KindProvider)
	if ok {
		resourceKind = kindProvider.Kind()
	} else {
		resourceKind = kind
	}

	tableProvider, isTableProvider := storage.(rest.TableConvertor)
	if isLister && !isTableProvider {
		// All listers must implement TableProvider
		return nil, nil, fmt.Errorf("%q must implement TableConvertor", resource)
	}

	var apiResource metav1.APIResource
	if utilfeature.DefaultFeatureGate.Enabled(features.StorageVersionHash) &&
		isStorageVersionProvider &&
		storageVersionProvider.StorageVersion() != nil {
		versioner := storageVersionProvider.StorageVersion()
		gvk, err := getStorageVersionKind(versioner, storage, a.group.Typer)
		if err != nil {
			return nil, nil, err
		}
		apiResource.StorageVersionHash = discovery.StorageVersionHash(gvk.Group, gvk.Version, gvk.Kind)
	}
```

1. Watch 能力判断：

   ```go
   allowWatchList := isWatcher && isLister
   ```

   * 判断是否同时支持 **Watch** 和 **List**
   * 只有两者都支持，才允许对列表进行 watch 操作

2. 创建通用路径参数

   ```go
   nameParam := ws.PathParameter("name", "name of the "+kind).DataType("string")
   pathParam := ws.PathParameter("path", "path to the resource").DataType("string")
   ```

   * `nameParam`：用于具体资源的路径，如 `/pods/{name}`
   * `pathParam`：用于子路径访问，如 `/pods/{name}/log/{path:*}`

3. 核心数组初始化

   ```go
   params := []*restful.Parameter{}
   actions := []action{}
   ```

   * `actions`：即将构建的 action 数组，后续会根据资源能力向这个数组添加各种 HTTP 操作

4. 确定资源 Kind 名称

   ```go
   var resourceKind string
   kindProvider, ok := storage.(rest.KindProvider)
   if ok {
       resourceKind = kindProvider.Kind()
   } else {
       resourceKind = kind
   }
   ```

   * 优先使用 Storage 自定义的 Kind（通过 `KindProvider` 接口）
   * 否则使用默认的 Kind
   * 某些子资源可能需要返回父资源的 Kind

5. TableProvider 验证

   ```go
   tableProvider, isTableProvider := storage.(rest.TableConvertor)
   if isLister && !isTableProvider {
       // All listers must implement TableProvider
       return nil, nil, fmt.Errorf("%q must implement TableConvertor", resource)
   }
   ```

   * 所有支持 LIST 的资源**必须**实现 `TableConvertor` 接口
   * 用于将对象转换为表格格式（`kubectl get` 的表格输出）
   * 不满足直接报错，不允许注册

6. 初始化 APIResource 并计算存储版本哈希

   ```go
   var apiResource metav1.APIResource
   if utilfeature.DefaultFeatureGate.Enabled(features.StorageVersionHash) &&
   isStorageVersionProvider &&
   storageVersionProvider.StorageVersion() != nil {
       versioner := storageVersionProvider.StorageVersion()
       gvk, err := getStorageVersionKind(versioner, storage, a.group.Typer)
       if err != nil {
           return nil, nil, err
       }
       apiResource.StorageVersionHash = discovery.StorageVersionHash(gvk.Group, gvk.Version, gvk.Kind)
   }
   ```

   * 创建 `APIResource` 对象（函数的返回值之一）；
   * 如果启用 `StorageVersionHash` 特性，计算并设置存储版本哈希；
   * 用于 API Discovery，客户端可以通过哈希判断存储版本是否变化。

### Actions 的构建

根据资源的作用域（Cluster-scoped 或 Namespace-scoped）构建 **actions 数组**

由于我们以 Pod 为例，即进入到 `default` 分支：

1. 参数和路径的准备工作：

   ```go
   // L543-L560
   namespaceParamName := "namespaces"
   namespaceParam := ws.PathParameter("namespace", "object name and auth scope, such as for teams and projects").DataType("string")
   namespacedPath := namespaceParamName + "/{namespace}/" + resource
   // 结果：namespacedPath = "namespaces/{namespace}/pods"
   
   namespaceParams := []*restful.Parameter{namespaceParam}
   // 结果：namespaceParams = [namespace参数]
   
   resourcePath := namespacedPath
   // 结果：resourcePath = "namespaces/{namespace}/pods"
   
   resourceParams := namespaceParams
   // 结果：resourceParams = [namespace参数]
   
   itemPath := namespacedPath + "/{name}"
   // 结果：itemPath = "namespaces/{namespace}/pods/{name}"
   
   nameParams := append(namespaceParams, nameParam)
   // 结果：nameParams = [namespace参数, name参数]
   
   proxyParams := append(nameParams, pathParam)
   // 结果：proxyParams = [namespace参数, name参数, path参数]
   ```

   下面看实际例子：

   | **场景**        | **实际 URL 路径**                                          | **namespace**   | **name**                         | **path**                                               |
   | --------------- | ---------------------------------------------------------- | --------------- | -------------------------------- | ------------------------------------------------------ |
   | **List**        | `/api/v1/namespaces/default/pods`                          | `ns: "default"` | *(无)*                           | *(无)*                                                 |
   | **Get**         | `/api/v1/namespaces/default/pods/my-pod`                   | `ns: "default"` | `ns: "default"` `name: "my-pod"` | *(无)*                                                 |
   | **Subresource** | `/api/v1/namespaces/default/pods/my-pod/status`            | `ns: "default"` | `ns: "default"` `name: "my-pod"` | *(无)*  <small>*注1*</small>                           |
   | **Proxy**       | `/api/v1/namespaces/default/pods/my-pod/proxy/api/metrics` | `ns: "default"` | `ns: "default"` `name: "my-pod"` | `ns: "default"` `name: "my-pod"` `path: "api/metrics"` |

2. 子资源特殊处理

   ```go
   // 第 554-559 行
   itemPathSuffix := ""
   if isSubresource {
       itemPathSuffix = "/" + subresource
       itemPath = itemPath + itemPathSuffix
       resourcePath = itemPath
       resourceParams = nameParams
   }
   ```

   ```go
   // 主资源
   isSubresource = false
   subresource = ""
   
   // 路径不变
   resourcePath = "namespaces/{namespace}/pods"
   resourceParams = [namespace参数]
   itemPath = "namespaces/{namespace}/pods/{name}"
   
   
   // 子资源
   isSubresource = true
   subresource = "status"
   
   // 路径变化
   itemPathSuffix = "/status"
   itemPath = "namespaces/{namespace}/pods/{name}/status"
   resourcePath = "namespaces/{namespace}/pods/{name}/status"  // ← 注意：变成了 itemPath
   resourceParams = [namespace参数, name参数]  // ← 注意：nameParams := append(namespaceParams, nameParam)，这里比主资源多了一个 name 参数
   ```

   * 主资源：resourcePath 是集合路径（不含 name），itemPath 是单个资源路径（含 name）

   * 子资源：resourcePath 和 itemPath 都是单个资源路径（都含 name），因为子资源必须依附于父资源

3. 最后构建 Actions 数组即可

   ```go
   actions = appendIf(actions, action{request.MethodList, resourcePath, resourceParams, namer, false}, isLister)
   actions = appendIf(actions, action{request.MethodPost, resourcePath, resourceParams, namer, false}, isCreater)
   actions = appendIf(actions, action{request.MethodDeleteCollection, resourcePath, resourceParams, namer, false}, isCollectionDeleter)
   // DEPRECATED in 1.11
   actions = appendIf(actions, action{request.MethodWatchList, "watch/" + resourcePath, resourceParams, namer, false}, allowWatchList)
   
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
   
   // list or post across namespace.
   // For ex: LIST all pods in all namespaces by sending a LIST request at /api/apiVersion/pods.
   // TODO: more strongly type whether a resource allows these actions on "all namespaces" (bulk delete)
   if !isSubresource {
       actions = appendIf(actions, action{request.MethodList, resource, params, namer, true}, isLister)
       // DEPRECATED in 1.11
       actions = appendIf(actions, action{request.MethodWatchList, "watch/" + resource, params, namer, true}, allowWatchList)
   }
   ```

## 阶段5. 构建版本存储信息

> **L597-L628**

```go
var resourceInfo *storageversion.ResourceInfo
if utilfeature.DefaultFeatureGate.Enabled(features.StorageVersionAPI) &&
utilfeature.DefaultFeatureGate.Enabled(features.APIServerIdentity) &&
isStorageVersionProvider &&
storageVersionProvider.StorageVersion() != nil {

    versioner := storageVersionProvider.StorageVersion()
    encodingGVK, err := getStorageVersionKind(versioner, storage, a.group.Typer)
    if err != nil {
        return nil, nil, err
    }
    decodableVersions := []schema.GroupVersion{}
    if a.group.ConvertabilityChecker != nil {
        decodableVersions = a.group.ConvertabilityChecker.VersionsForGroupKind(fqKindToRegister.GroupKind())
    }

    resourceInfo = &storageversion.ResourceInfo{
        GroupResource: schema.GroupResource{
            Group:    a.group.GroupVersion.Group,
            Resource: apiResource.Name,
        },
        EncodingVersion: encodingGVK.GroupVersion().String(),
        // We record EquivalentResourceMapper first instead of calculate
        // DecodableVersions immediately because API installation must
        // be completed first for us to know equivalent APIs
        EquivalentResourceMapper: a.group.EquivalentResourceRegistry,

        DirectlyDecodableVersions: decodableVersions,

        ServedVersions: a.group.AllServedVersionsByResource[path],
    }
}
```

该功能为**可选的高级功能**，主要用于：

1.  **版本管理**：跟踪资源在 etcd 中的实际存储格式
2. **升级协调**：支持集群升级时的版本迁移
3. **多版本支持**：记录资源可以被解码为哪些版本

## 阶段6. 构建 RequestScope

> **L647-L687**

1. 设置 MediaTypes

   ```go
   for _, s := range a.group.Serializer.SupportedMediaTypes() {
       if len(s.MediaTypeSubType) == 0 || len(s.MediaTypeType) == 0 {
           return nil, nil, fmt.Errorf("all serializers in the group Serializer must have MediaTypeType and MediaTypeSubType set: %s", s.MediaType)
       }
   }
   mediaTypes, streamMediaTypes := negotiation.MediaTypesForSerializer(a.group.Serializer)
   allMediaTypes := append(mediaTypes, streamMediaTypes...)
   ws.Produces(allMediaTypes...)
   ```

   以 MIME 类型 `"application/json"` 为例，其 `MediaTypeSubType` 为 `json`，`MediaTypeType`为`application`。

   通过 `negotiation.MediaTypesForSerializer` 获取支持的格式并整合后，再什么 WebService 支持的格式。

2. 通过 `a.group` 的各个组件，来构建 `RequestScope`

   ```go
   	kubeVerbs := map[string]struct{}{}
   	reqScope := handlers.RequestScope{
   		Serializer:      a.group.Serializer,
   		ParameterCodec:  a.group.ParameterCodec,
   		Creater:         a.group.Creater,
   		Convertor:       a.group.Convertor,
   		Defaulter:       a.group.Defaulter,
   		Typer:           a.group.Typer,
   		UnsafeConvertor: a.group.UnsafeConvertor,
   		Authorizer:      a.group.Authorizer,
   
   		EquivalentResourceMapper: a.group.EquivalentResourceRegistry,
   
   		// TODO: Check for the interface on storage
   		TableConvertor: tableProvider,
   
   		// TODO: This seems wrong for cross-group subresources. It makes an assumption that a subresource and its parent are in the same group version. Revisit this.
   		Resource:    a.group.GroupVersion.WithResource(resource),
   		Subresource: subresource,
   		Kind:        fqKindToRegister,
   
   		AcceptsGroupVersionDelegate: gvAcceptor,
   
   		HubGroupVersion: schema.GroupVersion{Group: fqKindToRegister.Group, Version: runtime.APIVersionInternal},
   
   		MetaGroupVersion: metav1.SchemeGroupVersion,
   
   		MaxRequestBodyBytes: a.group.MaxRequestBodyBytes,
   	}
   	if a.group.MetaGroupVersion != nil {
   		reqScope.MetaGroupVersion = *a.group.MetaGroupVersion
   	}
   ```

   * `reqScope`：完整的请求处理上下文；
   * `kubeVerbs  = map[string]struct{}{}`。

## 阶段7. 转换 Actions 为 Routers 的准备工作

> **L700-L724**

即设置 `FieldManager`

```go
var resetFieldsFilter map[fieldpath.APIVersion]fieldpath.Filter
resetFieldsStrategy, isResetFieldsStrategy := storage.(rest.ResetFieldsStrategy)
if isResetFieldsStrategy {
    resetFieldsFilter = fieldpath.NewExcludeFilterSetMap(resetFieldsStrategy.GetResetFields())
}
if resetFieldsStrategy, isResetFieldsFilterStrategy := storage.(rest.ResetFieldsFilterStrategy); isResetFieldsFilterStrategy {
    if isResetFieldsStrategy {
        return nil, nil, fmt.Errorf("may not implement both ResetFieldsStrategy and ResetFieldsFilterStrategy")
    }
    resetFieldsFilter = resetFieldsStrategy.GetResetFieldsFilter()
}

reqScope.FieldManager, err = managedfields.NewDefaultFieldManager(
    a.group.TypeConverter,
    a.group.UnsafeConvertor,
    a.group.Defaulter,
    a.group.Creater,
    fqKindToRegister,
    reqScope.HubGroupVersion,
    subresource,
    resetFieldsFilter,
)
if err != nil {
    return nil, nil, fmt.Errorf("failed to create field manager: %v", err)
}
```

这里后续会讲解其具体逻辑，这里先只做大致了解：

1. 获取 ResetFields 配置：从 storage 的 strategy 中获取哪些字段需要被重置
2. 创建 FieldManager：用于 Server-Side Apply 和字段管理
3. 设置到 RequestScope：让所有 HTTP handlers 都能使用 FieldManager

## 阶段8. Action 元数据准备

> L726-L

1. 确定响应对象类型

   ```go
   producedObject := storageMeta.ProducesObject(action.Verb)
   if producedObject == nil {
       producedObject = defaultVersionedObject
   }
   reqScope.Namer = action.Namer
   ```

   确定这个 action 返回什么类型的对象，每个对象的 REST 对象会实现 `ProducesObject` 接口，根据该接口就可以知道该返回怎样的类型对象。

2. 确定请求作用域

   ```go
   requestScope := "cluster"
   var namespaced string
   var operationSuffix string
   if apiResource.Namespaced {
       requestScope = "namespace"
       namespaced = "Namespaced"
   }
   if strings.HasSuffix(action.Path, "/{path:*}") {
       requestScope = "resource"
       operationSuffix = operationSuffix + "WithPath"
   }
   if strings.Contains(action.Path, "/{name}") || action.Verb == request.MethodPost {
       requestScope = "resource"
   }
   if action.AllNamespaces {
       requestScope = "cluster"
       operationSuffix = operationSuffix + "ForAllNamespaces"
       namespaced = ""
   }
   ```

   以 List 为例：

   ```go
   // LIST 命名空间内的 Pods
   action.Path = "namespaces/{namespace}/pods"
   action.Verb = "LIST"
   apiResource.Namespaced = true
   action.AllNamespaces = false
   
   → requestScope = "namespace"  // 先设为 namespace
   → requestScope = "resource"   // 因为是 POST，改为 resource（实际上 LIST 不会改）
   → namespaced = "Namespaced"
   → operationSuffix = ""
   
   // 结果：operationId = "listCoreV1NamespacedPod"
   
   // LIST 所有命名空间的 Pods
   action.Path = "pods"
   action.Verb = "LIST"
   apiResource.Namespaced = true
   action.AllNamespaces = true  // ← 关键
   
   → requestScope = "namespace"  // 先设为 namespace
   → requestScope = "cluster"    // 因为 AllNamespaces=true，改为 cluster
   → namespaced = ""
   → operationSuffix = "ForAllNamespaces"
   
   // 结果：operationId = "listCoreV1PodForAllNamespaces"
   ```

3. 映射到 Discovery API 的 Verb

   ```go
   if kubeVerb, found := toDiscoveryKubeVerb[action.Verb]; found {
       if len(kubeVerb) != 0 {
           kubeVerbs[kubeVerb] = struct{}{}
       }
   } else {
       return nil, nil, fmt.Errorf("unknown action verb for discovery: %s", action.Verb)
   }
   ```

   将内部的 HTTP 方法映射到 Kubernetes Discovery API 的动词，映射表如下：

   ```go
   toDiscoveryKubeVerb = map[string]string{
       "GET":               "get",
       "LIST":              "list",
       "POST":              "create",
       "PUT":               "update",
       "PATCH":             "patch",
       "DELETE":            "delete",
       "DELETECOLLECTION":  "deletecollection",
       "WATCH":             "watch",
       "WATCHLIST":         "watch",
       "CONNECT":           "connect",
   }
   ```

4. 初始化 routes 

   ```go
   routes := []*restful.RouteBuilder{}
   ```

5. 处理子资源的父 Kind

   ```go
   // If there is a subresource, kind should be the parent's kind.
   if isSubresource {
       parentStorage, ok := a.group.Storage[resource]
       if !ok {
           return nil, nil, fmt.Errorf("missing parent storage: %q", resource)
       }
   
       fqParentKind, err := GetResourceKind(a.group.GroupVersion, parentStorage, a.group.Typer)
       if err != nil {
           return nil, nil, err
       }
       kind = fqParentKind.Kind
   }
   ```

   对于子资源，获取父资源的 Kind，子资源的 operationId 需要包含父资源的名称，

   例如：`replaceCoreV1NamespacedPodStatus` 而不是 `replaceCoreV1NamespacedStatus`

6. 检查是否需要覆盖 Metrics

   ```go
   verbOverrider, needOverride := storage.(StorageMetricsOverride)
   ```

   某些 storage 可能需要自定义 metrics 的动词名称

   ```go
   // 默认情况
   action.Verb = "GET"
   metricsVerb = "GET"
   
   // 如果 storage 实现了 StorageMetricsOverride
   type MyStorage struct {
       *genericregistry.Store
   }
   
   func (s *MyStorage) OverrideMetricsVerb(verb string) (newVerb string) {
       if verb == "GET" {
           return "READ"  // 在 metrics 中显示为 READ
       }
       return verb
   }
   
   // 结果：metrics 中显示为 "READ" 而不是 "GET"
   ```

7. 处理 API 弃用警告

   ```go
   var (
       warnings       []string
       deprecated     bool
       removedRelease string
   )
   
   {
       versionedPtrWithGVK := versionedPtr.DeepCopyObject()
       versionedPtrWithGVK.GetObjectKind().SetGroupVersionKind(fqKindToRegister)
       currentMajor, currentMinor, _ := deprecation.MajorMinor(versioninfo.Get())
       deprecated = deprecation.IsDeprecated(versionedPtrWithGVK, currentMajor, currentMinor)
       if deprecated {
           removedRelease = deprecation.RemovedRelease(versionedPtrWithGVK)
           warnings = append(warnings, deprecation.WarningMessage(versionedPtrWithGVK))
       }
   ```

   检查这个 API 版本是否已弃用，生成警告信息。

## 阶段9. 生成 Router

下面有很多重复的部分，以 GET 为例进行学习：

```go
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
    route := ws.GET(action.Path).To(handler).
    Doc(doc).
    Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed. Defaults to 'false' unless the user-agent indicates a browser or command-line HTTP tool (curl and wget).")).
    Operation("read"+namespaced+kind+strings.Title(subresource)+operationSuffix).
    Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
    Returns(http.StatusOK, "OK", producedObject).
    Writes(producedObject)
    if isGetterWithOptions {
        if err := AddObjectParams(ws, route, versionedGetOptions); err != nil {
            return nil, nil, err
        }
    }
    addParams(route, action.Params)
    routes = append(routes, route)
```

1. 构建符合 go-restful 的 Handler，：

   ```go
   var handler restful.RouteFunction
   if isGetterWithOptions {
       handler = restfulGetResourceWithOptions(getterWithOptions, reqScope, isSubresource)
   } else {
       handler = restfulGetResource(getter, reqScope)
   }
   ```

   根据 GET 请求是否需要额外配置，来选择进入那条分支，

   * `restfulGetResource`：适配大多数资源，不需要额外的查询参数。
   * `restfulGetResourceWithOptions`：需要自定义的 Options 对象，支持额外的查询参数，可能支持子路径。

   以 `restfulGetResource` 为例，返回闭包：

   ```go
   func restfulGetResource(r rest.Getter, scope handlers.RequestScope) restful.RouteFunction {
   	return func(req *restful.Request, res *restful.Response) {
   		handlers.GetResource(r, &scope)(res.ResponseWriter, req.Request)
   	}
   }
   ```

   该函数调用分两步：

   * 第一次调用：`handlers.GetResource(r, &scope)`

     ```go
     // handlers/get.go line 89
     func GetResource(r rest.Getter, scope *RequestScope) http.HandlerFunc {
         return getResourceHandler(scope,
             func(ctx context.Context, name string, req *http.Request) (runtime.Object, error) {
                 // ... 解析参数
                 return r.Get(ctx, name, &options)
             })
     }
     ```

     * 返回 Go 标准库定义的 HandlerFunc，签名为：`func(http.ResponseWriter, *http.Request)`

   * 第二次调用：`(res.ResponseWriter, req.Request)`，即立即执行该函数。

     * `res.ResponseWriter` - go-restful 的 Response 包含标准的 `http.ResponseWriter`
     * `req.Request` - go-restful 的 Request 包含标准的 `*http.Request`

   该函数返回如下函数：

   1. 入参是 go-restful 的格式：`(req *restful.Request, res *restful.Response)`
   2. 内部逻辑：从入参中提取标准 HTTP 对象，调用标准 HTTP 处理器
   3. 返回的函数符合 go-restful 的 `RouteFunction` 类型

2. 包装 `metrics`

   ```go
   if needOverride {
       // need change the reported verb
       handler = metrics.InstrumentRouteFunc(verbOverrider.OverrideMetricsVerb(action.Verb), group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, handler)
   } else {
       handler = metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, handler)
   }
   ```

   * 给 handler 包一层监控，记录请求次数、延迟等指标

3. 添加警告处理

   ```go
   handler = utilwarning.AddWarningsHandler(handler, warnings)
   ```

   * 如果 API 已废弃，在响应头里加警告信息

4. 构建路由

   ```go
   route := ws.GET(action.Path).To(handler).
       Doc(doc).  // 文档说明
       Param(ws.QueryParameter("pretty", "...")).  // 添加查询参数
       Operation("read"+namespaced+kind+strings.Title(subresource)+operationSuffix).  // 操作名
       Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).  // 支持的响应格式
       Returns(http.StatusOK, "OK", producedObject).  // 成功返回 200
       Writes(producedObject)  // 返回的对象类型
   ```

5. 添加额外参数并保存路由

   ```go
   if isGetterWithOptions {
       // 如果有选项，添加选项参数（比如 tailLines, follow 等）
       if err := AddObjectParams(ws, route, versionedGetOptions); err != nil {
           return nil, nil, err
       }
   }
   addParams(route, action.Params)  // 添加路径参数（namespace, name）
   routes = append(routes, route)   // 保存路由
   ```

## 阶段10. 保存 Route

```go
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

设置好元数据后，将路由注册到 WebService。

## 阶段11. 收尾工作

1. 填充支持的操作

   ```go
   apiResource.Verbs = make([]string, 0, len(kubeVerbs))
   for kubeVerb := range kubeVerbs {
       apiResource.Verbs = append(apiResource.Verbs, kubeVerb)
   }
   sort.Strings(apiResource.Verbs)
   ```

   * 作用：告诉客户端这个资源支持哪些操作


   * 结果：["create", "delete", "get", "list", "patch", "update", "watch"]


   * 用途：kubectl 知道可以对 Pod 执行哪些命令

2. 填充其他元数据

   ```go
   // 短名称：po, svc, deploy
   if shortNamesProvider, ok := storage.(rest.ShortNamesProvider); ok {
       apiResource.ShortNames = shortNamesProvider.ShortNames()
   }
   
   // 分类：all
   if categoriesProvider, ok := storage.(rest.CategoriesProvider); ok {
       apiResource.Categories = categoriesProvider.Categories()
   }
   
   // 单数名称：pod, service
   if !isSubresource {
       apiResource.SingularName = singularNameProvider.GetSingularName()
   }
   
   // GVK 信息（特殊情况才覆盖）
   if gvkProvider, ok := storage.(rest.GroupVersionKindProvider); ok {
       gvk := gvkProvider.GroupVersionKind(a.group.GroupVersion)
       apiResource.Group = gvk.Group
       apiResource.Version = gvk.Version
       apiResource.Kind = gvk.Kind
   }
   ```

3. 注册映射关系并返回

   ```go
   // 注册：GVR (pods) → GVK (Pod)
   a.group.EquivalentResourceRegistry.RegisterKindFor(reqScope.Resource, reqScope.Subresource, fqKindToRegister)
   
   // 返回结果
   return &apiResource, resourceInfo, nil
   ```

   在等价资源注册表中注册 GroupVersionResource (GVR) 到 GroupVersionKind (GVK) 的映射关系，用于查找URL 和 Go 类型的对应关系，示例如下：

   ```go
   // 主资源
   GVR: {Group:"", Version:"v1", Resource:"pods"}
     ↔
   GVK: {Group:"", Version:"v1", Kind:"Pod"}
   
   // 子资源
   GVR: {Group:"", Version:"v1", Resource:"pods"}, Subresource: "status"
     ↔
   GVK: {Group:"", Version:"v1", Kind:"Pod"}
   
   // 跨版本映射
   GVR: {Group:"apps", Version:"v1", Resource:"deployments"}
     ↔
   GVK: {Group:"apps", Version:"v1", Kind:"Deployment"}
   
   GVR: {Group:"apps", Version:"v1beta1", Resource:"deployments"}
     ↔
   GVK: {Group:"apps", Version:"v1beta1", Kind:"Deployment"}
   ```

   
