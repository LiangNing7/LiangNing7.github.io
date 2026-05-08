+++
title = "K8s Generic Server Strategy"
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

series_order = 10

+++

# K8s Generic Server Strategy

## REST 结构体定义

以 Pod 和 Deployment 为例，它们 `rest.Storage` 实例的实际类型分别是如下 REST 结构体：

* Pod 的 `REST` 结构体如下：

  ```go
  // 代码: pkg\registry\core\pod\storage\storage.go#L67-L71
  // REST implements a RESTStorage for pods
  type REST struct {
  	*genericregistry.Store
  	proxyTransport http.RoundTripper
  }
  ```

* Deployment 的 `REST` 结构体如下：

  ```go
  // 代码: pkg\registry\apps\deployment\storage\storage.go#L87-L90
  // REST implements a RESTStorage for Deployments.
  type REST struct {
  	*genericregistry.Store
  }
  ```

创建 Pod 和 Deployment 不可能一样。规避这个问题的方法隐藏在`generic/registry.Store` 结构体的字段中，这些字段绝大多数都是扩展点，不同 API 通过给这些扩展点赋予不同的值来控制响应方法的内部操作，比较典型的是`*Strategy` 系列属性：`CreateStrategy`、`DeleteStrategy`、`UpdateStrategy`、`ResetFieldsStrategy`。

## Generic Server 的 Strategy 定义

**策略用来落实存储前后的个性化需求**。实现一个策略只需要实现它对应的接口，这些接口由 Generic Server 库定义， 以 `rest.RESTCreateStrategy` 接口为例：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\registry\rest\create.go#L40-#L90
type RESTCreateStrategy interface {
	runtime.ObjectTyper

	names.NameGenerator

	NamespaceScoped() bool

	Validate(ctx context.Context, obj runtime.Object) field.ErrorList

	WarningsOnCreate(ctx context.Context, obj runtime.Object) []string

	Canonicalize(obj runtime.Object)
}
```

介绍一下该接口的几个重要方法作用：

* `NamespaceScoped()`：如果该 API 实例必须明确隶属于命名空间，则这个方法需要返回 true。
* `PrepareForCreate()`：对实例信息进行整理，例如从实例属性上抹去那些不想保存的信息，对顺序敏感的内容排序等等。它发生在 `Validate()` 方法之前。
* `Validate()`：对实例信息进行校验，返回发现的所有错误的列表。
* `WarningsOnCreate()`：给客户端返回告警信息，例如请求中使用了某个即将废弃的 字段。它发生在 `Validate()` 之后、`Canonicalize()` 方法之前，数据也还没有保存进 ETCD。本方 法内不可以修改 API 实例信息。
* `Canonicalize()`：在保存进 ETCD 前修改 API 实例。这些修改一般出于信息格式化的目的，使得格式更符合惯例。如果不需要那它的实现也可以留空。

## 资源自己实现 Storage 接口

在 Generic Store 中定义了这些策略接口后，在对应的资源中，就会实现该接口，从而实现该策略。以 Pod 和 Deployment 为例，

* Pod 的策略实现位于：`pkg\registry\core\pod\strategy.go`；
* Deployment 的策略实现位于：`pkg\registry\apps\deployment\strategy.go`。

在完成策略的实现后，就可以在实例化 `Storage` 中使用该策略了。

## Storage 初始化

而在对 Pod 与 Deployment 的 Storage 初始化如下：

* Pod 的 Store 初始化：

  ```go
  // 代码: pkg\registry\core\pod\storage\storage.go#L74-L124
  // NewStorage returns a RESTStorage object that will work against pods.
  func NewStorage(optsGetter generic.RESTOptionsGetter, k client.ConnectionInfoGetter, proxyTransport http.RoundTripper, podDisruptionBudgetClient policyclient.PodDisruptionBudgetsGetter) (PodStorage, error) {
  
  	store := &genericregistry.Store{
  		NewFunc:                   func() runtime.Object { return &api.Pod{} },
  		NewListFunc:               func() runtime.Object { return &api.PodList{} },
  		PredicateFunc:             registrypod.MatchPod,
  		DefaultQualifiedResource:  api.Resource("pods"),
  		SingularQualifiedResource: api.Resource("pod"),
  
          // 要点①
  		CreateStrategy:      registrypod.Strategy,
  		UpdateStrategy:      registrypod.Strategy,
  		DeleteStrategy:      registrypod.Strategy,
  		ResetFieldsStrategy: registrypod.Strategy,
  		ReturnDeletedObject: true,
  
  		TableConvertor: printerstorage.TableConvertor{TableGenerator: printers.NewTableGenerator().With(printersinternal.AddHandlers)},
  	}
      ...
  }
  ```

* Deployment 的 Store 初始化：

  ```go
  // 代码: pkg\registry\apps\deployment\storage\storage.go#L93-L106
  // NewREST returns a RESTStorage object that will work against deployments.
  func NewREST(optsGetter generic.RESTOptionsGetter) (*REST, *StatusREST, *RollbackREST, error) {
  	store := &genericregistry.Store{
  		NewFunc:                   func() runtime.Object { return &apps.Deployment{} },
  		NewListFunc:               func() runtime.Object { return &apps.DeploymentList{} },
  		DefaultQualifiedResource:  apps.Resource("deployments"),
  		SingularQualifiedResource: apps.Resource("deployment"),
  
          // 要点②
  		CreateStrategy:      deployment.Strategy,
  		UpdateStrategy:      deployment.Strategy,
  		DeleteStrategy:      deployment.Strategy,
  		ResetFieldsStrategy: deployment.Strategy,
  
  		TableConvertor: printerstorage.TableConvertor{TableGenerator: printers.NewTableGenerator().With(printersinternal.AddHandlers)},
  	}
  	options := &generic.StoreOptions{RESTOptions: optsGetter}
  	if err := store.CompleteWithOptions(options); err != nil {
  		return nil, nil, nil, err
  	}
  
  	statusStore := *store
  	statusStore.UpdateStrategy = deployment.StatusStrategy
  	statusStore.ResetFieldsStrategy = deployment.StatusStrategy
  	return &REST{store}, &StatusREST{store: &statusStore}, &RollbackREST{store: store}, nil
  }
  ```

分别在要点①和要点②处注入个性化的策略逻辑。

## 策略执行流程详解

下面将详细讲解 Kubernetes API Server 中策略模式的执行流程，以创建 Pod 为例，展示从 HTTP 请求到 etcd 存储的完整过程。

### 1. 执行流程概览

```
HTTP POST /api/v1/namespaces/default/pods
    ↓
PodREST.Create()
    ↓
Store.Create() → Store.create()
    ↓
通用逻辑：初始化元数据
    ↓
策略钩子：rest.BeforeCreate(e.CreateStrategy, ctx, obj)
    ├─ strategy.PrepareForCreate()  ← Pod 特有逻辑
    ├─ strategy.Validate()          ← Pod 特有验证
    ├─ strategy.WarningsOnCreate()  ← Pod 特有警告
    └─ strategy.Canonicalize()      ← Pod 特有规范化
    ↓
通用逻辑：生成 etcd key
    ↓
通用逻辑：保存到 etcd
    ↓
策略钩子：AfterCreate (可选)
    ↓
返回创建的对象
```

### 2. 详细步骤

#### 2.1 HTTP 请求到达

```
HTTP POST /api/v1/namespaces/default/pods
Content-Type: application/json

{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "nginx"
  },
  "spec": {
    "containers": [...]
  }
}
```

#### 2.2 路由到 REST 层

请求被路由到 `PodREST.Create()`，但因为 `PodREST` 嵌套了 `*Store`，实际调用的是 `Store.Create()`

**代码位置：** `staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go:447`

```go
func (e *Store) Create(ctx context.Context, obj runtime.Object, 
                       createValidation rest.ValidateObjectFunc, 
                       options *metav1.CreateOptions) (runtime.Object, error) {
    if utilfeature.DefaultFeatureGate.Enabled(features.RetryGenerateName) && 
       needsNameGeneration(obj) {
        return e.createWithGenerateNameRetry(ctx, obj, createValidation, options)
    }
    
    return e.create(ctx, obj, createValidation, options)
}
```

#### 2.3 进入 Store.create() 方法

**代码位置：** `staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go:477`

##### 2.3.1 初始化元数据（通用逻辑）

**代码位置：** `store.go:480-488`

```go
func (e *Store) create(ctx context.Context, obj runtime.Object, 
                       createValidation rest.ValidateObjectFunc, 
                       options *metav1.CreateOptions) (runtime.Object, error) {
    var finishCreate FinishFunc = finishNothing

    // 通用逻辑：填充系统字段
    if objectMeta, err := meta.Accessor(obj); err != nil {
        return nil, err
    } else {
        // 填充 CreationTimestamp, UID 等系统字段
        rest.FillObjectMetaSystemFields(objectMeta)
        
        // 如果使用 generateName，生成实际名称
        if len(objectMeta.GetGenerateName()) > 0 && len(objectMeta.GetName()) == 0 {
            // 注意：这里调用了策略的 GenerateName 方法
            objectMeta.SetName(e.CreateStrategy.GenerateName(objectMeta.GetGenerateName()))
        }
    }
    
    // ... 省略 BeginCreate 钩子 ...
}
```

**此时对象状态（Pod 示例）：**
```yaml
metadata:
  name: nginx-abc123              # 如果用了 generateName，这里已生成
  creationTimestamp: "2024-01-01T00:00:00Z"  # 已填充
  uid: "xxx-xxx-xxx"              # 已填充
```

##### 2. 3.2 调用策略钩子 BeforeCreate

**代码位置：** `store.go:500`

```go
// 关键！调用策略钩子
if err := rest.BeforeCreate(e.CreateStrategy, ctx, obj); err != nil {
    return nil, err
}
```

**这里发生了什么？**

`e.CreateStrategy` 是在初始化时注入的策略对象：

- 对于 Pod：`e.CreateStrategy = registrypod.Strategy`（在 `pod/storage/storage.go:82` 注入）
- 对于 Deployment：`e.CreateStrategy = deployment.Strategy`（在 `deployment/storage/storage.go:100` 注入）

#### 2.4 BeforeCreate 内部的策略调用

**代码位置：** `staging/src/k8s.io/apiserver/pkg/registry/rest/create.go:95`

```go
func BeforeCreate(strategy RESTCreateStrategy, ctx context.Context, 
                  obj runtime.Object) error {
    objectMeta, kind, kerr := objectMetaAndKind(strategy, obj)
    if kerr != nil {
        return kerr
    }

    // 通用检查：确保元数据已初始化
    if !metav1.HasObjectMetaSystemFieldValues(objectMeta) {
        return errors.NewInternalError(fmt.Errorf("system metadata was not initialized"))
    }

    // 通用检查：确保名称已生成
    if len(objectMeta.GetGenerateName()) > 0 && len(objectMeta.GetName()) == 0 {
        return errors.NewInternalError(fmt.Errorf("metadata.name was not generated"))
    }

    // 通用检查：确保命名空间正确
    requestNamespace, ok := genericapirequest.NamespaceFrom(ctx)
    if !ok {
        return errors.NewInternalError(fmt.Errorf("no namespace information found in request context"))
    }
    if err := EnsureObjectNamespaceMatchesRequestNamespace(
        ExpectedNamespaceForScope(requestNamespace, strategy.NamespaceScoped()), 
        objectMeta); err != nil {
        return err
    }

    // ========== 策略方法调用开始 ==========
    
    // 4.1 调用策略的 PrepareForCreate
    strategy.PrepareForCreate(ctx, obj)

    // 4.2 调用策略的 Validate
    if errs := strategy.Validate(ctx, obj); len(errs) > 0 {
        return errors.NewInvalid(kind.GroupKind(), objectMeta.GetName(), errs)
    }

    // 通用验证：验证对象元数据
    if errs := genericvalidation.ValidateObjectMetaAccessor(
        objectMeta, strategy.NamespaceScoped(), 
        path.ValidatePathSegmentName, field.NewPath("metadata")); len(errs) > 0 {
        return errors.NewInvalid(kind.GroupKind(), objectMeta.GetName(), errs)
    }

    // 4.3 调用策略的 WarningsOnCreate
    for _, w := range strategy.WarningsOnCreate(ctx, obj) {
        warning.AddWarning(ctx, "", w)
    }

    // 4.4 调用策略的 Canonicalize
    strategy.Canonicalize(obj)

    return nil
}
```

##### 2.4.1 策略方法：PrepareForCreate

**这是第一个策略方法调用！** 不同资源有不同的实现。

**Pod 的实现：** `pkg/registry/core/pod/strategy.go:87`

```go
func (podStrategy) PrepareForCreate(ctx context.Context, obj runtime.Object) {
    pod := obj.(*api.Pod)
    
    // Pod 特有：设置初始 Generation
    pod.Generation = 1
    
    // Pod 特有：设置初始状态
    pod.Status = api.PodStatus{
        Phase:    api.PodPending,      // 设置为 Pending 状态
        QOSClass: qos.GetPodQOS(pod),  // 计算 QoS 等级
    }

    // Pod 特有：删除禁用的字段
    podutil.DropDisabledPodFields(pod, nil)

    // Pod 特有：应用调度门控
    applySchedulingGatedCondition(pod)
    
    // Pod 特有：处理亲和性
    mutatePodAffinity(pod)
    
    // Pod 特有：处理拓扑分布约束
    mutateTopologySpreadConstraints(pod)
    
    // Pod 特有：应用 AppArmor 版本偏差
    applyAppArmorVersionSkew(ctx, pod)
}
```

**Deployment 的实现：** `pkg/registry/apps/deployment/strategy.go:74`

```go
func (deploymentStrategy) PrepareForCreate(ctx context.Context, obj runtime.Object) {
    deployment := obj.(*apps.Deployment)
    
    // Deployment 特有：清空状态
    deployment.Status = apps.DeploymentStatus{}
    
    // Deployment 特有：设置初始 Generation
    deployment.Generation = 1

    // Deployment 特有：处理 Pod 模板字段
    pod.DropDisabledTemplateFields(&deployment.Spec.Template, nil)
}
```

**对比总结：**

| 操作 | Pod | Deployment |
|------|-----|------------|
| **设置 Generation** | ✓ 设置为 1 | ✓ 设置为 1 |
| **初始化状态** | 设置 Phase=Pending<br>计算 QoS 等级 | 清空 Status |
| **特有逻辑** | 调度门控<br>亲和性处理<br>拓扑分布约束 | 处理 Pod 模板 |

**此时对象状态（Pod 示例）：**
```yaml
metadata:
  name: nginx-abc123
  generation: 1                    # ← PrepareForCreate 设置
  creationTimestamp: "2024-01-01T00:00:00Z"
  uid: "xxx-xxx-xxx"
status:
  phase: Pending                   # ← PrepareForCreate 设置
  qosClass: BestEffort             # ← PrepareForCreate 计算
```

##### 2.4.2 策略方法：Validate

**这是第二个策略方法调用！** 验证对象是否符合规范。

**Pod 的实现：** `pkg/registry/core/pod/strategy.go:113`

```go
func (podStrategy) Validate(ctx context.Context, obj runtime.Object) field.ErrorList {
    pod := obj.(*api.Pod)
    opts := podutil.GetValidationOptionsFromPodSpecAndMeta(
        &pod.Spec, nil, &pod.ObjectMeta, nil)
    opts.ResourceIsPod = true
    
    // Pod 特有验证：调用核心验证逻辑
    return corevalidation.ValidatePodCreate(pod, opts)
}
```

**Pod 验证内容包括：**
- 容器配置验证（镜像、命令、参数等）
- 资源限制验证（CPU、内存等）
- 卷挂载验证
- 端口配置验证
- 探针配置验证
- 安全上下文验证

**Deployment 的实现：** `pkg/registry/apps/deployment/strategy.go:83`

```go
func (deploymentStrategy) Validate(ctx context.Context, obj runtime.Object) field.ErrorList {
    deployment := obj.(*apps.Deployment)
    opts := pod.GetValidationOptionsFromPodTemplate(&deployment.Spec.Template, nil)
    
    // Deployment 特有验证：调用 apps 验证逻辑
    return appsvalidation.ValidateDeployment(deployment, opts)
}
```

**Deployment 验证内容包括：**
- 副本数验证（必须 >= 0）
- 选择器验证（必须匹配 Pod 模板标签）
- 滚动更新策略验证
- Pod 模板验证
- 暂停状态验证

**如果验证失败：**
```go
if errs := strategy.Validate(ctx, obj); len(errs) > 0 {
    return errors.NewInvalid(kind.GroupKind(), objectMeta.GetName(), errs)
    // 返回 HTTP 422 Unprocessable Entity
    // 创建流程终止
}
```

**验证失败示例：**
```json
{
  "kind": "Status",
  "status": "Failure",
  "message": "Pod \"nginx\" is invalid",
  "reason": "Invalid",
  "details": {
    "name": "nginx",
    "kind": "Pod",
    "causes": [
      {
        "reason": "FieldValueRequired",
        "message": "Required value: must specify at least one container",
        "field": "spec.containers"
      }
    ]
  },
  "code": 422
}
```

##### 2.4.3 策略方法：WarningsOnCreate

**这是第三个策略方法调用！** 返回警告信息（不会阻止创建）。

**Pod 的实现：** `pkg/registry/core/pod/strategy.go:120`

```go
func (podStrategy) WarningsOnCreate(ctx context.Context, obj runtime.Object) []string {
    newPod := obj.(*api.Pod)
    var warnings []string
    
    // 警告：Pod 名称不符合 DNS 标签规范
    if msgs := utilvalidation.IsDNS1123Label(newPod.Name); len(msgs) != 0 {
        warnings = append(warnings, fmt.Sprintf(
            "metadata.name: this is used in the Pod's hostname, "+
            "which can result in surprising behavior; "+
            "a DNS label is recommended: %v", msgs))
    }
    
    // 获取 Pod 相关的其他警告
    warnings = append(warnings, podutil.GetWarningsForPod(ctx, newPod, nil)...)
    
    return warnings
}
```

**Deployment 的实现：** `pkg/registry/apps/deployment/strategy.go:89`

```go
func (deploymentStrategy) WarningsOnCreate(ctx context.Context, obj runtime.Object) []string {
    newDeployment := obj.(*apps.Deployment)
    var warnings []string
    
    // 警告：Deployment 名称不符合 DNS 标签规范
    if msgs := utilvalidation.IsDNS1123Label(newDeployment.Name); len(msgs) != 0 {
        warnings = append(warnings, fmt.Sprintf(
            "metadata.name: this is used in Pod names and hostnames, "+
            "which can result in surprising behavior; "+
            "a DNS label is recommended: %v", msgs))
    }
    
    // 获取 Pod 模板相关的警告
    warnings = append(warnings, pod.GetWarningsForPodTemplate(
        ctx, field.NewPath("spec", "template"), 
        &newDeployment.Spec.Template, nil)...)
    
    return warnings
}
```

**警告示例：**
```
Warning: metadata.name: this is used in the Pod's hostname, which can result in 
surprising behavior; a DNS label is recommended: [must be no more than 63 characters]
```

**注意：** 警告不会阻止创建，只是提醒用户注意潜在问题。

##### 2.4.4 策略方法：Canonicalize

**这是第四个策略方法调用！** 规范化对象（例如排序、去重等）。

```go
func (podStrategy) Canonicalize(obj runtime.Object) {
    // Pod 的规范化逻辑（如果有）
}

func (deploymentStrategy) Canonicalize(obj runtime.Object) {
    // Deployment 的规范化逻辑（如果有）
}
```

大多数资源的 Canonicalize 方法是空实现，但提供了扩展点。

#### 2.5 回到 Store.create() 继续执行

策略方法执行完毕后，回到 `Store.create()` 继续执行通用逻辑。

##### 2.5.1 生成 etcd key

**代码位置：** `store.go:511-516`

```go
name, err := e.ObjectNameFunc(obj)  // 获取对象名称
if err != nil {
    return nil, err
}

key, err := e.KeyFunc(ctx, name)    // 生成 etcd key
if err != nil {
    return nil, err
}
```

**生成的 key 示例：**
- Pod: `/registry/pods/default/nginx-abc123`
- Deployment: `/registry/deployments/default/nginx-deployment`
- Service: `/registry/services/specs/default/nginx-service`

##### 2.5.2 计算 TTL

**代码位置：** `store.go:522-525`

```go
qualifiedResource := e.qualifiedResourceFromContext(ctx)
ttl, err := e.calculateTTL(obj, 0, false)
if err != nil {
    return nil, err
}
```

TTL（Time To Live）用于设置对象在 etcd 中的过期时间，大多数资源不设置 TTL（永久存储）。

##### 2.5.3 保存到 etcd

**代码位置：** `store.go:527`

```go
out := e.NewFunc()
if err := e.Storage.Create(ctx, key, obj, out, ttl, 
                           dryrun.IsDryRun(options.DryRun)); err != nil {
    err = storeerr.InterpretCreateError(err, qualifiedResource, name)
    err = rest.CheckGeneratedNameError(ctx, e.CreateStrategy, err, obj)
    if !apierrors.IsAlreadyExists(err) {
        return nil, err
    }
    // ... 错误处理 ...
    return nil, err
}
```

**这里实际写入 etcd 数据库！**

写入的数据格式（简化）：
```
Key: /registry/pods/default/nginx-abc123
Value: {
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "nginx-abc123",
    "namespace": "default",
    "uid": "xxx-xxx-xxx",
    "creationTimestamp": "2024-01-01T00:00:00Z",
    "generation": 1
  },
  "spec": { ... },
  "status": {
    "phase": "Pending",
    "qosClass": "BestEffort"
  }
}
```

##### 2.5.4 调用 AfterCreate 钩子（可选）

**代码位置：** `store.go:551-553`

```go
// The operation has succeeded. Call the finish function if there is one,
// and then make sure the defer doesn't call it again.
fn := finishCreate
finishCreate = finishNothing
fn(ctx, true)

if e.AfterCreate != nil {
    e.AfterCreate(out, options)  // 可选的后置钩子
}
```

`AfterCreate` 是一个可选的钩子，大多数资源不使用。如果定义了，可以在对象创建后执行额外的逻辑。

##### 2.5.5 装饰和返回结果

**代码位置：** `store.go:554-557`

```go
if e.Decorator != nil {
    e.Decorator(out)  // 装饰返回对象（可选）
}

return out, nil  // 返回创建的对象
```

**HTTP 响应：**
```
HTTP/1.1 201 Created
Content-Type: application/json

{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "nginx-abc123",
    "namespace": "default",
    "uid": "xxx-xxx-xxx",
    "resourceVersion": "12345",
    "creationTimestamp": "2024-01-01T00:00:00Z",
    "generation": 1
  },
  "spec": { ... },
  "status": {
    "phase": "Pending",
    "qosClass": "BestEffort"
  }
}
```

### 3. 完整流程图（带代码位置）

```
HTTP POST /api/v1/namespaces/default/pods
    ↓
┌────────────────────────────────────────────────────────────────┐
│ 1. PodREST.Create()                                            │
│    → 因为嵌套了 *Store，实际调用 Store.Create()                 │
└────────────────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────────────────┐
│ 2. Store.Create() [store.go:447]                              │
│    → 调用 e.create()                                           │
└────────────────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────────────────┐
│ 3. Store.create() [store.go:477]                              │
│                                                                │
│    3.1 通用逻辑：初始化元数据 [480-488行]                       │
│        - rest.FillObjectMetaSystemFields()                    │
│        - 生成名称（如果需要）                                   │
│                                                                │
│    3.2 策略钩子：rest.BeforeCreate() [500行]                   │
│        传入参数：e.CreateStrategy                              │
│        → Pod: e.CreateStrategy = registrypod.Strategy         │
│        → Deployment: e.CreateStrategy = deployment.Strategy   │
└────────────────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────────────────┐
│ 4. rest.BeforeCreate() [create.go:95]                         │
│                                                                │
│    4.1 通用检查 [96-118行]                                     │
│        - 检查元数据是否初始化                                   │
│        - 检查命名空间是否正确                                   │
│                                                                │
│    4.2 策略方法：strategy.PrepareForCreate() [120行]          │
│        ┌─────────────────────────────────────────────┐        │
│        │ Pod 的实现 [pod/strategy.go:87]             │        │
│        │   pod.Generation = 1                        │        │
│        │   pod.Status.Phase = Pending                │        │
│        │   pod.Status.QOSClass = qos.GetPodQOS(pod)  │        │
│        │   applySchedulingGatedCondition(pod)        │        │
│        └─────────────────────────────────────────────┘        │
│        ┌─────────────────────────────────────────────┐        │
│        │ Deployment 的实现 [deployment/strategy.go:74]│       │
│        │   deployment.Status = DeploymentStatus{}    │        │
│        │   deployment.Generation = 1                 │        │
│        │   pod.DropDisabledTemplateFields(...)       │        │
│        └─────────────────────────────────────────────┘        │
│                                                                │
│    4.3 策略方法：strategy.Validate() [122行]                  │
│        ┌─────────────────────────────────────────────┐        │
│        │ Pod 的实现 [pod/strategy.go:113]            │        │
│        │   corevalidation.ValidatePodCreate(pod)     │        │
│        │   验证：容器配置、资源限制、卷挂载           │        │
│        └─────────────────────────────────────────────┘        │
│        ┌─────────────────────────────────────────────┐        │
│        │ Deployment 的实现 [deployment/strategy.go:83]│       │
│        │   appsvalidation.ValidateDeployment(...)    │        │
│        │   验证：副本数、选择器、滚动更新策略         │        │
│        └─────────────────────────────────────────────┘        │
│                                                                │
│    4.4 策略方法：strategy.WarningsOnCreate() [133行]          │
│        返回警告信息（不阻止创建）                               │
│                                                                │
│    4.5 策略方法：strategy.Canonicalize() [137行]              │
│        规范化对象                                              │
└────────────────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────────────────┐
│ 5. 回到 Store.create() 继续执行                                │
│                                                                │
│    5.1 通用逻辑：生成 etcd key [511-516行]                     │
│        key = "/registry/pods/default/nginx-abc123"            │
│                                                                │
│    5.2 通用逻辑：计算 TTL [522-525行]                          │
│                                                                │
│    5.3 通用逻辑：保存到 etcd [527行]                           │
│        e.Storage.Create(ctx, key, obj, out, ttl, ...)         │
│        ← 实际写入 etcd 数据库                                  │
│                                                                │
│    5.4 策略钩子：e.AfterCreate() [551-553行]                  │
│        可选的后置钩子                                          │
│                                                                │
│    5.5 通用逻辑：返回结果 [554-557行]                          │
│        return out, nil                                        │
└────────────────────────────────────────────────────────────────┘
    ↓
HTTP 201 Created
Body: { 创建的 Pod 对象 }
```
