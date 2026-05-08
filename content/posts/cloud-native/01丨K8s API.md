+++
title = "K8s API"
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
series_order = 1

+++

# 1. Kubernetes API 源代码

本节聚焦 Kubernetes API 的技术形态，讲解 Kubernetes 代码中如何表现这一事物。为了便于理解，选 Deployment 这一常用的 API 来做示例。Deployment 资源的定义文件如下：

```yaml
apiVersion: apps/v1    # 组名/版本
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

资源定义文件用于描述 API 实例，所以其内容反映了 API 的部分属性。一个 API 至少有如下属性或属性集合：

1. `apiVersion`：由 API 所在的组和版本构成，中间加斜线分割。
2. `kind`：API 种类。它和 `apiVersion` 一起给出了 API 的 GVK 信息，GVK 精确确定 了 API，确定了其技术类型。这一点在源码的 Scheme 部分有印证：`Scheme` 内部维护了 GVK 和用于承载 API 的 Go 基座结构体的映射关系。
3. `metadata`：`API` 的元数据，主要是名字，标签，注解（Annotation）等等。
4. `spec`：对目标状态期望的描述，例如这里我们指定希望有 3 个副本同时运行

我们可以通过 `kubectl` 提供的 `describe` 命令来获得完整的资源属性：使用上述资源定义文件在 `API Server` 上创建出 `Deployment` 后，针对创建出的 `Deployment` 运行 `describe` 命令：

![image-20251119150901223](http://images.liangning7.cn/typora/202511191509358.png)

通过资源定义文件和 `describe` 命令，确实看到了不少 API 属性，但也只是系统显示的那一部分，依然不完全。其实，可以通过代码来获取 `Deployment` 所具有的所有属性，这是更为精确的。在进入代码前，先要认识 API 实例的内外部版本。

## 1.1 内部版本与外部版本

### 1.1.1 版本转换

API 是以组（Group）为单位，按照版本演化的，一般来说，组的演化过程是这样的：最开始是 `alpha` 版，如 `v1alpha1`；然后是 beta 版，如 `v1beta1`, `v1beta2`；最后是正式版本 `v1`，如此延续。Deployment 所在的 apps 组就经过了这样一个过程：

`v1beta1 → v1beta2 → v1`

`Deployment` 在最早的 `v1beta1` 中就被引入了，在后续版本中也一直存在，那么这里就有个问题：用户基于老版本 `Deployment` 编写的资源定义文件是否能被新版本 `API Server` 正确理解呢？答案是肯定的。

为了新版本的 `API Server` 能正确理解老版本 `Deployment`，支持向后兼容是关键。为了达到每个版本之间都能相互转换，一种笨办法就是在各个版本之间进行两两转换，如果有 3 个版本就会有 6 段转换逻辑，分别是 1 到 2、1 到 3、2 到 3、 以及逆向过程。推而广之，如果有 N 个版本，转换关系将是个网状的结构，数量级是 N 的平方，规模很大的。

![image-20251119151611556](http://images.liangning7.cn/typora/202511191518748.png)

Kubernetes 引入了内部版本和外部版本来简化转换过程。内部版本只在 Kubernetes 代码内部使用，业务逻辑代码是基于内部版本编写的。内部版本始终只有一个，不需要版本编号。 在每个新 Kubernetes 的发布版本中，众多 API 的内部版本都会被更新，从而能同时承载该API 在老版本中已有属性和在新版本中加入的属性。而外部版本是为 API Server 的消费者准备的，一个 API会有多个外部版本，分别对应API 组在演化过程中出现的各个版本：v1beta1、 v1beta2、v1 等等。与内部版本不同，一个外部版本是固定的，后续版本的发布不会影响前序版本的定义。

内外部版本转换的机制如下图所示：

![image-20251017223138375](http://images.liangning7.cn/typora/202510172231553.png)

Kubernetes 的版本转换呈星状结构。外部版本号不会直接互转，通常都是先转换成内部版本号，再转换为期望的外部版本号。例如：

```
v1 --> internal version --> v2beta
```

所有外部版本，都先转换为内部版本，再转换为目标版本，通过这种方式，可以让整个版本转换更加简洁、高效。否则，每增加一个新的版本，其他版本都要实现一个新的向目标版本转换的方法，会让程序变得非常难以维护、臃肿。

### 1.1.2 内部版本

再进一步，看代码上内外部版本的技术类型。由于始终只有一个，API 的内部版本只需要一个 Go 结构体就可以表示了。以 Deployment 为例，其内部版本的 Go 结构体定义如下：

```go
//代码: pkg/apis/apps/types.go
// Deployment provides declarative updates for Pods and ReplicaSets.
type Deployment struct {
	metav1.TypeMeta
	// +optional
	metav1.ObjectMeta

	// Specification of the desired behavior of the Deployment.
    // 期望的说明
	// +optional
	Spec DeploymentSpec

	// Most recently observed status of the Deployment.
    // 最新观测到的状态
	// +optional
	Status DeploymentStatus
}
```

### 1.1.3 外部版本

而外部版本则不然，需要多个 Go 结构体表示，每个结构体都与该外部版本的 `G(roup)`, `V(ersion)`和 `K(ind)`一一对应（这个对应关系会反映在 `scheme` 中），由于 G 和 K 都不会变化，所以实际上一个 API 的外部类型的 Go 结构体由 V（版本）确定。拿上面 Deployment 实例来说，它的资源定义文件给出的属性 `apiVersion` 指出其版本是 `v1`，会有个 Go 结构体表示该版本下的 Deployment，代码如下所示：

```go
//代码: staging/src/k8s.io/api/apps/v1/types.go
// Deployment enables declarative updates for Pods and ReplicaSets.
type Deployment struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Specification of the desired behavior of the Deployment.
    // 期望的说明
	// +optional
	Spec DeploymentSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

	// Most recently observed status of the Deployment.
    // 最新观测到的状态
	// +optional
	Status DeploymentStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

注意这个 `types.go` 所在的包名是 v1，不难猜测，`v1beta1` 版本的 `Deployment` 基座结构体一定是定义在 `staging/src/k8s.io/api/apps/v1beta1/types.go` 中，的确如此。

## 1.2 API 属性

具有了内外部版本的知识，现在可以更进一步，从代码上查阅一个 API 的属性了。本节依旧以 Development 为例。

### 1.2.1 Deployment 外部类型

先看最贴近用户的外部类型。资源定义文件内容决定于外部类型基座结构体字段。[1.1.3节](#1.1.3 外部版本) 的代码展示了 Deployment 的 v1 版本的基座结构体，其名字也是 Deployment，它定义中所含的字段很有代表性，每个 API 资源类型的结构体基本都具有如下字段：

1. **通过内嵌 TypeMeta 结构体获得其字段**

   根据 Go 结构体嵌套的规则，`TypeMeta` 的字段会直接“传递”给 Deployment 结构体

   ```go
   // 代码: staging\src\k8s.io\apimachinery\pkg\apis\meta\v1\types.go
   // TypeMeta describes an individual object in an API response or request
   // with strings representing the type of the object and its API schema version.
   // Structures that are versioned or persisted should inline TypeMeta.
   //
   // +k8s:deepcopy-gen=false
   type TypeMeta struct {
   	// Kind is a string value representing the REST resource this object represents.
   	// Servers may infer this from the endpoint the client submits requests to.
   	// Cannot be updated.
   	// In CamelCase.
   	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds
   	// +optional
   	Kind string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`
   
   	// APIVersion defines the versioned schema of this representation of an object.
   	// Servers should convert recognized schemas to the latest internal value, and
   	// may reject unrecognized values.
   	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources
   	// +optional
   	APIVersion string `json:"apiVersion,omitempty" protobuf:"bytes,2,opt,name=apiVersion"`
   }
   ```

   于是 APIVersion 和 Kind 两个 TypeMeta 的字段就被放到了 Deployment 结构体上。这两个字段与资源定义文件中 `apiVersion` 和 `kind` 对应。程序运行时 `Deployment` 结构体实例会用这两个字段承接资源文件中的 `apiVersion` 和 `kind` 信息。

2. 通过内嵌 `ObjectMeta` 结构体获得其字段

   类似 `TypeMeta` 结构体的情况，`Deployment` 通过内嵌 `ObjectMeta` 结构体获取了它的所有字段。`ObjectMeta` 的字段均是用来表示一个 `Deployment` 实例自身信息。

   ```go
   // 代码: staging\src\k8s.io\apimachinery\pkg\apis\meta\v1\types.go
   // ObjectMeta 保存所有持久化资源必须包含的元数据。  
   // 下面对每个字段含义逐一说明（已去掉 json / protobuf 等标签）。  
   type ObjectMeta struct {  
   	// Name 是对象在同一命名空间内唯一的名称。  
   	// 创建时通常必填，用于幂等与配置声明；创建后不可修改。  
   	Name string  
   
   	// GenerateName 是服务器在 Name 未指定时生成唯一名称的前缀。  
   	// 返回给客户端的最终名称会在该前缀后附加随机后缀；若生成后的名称冲突将返回 409。  
   	GenerateName string  
   
   	// Namespace 决定名称的作用域。不填写等同于 "default" 命名空间。  
   	// 并非所有资源都需要归属命名空间；集群级资源该字段为空。  
   	Namespace string  
   
   	// SelfLink（已废弃）曾用于存放对象的完整访问路径，现已停止填充。  
   	SelfLink string  
   
   	// UID 是对象在时间与空间上的唯一标识，成功创建后由服务器分配；PUT 不可变。  
   	UID types.UID  
   
   	// ResourceVersion 表示对象的内部版本号，用于并发控制、变更检测及 watch。  
   	// 客户端需视为不透明字符串，原样回传。  
   	ResourceVersion string  
   
   	// Generation 表示期望状态（Spec）的版本号，每次用户修改 Spec 时自动递增。  
   	Generation int64  
   
   	// CreationTimestamp 表示对象在服务器上的创建时间（UTC，RFC3339 格式）。  
   	// 由系统填充，客户端只读；列表对象中为 null。  
   	CreationTimestamp Time  
   
   	// DeletionTimestamp 指定资源将被删除的时间（优雅删除场景）。  
   	// 设置后需等待 Finalizers 清空才真正删除；若为空代表未请求删除。  
   	DeletionTimestamp *Time  
   
   	// DeletionGracePeriodSeconds 与 DeletionTimestamp 配合，表示优雅终止的宽限秒数。  
   	// 仅在 DeletionTimestamp 非空时设置，只允许缩短。  
   	DeletionGracePeriodSeconds *int64  
   
   	// Labels 是键值对，用于对象的分组与选择（label 选择器）。  
   	Labels map[string]string  
   
   	// Annotations 也是键值对，用于存放非查询型的附加信息，由外部工具自由读写。  
   	Annotations map[string]string  
   
   	// OwnerReferences 记录该对象依赖的父对象列表，全部父对象删除后本对象可被垃圾回收。  
   	// 若由控制器管理，其中一个引用的 controller 字段会为 true；同一对象最多一个控制器。  
   	OwnerReferences []OwnerReference  
   
   	// Finalizers 在删除前必须被清空的标记列表，用于实现外部/级联清理逻辑。  
   	// 条目顺序不保证，任何拥有权限的主体都可增删重排。  
   	Finalizers []string  
   
   	// ManagedFields 记录不同「工作流」对对象字段的管理关系，  
   	// 用于服务器端 Apply 等变更合并逻辑。一般用户无需关心。  
   	ManagedFields []ManagedFieldsEntry  
   }  
   ```

   典型的有：

   1. `Name`：API 实例的名称。
   2. `UID`：唯一标识。
   3. `NameSpace`：所隶属的命名空间。
   4. `Annotations`：实例上的注解。注解不同于普通标签，它不会被用于实例查询操作。
   5. `Labels`：实例上的标签。
   6. `Finalizers`：一组字符串构成的数组，当系统删除一个 API 实例时，会要求这个数组为空，如若不为空，则不删除该实例，等待下次控制循环再检查。这为外界影响一个实例的销毁提供途径。

3. Spec

   用户对系统最终状态的期望，它是对接资源定义文件内容的重要字段。每个类型都会有其独特的属性。

   ```go
   // 代码: staging\src\k8s.io\api\apps\v1\types.go
   // DeploymentSpec is the specification of the desired behavior of the Deployment.
   type DeploymentSpec struct {
   	// Number of desired pods. This is a pointer to distinguish between explicit
   	// zero and not specified. Defaults to 1.
   	// +optional
   	Replicas *int32 `json:"replicas,omitempty" protobuf:"varint,1,opt,name=replicas"`
   
   	// Label selector for pods. Existing ReplicaSets whose pods are
   	// selected by this will be the ones affected by this deployment.
   	// It must match the pod template's labels.
   	Selector *metav1.LabelSelector `json:"selector" protobuf:"bytes,2,opt,name=selector"`
   
   	// Template describes the pods that will be created.
   	// The only allowed template.spec.restartPolicy value is "Always".
   	Template v1.PodTemplateSpec `json:"template" protobuf:"bytes,3,opt,name=template"`
   
   	// The deployment strategy to use to replace existing pods with new ones.
   	// +optional
   	// +patchStrategy=retainKeys
   	Strategy DeploymentStrategy `json:"strategy,omitempty" patchStrategy:"retainKeys" protobuf:"bytes,4,opt,name=strategy"`
   
   	// Minimum number of seconds for which a newly created pod should be ready
   	// without any of its container crashing, for it to be considered available.
   	// Defaults to 0 (pod will be considered available as soon as it is ready)
   	// +optional
   	MinReadySeconds int32 `json:"minReadySeconds,omitempty" protobuf:"varint,5,opt,name=minReadySeconds"`
   
   	// The number of old ReplicaSets to retain to allow rollback.
   	// This is a pointer to distinguish between explicit zero and not specified.
   	// Defaults to 10.
   	// +optional
   	RevisionHistoryLimit *int32 `json:"revisionHistoryLimit,omitempty" protobuf:"varint,6,opt,name=revisionHistoryLimit"`
   
   	// Indicates that the deployment is paused.
   	// +optional
   	Paused bool `json:"paused,omitempty" protobuf:"varint,7,opt,name=paused"`
   
   	// The maximum time in seconds for a deployment to make progress before it
   	// is considered to be failed. The deployment controller will continue to
   	// process failed deployments and a condition with a ProgressDeadlineExceeded
   	// reason will be surfaced in the deployment status. Note that progress will
   	// not be estimated during the time a deployment is paused. Defaults to 600s.
   	ProgressDeadlineSeconds *int32 `json:"progressDeadlineSeconds,omitempty" protobuf:"varint,9,opt,name=progressDeadlineSeconds"`
   }
   ```

   Kubernetes 会根据用户期望不断调整内部状态，直到满足用户期望。 用户是通过资源定义文件描述期望的，资源定义文件最终又被加载到 Go 结构体实例中供控制器使用，期望内容正是放到这个 Spec 字段中。

4. Status

   上述 Spec 承载了用户描述的期望状态，实例的当前状态则存放在 Status 字段中。

   ```go
   // 代码: staging\src\k8s.io\api\apps\v1\types.go
   // DeploymentStatus is the most recently observed status of the Deployment.
   type DeploymentStatus struct {
   	// The generation observed by the deployment controller.
   	// +optional
   	ObservedGeneration int64 `json:"observedGeneration,omitempty" protobuf:"varint,1,opt,name=observedGeneration"`
   
   	// Total number of non-terminating pods targeted by this deployment (their labels match the selector).
   	// +optional
   	Replicas int32 `json:"replicas,omitempty" protobuf:"varint,2,opt,name=replicas"`
   
   	// Total number of non-terminating pods targeted by this deployment that have the desired template spec.
   	// +optional
   	UpdatedReplicas int32 `json:"updatedReplicas,omitempty" protobuf:"varint,3,opt,name=updatedReplicas"`
   
   	// Total number of non-terminating pods targeted by this Deployment with a Ready Condition.
   	// +optional
   	ReadyReplicas int32 `json:"readyReplicas,omitempty" protobuf:"varint,7,opt,name=readyReplicas"`
   
   	// Total number of available non-terminating pods (ready for at least minReadySeconds) targeted by this deployment.
   	// +optional
   	AvailableReplicas int32 `json:"availableReplicas,omitempty" protobuf:"varint,4,opt,name=availableReplicas"`
   
   	// Total number of unavailable pods targeted by this deployment. This is the total number of
   	// pods that are still required for the deployment to have 100% available capacity. They may
   	// either be pods that are running but not yet available or pods that still have not been created.
   	// +optional
   	UnavailableReplicas int32 `json:"unavailableReplicas,omitempty" protobuf:"varint,5,opt,name=unavailableReplicas"`
   
   	// Total number of terminating pods targeted by this deployment. Terminating pods have a non-null
   	// .metadata.deletionTimestamp and have not yet reached the Failed or Succeeded .status.phase.
   	//
   	// This is an alpha field. Enable DeploymentReplicaSetTerminatingReplicas to be able to use this field.
   	// +optional
   	TerminatingReplicas *int32 `json:"terminatingReplicas,omitempty" protobuf:"varint,9,opt,name=terminatingReplicas"`
   
   	// Represents the latest available observations of a deployment's current state.
   	// +patchMergeKey=type
   	// +patchStrategy=merge
   	// +listType=map
   	// +listMapKey=type
   	Conditions []DeploymentCondition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,6,rep,name=conditions"`
   
   	// Count of hash collisions for the Deployment. The Deployment controller uses this
   	// field as a collision avoidance mechanism when it needs to create the name for the
   	// newest ReplicaSet.
   	// +optional
   	CollisionCount *int32 `json:"collisionCount,omitempty" protobuf:"varint,8,opt,name=collisionCount"`
   }
   ```

每个 API 结构体都会有名为 `Spec` 和 `Status` 字段，但不同 API 中的 `Spec` 与 `Status` 的 Go 类型则不相同，因为每个资源都会有独特的属性。下面不再逐个介绍其各个字段，它们为 Deployment 特有并不具备一般意义。

**下面来看 `DeploymentSpec` 结构体：**

不难发现，资源定义文件中的 `Spec` 下定义的属性和 `DeploymentSpec` 结构体的字段有着明显的对应关系，例如 `replicas` 对应 `Replicas`，而且这种对应关系已经被结构体属性后面的注解明确标识出来了，来看 Replicas 的注解：

```go
Replicas *int32 `json:"replicas,omitempty" protobuf:"varint,1,opt,name=replicas"`
```

上述注解翻译一下含义是：`Replicas` 这个字段对应 `json` 格式下的 `replicas` 属性，或 `protobuf` 格式下的 `replicas` 属性。用户一般使用 Yaml 文件表述资源定义文件，但在代码级别 `Yaml` 格式的信息会被转为 `Json` 格式，再由 `Json` 直接向 `Go` 结构体实例转换，要知道 `Go` 是天然支持 `Json` 与其数据结构进行相互转换的。

**下面再来看 v1 版本的 `DeploymentStatus` 结构体**

`Status` 属性也类似，`Deployment` 结构体的 `Status` 类型是 `DeploymentStatus`。`Status` 用于 承载一个 `Deployment` 实例的当前状态信息，当用户调用 `kubectl` 的 `describe` 命令查看 API 实例的详细信息时，Status 属性也包含其中。对于 `Deployment` 来说，它的状态信息包括总副本数量，可用/不可用副本的数量，就绪副本数量，条件（conditions）信息。这里“条件”是 一组用于衡量 Deployment 实例是否正常的标准。

### 1.2.2 Deployment 内部类型

再来看内部类型。我们说一个 API 的内部版本始终只有一个结构体，但这个结构体的属性确是不断变化的，随着版本的更新不断而更新，因为一个 API 所具有的属性可能会在不同版本中变更。一般来说，内部版本的结构体和最高版本外部类型的结构体相似度最高， 毕竟废弃已有属性的情况不是很多，更多的是在 API 上添加属性。Deployment 内部版本的 Go 基座结构体如下所示：

```go
// Deployment provides declarative updates for Pods and ReplicaSets.
type Deployment struct {
	metav1.TypeMeta
	// +optional
	metav1.ObjectMeta

	// Specification of the desired behavior of the Deployment.
	// +optional
	Spec DeploymentSpec

	// Most recently observed status of the Deployment.
	// +optional
	Status DeploymentStatus
}
```

这和 v1 版的基座结构体十分类似，最明显的区别是这个结构体的属性没有带注解，因为根本没有把信息从 `Yaml` 向 `Json` 进而再向内部版本结构体实例转化的需求。除此之外有个隐含的不同：字段 Spec 和外部版本字段 Spec 的类型**只是同名，但是不同结构体**；属性 Status 也一样。内部版本 Spec 类型是 DeploymentSpec，定义在 pkg/apis/apps 包下，而外部版本的却不是。

## 1.3 API 的方法与函数

上面讲解了 API 的基座结构体定义代码，接下来介绍与 API 紧密相关的方法与函数。它们有的直接定义在 API 基座结构体上，有的则单独存在。如果参考内置 API，每个 API 必须具有如下几类方法：

### 1.3.1 DeepCopy

API 的内外部版本都需要具有这组方法，它会把当前 API 实例做一次深拷贝得到一个新实例。控制器中，控制循环处理请求队列中的请求时，需从本地缓存取出待处理的实例做一次深拷贝，把改变放在新实例上而不改变老实例，最后用新实例更新缓存和 ETCD，这避免了破坏缓存机制。`Deployment` 的深拷贝方法实现代码如下所示：

```go
// 代码 pkg/apis/apps/zz_generated_deepcopy.go
// DeepCopy is an autogenerated deepcopy function, copying the receiver, creating a new DaemonSetSpec.
func (in *DaemonSetSpec) DeepCopy() *DaemonSetSpec {
	if in == nil {
		return nil
	}
	out := new(DaemonSetSpec)
	in.DeepCopyInto(out)
	return out
}

// DeepCopyInto is an autogenerated deepcopy function, copying the receiver, writing into out. in must be non-nil.
func (in *DaemonSetStatus) DeepCopyInto(out *DaemonSetStatus) {
	*out = *in
	if in.CollisionCount != nil {
		in, out := &in.CollisionCount, &out.CollisionCount
		*out = new(int32)
		**out = **in
	}
	if in.Conditions != nil {
		in, out := &in.Conditions, &out.Conditions
		*out = make([]DaemonSetCondition, len(*in))
		for i := range *in {
			(*in)[i].DeepCopyInto(&(*out)[i])
		}
	}
	return
}

// DeepCopy is an autogenerated deepcopy function, copying the receiver, creating a new DaemonSetStatus.
func (in *DaemonSetStatus) DeepCopy() *DaemonSetStatus {
	if in == nil {
		return nil
	}
	out := new(DaemonSetStatus)
	in.DeepCopyInto(out)
	return out
}
```

上述三个方法都是提供在 `Deployment` 结构体指针上，即 `*Deployment` 上：

1. `DeepCopyInto()` 方法：把当前实例复制到指定实例中；
2. `DeepCopy()` 方法：则新创建实例，然后调用 `DeepCopyInto()`；
3. `DeepCopyObject()` 方法和 `DeepCopy` 相比，只是返回值类型不同。

### 1.3.2 Convert

以修改 API 资源为例，系统接收到的用户请求是针对一个 API 资源的，目标在请求中以某一个外部版本表示，而系统内代码是针对内部版本进行编写的，这就需要将请求中的资源从外部版本转化为内部版本；反之，当系统要给客户端响应时，又需要把内部版本转化为客户端需要的外部版本。这组 Converter 函数负责这些工作。Deployment 的 v1 版与内部版本互相转换的实现代码如下：

```go
// Convert_apps_DeploymentSpec_To_v1_DeploymentSpec is defined here, because public
// conversion is not auto-generated due to existing warnings.
func Convert_apps_DeploymentSpec_To_v1_DeploymentSpec(in *apps.DeploymentSpec, out *appsv1.DeploymentSpec, s conversion.Scope) error {
	if err := autoConvert_apps_DeploymentSpec_To_v1_DeploymentSpec(in, out, s); err != nil {
		return err
	}
	return nil
}

func Convert_v1_Deployment_To_apps_Deployment(in *appsv1.Deployment, out *apps.Deployment, s conversion.Scope) error {
	if err := autoConvert_v1_Deployment_To_apps_Deployment(in, out, s); err != nil {
		return err
	}

	// Copy annotation to deprecated rollbackTo field for roundtrip
	// TODO: remove this conversion after we delete extensions/v1beta1 and apps/v1beta1 Deployment
	if revision := in.Annotations[appsv1.DeprecatedRollbackTo]; revision != "" {
		if revision64, err := strconv.ParseInt(revision, 10, 64); err != nil {
			return fmt.Errorf("failed to parse annotation[%s]=%s as int64: %v", appsv1.DeprecatedRollbackTo, revision, err)
		} else {
			out.Spec.RollbackTo = new(apps.RollbackConfig)
			out.Spec.RollbackTo.Revision = revision64
		}
		out.Annotations = deepCopyStringMap(out.Annotations)
		delete(out.Annotations, appsv1.DeprecatedRollbackTo)
	} else {
		out.Spec.RollbackTo = nil
	}

	return nil
}

func Convert_apps_Deployment_To_v1_Deployment(in *apps.Deployment, out *appsv1.Deployment, s conversion.Scope) error {
	if err := autoConvert_apps_Deployment_To_v1_Deployment(in, out, s); err != nil {
		return err
	}

	out.Annotations = deepCopyStringMap(out.Annotations) // deep copy because we modify it below

	// Copy deprecated rollbackTo field to annotation for roundtrip
	// TODO: remove this conversion after we delete extensions/v1beta1 and apps/v1beta1 Deployment
	if in.Spec.RollbackTo != nil {
		if out.Annotations == nil {
			out.Annotations = make(map[string]string)
		}
		out.Annotations[appsv1.DeprecatedRollbackTo] = strconv.FormatInt(in.Spec.RollbackTo.Revision, 10)
	} else {
		delete(out.Annotations, appsv1.DeprecatedRollbackTo)
	}

	return nil
}
```

上述代码包含两个函数，代表两个方向的转换，内容虽多但每个方法都只做两件事情： 第 一 件 ， 调 用 函 数 `autoConvert_v1_Deployment_To_apps_Deployment()` 或 函 数 `autoConvert_apps_Deployment_To_v1_Deployment()`，执行实际的转换；第二件，把转换结果做一些调整。被调用的两个函数完成实际转换操作，代码如下所示：

```go
// 代码 pkg/apis/apps/v1/zz_generated_conversion.go
func autoConvert_v1_Deployment_To_apps_Deployment(in *appsv1.Deployment, out *apps.Deployment, s conversion.Scope) error {
	out.ObjectMeta = in.ObjectMeta
	if err := Convert_v1_DeploymentSpec_To_apps_DeploymentSpec(&in.Spec, &out.Spec, s); err != nil {
		return err
	}
	if err := Convert_v1_DeploymentStatus_To_apps_DeploymentStatus(&in.Status, &out.Status, s); err != nil {
		return err
	}
	return nil
}

func autoConvert_apps_Deployment_To_v1_Deployment(in *apps.Deployment, out *appsv1.Deployment, s conversion.Scope) error {
	out.ObjectMeta = in.ObjectMeta
	if err := Convert_apps_DeploymentSpec_To_v1_DeploymentSpec(&in.Spec, &out.Spec, s); err != nil {
		return err
	}
	if err := Convert_apps_DeploymentStatus_To_v1_DeploymentStatus(&in.Status, &out.Status, s); err != nil {
		return err
	}
	return nil
}
```

上述两个函数继续调用 Deployment 基座结构体字段对应的转换函数，逐一转换 Spec 和 Status 子资源，这样逐层递进地完成全部信息的转换。

### 1.3.3 Default

顾名思义，默认值设置函数为新创建出的 API 实例赋默认值。依然以 Deployment 为例， 它 v1 版本的 Default 函数实现代码如下所示：

```go
func SetObjectDefaults_Deployment(in *appsv1.Deployment) {
	SetDefaults_Deployment(in)
	apiscorev1.SetDefaults_PodSpec(&in.Spec.Template.Spec)
	for i := range in.Spec.Template.Spec.Volumes {
		a := &in.Spec.Template.Spec.Volumes[i]
		apiscorev1.SetDefaults_Volume(a)
		if a.VolumeSource.HostPath != nil {
			apiscorev1.SetDefaults_HostPathVolumeSource(a.VolumeSource.HostPath)
		}
		if a.VolumeSource.Secret != nil {
			apiscorev1.SetDefaults_SecretVolumeSource(a.VolumeSource.Secret)
		}
		if a.VolumeSource.ISCSI != nil {
			if a.VolumeSource.ISCSI.ISCSIInterface == "" {
				a.VolumeSource.ISCSI.ISCSIInterface = "default"
			}
		}
		if a.VolumeSource.RBD != nil {
			if a.VolumeSource.RBD.RBDPool == "" {
				a.VolumeSource.RBD.RBDPool = "rbd"
			}
			if a.VolumeSource.RBD.RadosUser == "" {
				a.VolumeSource.RBD.RadosUser = "admin"
			}
			if a.VolumeSource.RBD.Keyring == "" {
				a.VolumeSource.RBD.Keyring = "/etc/ceph/keyring"
			}
		}
        ...
}
```

上述方法非常长，限于篇幅这里只截取最开始的一部分。它的第一行先调用了同包下的 `SetDefaults_Deployment` 方法，该方法是开发人员手动编写的，对一些属性的默认值进行人为设定。

以上就是各个 API 基座结构体相关的重要方法、函数以及它们的实现。观察全部内置 API，这些方法总的代码量还是很客观的，手工写起来工作量巨大而且容易出错。仔细对比各个 API 的同类方法的实现，发现内容大同小异，十分有规律，这就给计算机自动生成部分代码提供了条件。

以`zz_generated_`开头，所有这样的文件，其内容均是代码生成工具的产物，开发人员不用（也不能）进行修改就可以使用。只有当生成代码不能满足需要时，才需手动加入自有逻辑，`conversion.go` 包含的就是这类代码。

## 1.4 API 定义与实现的约定

为了统一不同 API 的定义方式，Kubernetes 项目制定了如下规约：

### 1.4.1 对象类型 API 的要求

对象类型 API 的实例会独立存储于 ETCD，它们也是系统中 API 集合的主体。在定义其 Go 基座结构体时，必须具有如下字段：

1. **metadata 字段**

   `metadata` 字段用于表示 API 实例元数据，其下必须有子字段 `namespace`、`name` 和 `uid`。`namespace` 提供了一种资源隔离的软机制，是 Kubernetes 体系内实现多租户的基础。具有 `namespace` 属性并不代表一定要给 API 实例的 `namespace` 属性赋值，有些资源就是跨 `namespace` 的；`name` 属性是必须的，每个 API 实例在一个 `namespace` 内不能同该 API 种类 的其它实例同名，但不同 API 种类的实例之间同名是允许的；`uid` 属性由系统生成并赋予 API 实例，将作为该 API 实例在集群内的唯一标识。

   `metadata` 下还应该具有如下子属性：`resourceVersion`、 `generation`、 `creationTimestamp`、 `deletionTimestamp`、 `labels` 和 `annotations`。前三个属性是系统维护的，用户不必处理；`labels` 和 `annotations` 是系统和用户共同维护的，在定义 API 资源定义文件时，用户可以给出期望 的 `labels` 和 `annotations`；而系统也会根据处理需要额外添加。Labels 会被用于资源的查询与选取，而 `annotations` 则被程序主要用于内部处理。

2. **`Spec` 和 `Status` 字段**

   Kubernetes API 需要分离对期望状态的描述和当前状态的描述：期望状态存放于 Spec 中，而当前状态存放在 Status 中。Spec 的部分由人和系统共同维护，用户会在资源定义文件中给出自己的期望，而系统会在请求接收与处理过程中进行补全甚至修改。而 Status 则显示该 API 实例有关的最新系统状态，它的信息可能同 API 实例一起存在 ETCD 中，也可能在需要该信息时进行即时抓取以确保最新状态。API Server 的代码逻辑确保用户不能够直接更新资源的 Status 内容，例如通过 PUT 操作不能更改目标资源实例的 Status 内容。

   `status` 中的 `Conditions` 被用来简化消费端对当前对象的状态理解。例如 `Deployment` 的 `Available Condition` 实际上综合考虑了 `readyReplicas` 和 `replicas` 字段的内容而给出的。在定义 `status` 结构体时，`Conditions` 应被作为其顶层子字段。

### 1.4.2 可选属性和必备属性

定义 API 的基座结构体时，用代码注解指明可选属性（指该字段上可以没有值）和必备属性（该属性上必须有值）是必要的，这将指导代码生成正常工作。

**定义可选字段时应保持：**

1. 该字段的 Go 类型应该是指针类型；
2. Server 处理 POST 和 PUT 请求时，应不依赖可选字段值；
3. 定义基座结构体时，可选字段的字段标签应该含有 `omitempty`，这样 `Go` 和 `Json`、 `Protobuf` 做数据转换时也允许该字段为空。

**而处理必备字段时则正相反：**

1. 该字段的 Go 类型非指针；
2. Server 不接受请求中缺失该属性的值；
3. 该字段的字段标签不能包含`omitempty`。

定义中设置可选和必备比较简单，只要在属性定义上方加 `// +optional` 或 `// +required` 注解。

```go
type StatefulSet struct {
	metav1.TypeMeta
	// +optional
	metav1.ObjectMeta
	// 定义本 SS 中 Pods 的期望标识.
	// +optional
	Spec StatefulSetSpec
	// 本 SS 中 Pods 的当前状态。这个字段中的信息相对特定时间窗口有可能是过时的
	// +optional
	Status StatefulSetStatus
}
```

### 1.4.3 API 实例的默认值设定

Kubernetes 希望 API 编写者明确为 API 的各个字段设置默认值，不欢迎笼统地规定`...没有提及的字段具有某个默认的值或行为` 。明确设定默认值的好处包括：

1. 默认值可以随着版本而演化，在新版本启用新默认值不影响系统对老版本对象的默认值设置。
2. 系统拿到的 API 实例的属性值是用户明确指定的，而那些由于各种原因没有值的属性就真的是系统可以自行决定的。系统不必担心例外，这会简化代码逻辑。

默认值特别适合那些逻辑上必须并且绝大部分情况其取值都固定的字段。设定默认值主要有两种手段，其一是静态指定，其二是通过准入控制机制动态设定。

1. **静态指定**

   每个版本都硬编码这些默认值，只要在 API 属性的定义之上加 `// +default=` 注解就可以。这样的硬编码也可以有简单逻辑：根据其它一些字段来决定出当前属性的值。只是这样的话需要程序处理额外的复杂性，因为当更新了被依赖属性时，同时需要更新当前属性。后续将讲解的 Defaulter 代码生成器简化了静态指定默认值的编码工作。

2. **通过准入控制机制设定默认值**

   静态指定默认值是比较死板的。举个例子，`PersistentVolumeClaim（PVC）` 的 `Storage Class` 属性逻辑上必须要一个 `Storage Class API` 实例，且绝大多数 PVC 的创建者会选用集群管理员做的全局设定，管理员指定什么这个 PVC 实例就用什么。这时静态指定默认值就无法胜任了，因为管理员的设定并非固定或有章可循。这时准入控制机制就派上用场。该机制是系统在处理用户请求时调用的一些插件，每个插件都实现对请求所含 API 实例信息的修改或校验，可以通过修改接口进行默认值的设定。

### 1.4.4 并发处理

API 的编写者在编写控制器时需要牢记如下事实：系统中的 API 实例可能被多个请求并发触达，Kubernetes 将采用**乐观锁**的机制协调并发处理。 API 的 `metadata` 字 段，看到它有一个子字段 `resourceVersion`，它由系统维护，每一个到达 API Server 的**请求都会带有目标资源的一个版本信息**，标明这次操作是基于哪个版本进行的，Server 在预处理请求时会进行一次检查，如果当前该资源的 `resourceVersion` 高于请求中指明的，则拒绝请求， 返回 `HTTP 409`。对于创建操作，不涉及并发问题所以不必指明 `resourceVersion`；而对于更改操作，`resourceVersion` 是需要的，客户端可以从前序交互中获得 API 实例信息，包括 `resourceVersion`。如果多个修改请求同时通过预检查，在修改数据库时还是可能发生版本冲突，开发者需要合理处理。

# 2. 内置 API

整个 API Server 都是在围绕 API 运作。一方面用户的需求全部被表述为 API 实例，另一方面 API Server 自己也以 API 实例来存储内部信息，它的运作机制也被设计成依赖 API 实例。那么 API 种类是否丰富，是否足够满足方方面面的需求就很重要了。

API Server 已经构建好了众多的 API，称为内置 API，绝大部分的功能需求都能被这些 API 种类所满足。

在众多的 API 组中，有两个特别巨大、也就是其所包含的 API 种类非常多组：一个是 core 组， 一个是 apps 组。core 组在下一小节单独讲解，apps 组是用户使用最多的组，它包含在集群中部署应用程序时使用的各种 API，例如 Deployment，按照如下指引定位内置 API 的代码：

1. **内部版本的定义**。位于包 `pkg/apis`，每个 API 组在该包下有一个子包，通常内部版本定义于该子包下的 `types.go` 文件。API 的内部版本的代码有可能会随着新 Kubernetes 版 本的发布而被调整，从而被增强，这和外部版本不同，已发布的外部版本的代码一般不大变， 除非 bug 修复。
2. **外部版本的定义**。位于包 `staging/src/k8s.io/api`，内置 API 的外部版本被抽取到 `staging` 中，作为单独代码库发布，便于在众多客户端代码中复用。每一个 API 组在该包下有一个子包；每一个版本会在该子包下又有一个子包，外部版本定义就位于此。
3. **控制器逻辑**。位于包 `pkg/controller`。除了 API 的定义，每个 API 的业务逻辑包含在各自控制器的控制循环中，每个 API 组对应一个子包，可以在其中找到相关内置 API 的 控制器代码。

控制器代码揭示了系统根据各个 API 实例执行操作的逻辑，很值的阅读。每个控制器的代码框架都十分类似，一旦框架掌握，将一通百通。这个后续会进行讲解。

# 3. 核心 API

核心 API 出现最早，用例已经遍布天下，修改其使用方式（API 接口，参数等）影响巨大难以被接受，这造成了核心 API 一些与众不同之处。核心 API 与其它内置 API 具有如下不同：

1. 组的名称很特殊。其它内置 API 组的名称大多以 `k8s.io` 或 `kubernetes.io` 结尾，即使不带这后缀也会有个名字，但核心 API 组特立独行，它的名字是`“”`（空字符串），没有名字就是它的名字！

2. 资源的 URI 构成模式不同。普通内置 API 的资源 URI 具有这个模式：

   ```go
   /apis/<API 组>/<版本>/namespaces/<命名空间>/<资源>
   ```

   而核心 API 的资源 URI 模式稍有不同，是这样的：

   ```go
   /api/<版本>/namespaces/<命名空间>/<资源>
   ```

   区别在于：其它 API 以 `apis` 为前缀，核心 API 是 `api`；其它 API 包含组名称，核心组没有。

3. 资源定义文件中的 apiVersion 内容不同。由于组名是空字符串，所以在定义核心 API 资源时，apiVersion 这一属性直接就是 `v1`，不带 API 组的。

# 4. 代码生成

API 实例的深拷贝（DeepCopy）、内外部版本的转换（Converter）和默认值的设定（Defaulter）代码逻辑在各个 API 之间基本相同，它们位于名称以 `zz_generated` 为前缀的文件，也都是代码生成的产物，在 Kubernetes 中代码生成主要服务如下场景：

1. 为 API 基座结构体添加深拷贝、版本转换和默认值设置函数或方法。
2. 为 API 生成 client-go 代码。client-go 包单独发布，是客户端和 API Server 交互的基础库，内置 API 的外部版本定义也会包含在其中。
   * 首先生成 Clientset。为客户端程序提供一组操作 API 实例的编程接口，它们负责从 API Server 即时获取目标 API 实例，当然创建修改等也没问题。所有交互细节客户端不必关心。
   * 然后生成 Informer。客户端从 API Server 获知 API 实例变更的高效机制，API Server 允许客户端对某个 API 种类状态变化进行 WATCH 操作，客户端利用 Informer 来对接 API Server 进行状态观测。Informer 内置缓存机制，它的存在将大大降低 API Server 的负担。 Informer 存在于以上的 `client-go` 包中，主要用于控制器的代码。
   * 最后会生成 Lister。作用类似 Informer，可以从 API Server 获取某一类资源的实例列表。 但内部没有缓存，比 Informer 简单，适用简单场景。同样地，Lister 也存在于 `client-go` 包中。
3. 为 API 的注册生成代码。API Server 的注册表机制要求每个 API 组都注册自己的 API 种类到注册表中，这部分代码也可以自动生成。
4. 为 API 种类生成 OpenAPI 的服务定义文件。每个 API 资源都会以端点形式暴露于 API Server，其格式符合 OpenAPI 规范，这就需要服务定义。

## 4.1 代码生成工作原理

Kubernetes 的代码生成基于库 `code-generator`。这是一个 Kubernetes 项目下的子库，算是项目的一个副产品。而 `code-generator` 的代码生成能力最终源于另外一个基础库：`gengo`， 而 `gengo` 也是 Kubernetes 项目的一个副产品，最初就是为了解决 Kubernetes 中大量重复代码编写的问题而创立的。`gengo` 项目可以服务于 Kubernetes 之外的开发——理论上说所有 Go 项目都在它的服务范围之内。而 `code-generator` 基于 `gengo`，为 Kubernetes API Server 的开发、 聚合 Server 的开发以及 `CustomResourceDefinition` 的开发进行功能定制。这里我们只关注 `code-generator` 库而不会深入 `gengo` 库。`code-generator` 源码位 于 `staging/src/k8s.io/code-generator` 包中。

`code-generator` 为 Kubernetes 制定了一系列**注解**，**注解需要以代码注释的方式放置在目标代码的上方**，`code-generator` 在运行时对目标源代码进行扫描，找到这些注解进而进行相 应的代码生成，最后把生成结果输出到目标文件夹。一个注解是具有如下格式的注释语句：

```go
// +<tag name>=<value>
// +<tag name>
```

例如：`// +groupName=admission.k8s.io`，`// +k8s:deepcopy-gen=package`. 注解又有全局和局部之分。

### 4.1.1 全局注解

全局注解放置在包级别，对整个包起作用的注解。它们被放在一个包的定义语句——也就是 `package` 语句之上。Go 推荐将包定义于 `doc.go` 文件中，这便于为该包提供注释文档等。 Kubernetes 遵从了这一建议，所以在 Kubernetes 源码中的各个 `doc.go` 文件内找到这些全局注解。举例来说，每个 API 种类的基座结构体都需要实现 `runtime.Object` 接口，只需在包定义语句上添加如下全局注解就可以确保这一点：

```go
// 代码 4-11 pkg/apis/apps/doc.go
// +k8s:deepcopy-gen=package 	//要点①

package apps // import "k8s.io/kubernetes/pkg/apis/apps"
```

有了要点①处的注解，对于 apps 内的每一个类型定义，`code-generator` 都会去生成 DeepCopy 系列方法，同时 `code-generator` 也提供了其它注解从包中剔除不需要生成这系列方法的类型。

主要的全局注解以及它们的作用见下表。

| 序号 | 标签                                            | 作用                                                         |
| ---- | ----------------------------------------------- | ------------------------------------------------------------ |
| 1    | `// +k8s:deepcopy-gen=<value>`                  | 为内部版本生成 Deepcopy 系列方法。作为全局注解使用时，value 可以是"package"（为包内所有类型都生成 DeepCopy 系列方法）或"false"（不生成） |
| 2    | `// +k8s:conversion-gen=<value>`                | 指导生成 API 内外部版本的转换代码。value 为 false 时，不生成；value 为一个包的路径时，代表内部版本的类型定义所在的包 |
| 3    | `// +k8s:conversion-gen-external-types=<value>` | 同样指导生成 API 内外部版本的转化代码。它的 value 是外部版本的类型定义所在的包，一般等同于外部版本的 types.go 文件所在的目录；这个注解可以省略，默认就在当前 doc.go 文件所在的包 |
| 4    | `// +k8s:defaulter-gen=<value>`                 | 当这个注解出现在 package 之上时，value 将代表一个字段名，例如 TypeMeta：如果一个结构体内包含以其为名的一个字段，则为这个结构体生成默认值生成器 |
| 5    | `// +groupName=<value>`                         | 指定 API 的组名，组名在生成 Lister 和 Informer 代码时会用到  |
| 6    | `// +k8s:openapi-gen=true`                      | 为当前包生成 OpenAPI 服务定义文件                            |

### 4.1.2 局部注解

局部注解放置在**类型/字段**定义处，一般是在定义内外部类型的 `types.go` 文件中。局部注解的作用域只限于其下方的一个**类型/字段**，影响针对它们的代码生成。它的主要作用是让目标类型/字段摆脱全局注解的设置，所以我们会看到大部分局部标签与全局标签同名。



由于 `code-generator` 的目标场景限于 Kubernetes 生态系统，比较小众化，可以获取的使用文档很有限，接下来对重要的代码生成器使用方式进行讲解，这对进行聚合 API Server 的开发很有帮助。

### 4.1.3 代码生成器

Conversion 生成器用于生成 API 内外部版本的转换代码。生成的代码可以把一个以内部版本数据结构表示的资源实例转换为外部版本数据结构的实例，以及反向转换。为了完成这项任务，Conversion 代码生成器需要三个输入信息：

1. 一组包含内部版本类型定义的包；
2. 一个包含目标外部版本类型定义的包；
3. 生成代码的存放地，也是一个包。

Conversion 如何得到运行需要的三个输入参数呢？依靠注解。

1. 对于第一个输入参数——内部版本定义所在包，通过如下全局注解来给出：

   ```go
   // +k8s:conversion-gen=<内部版本类型定义的导入路径>
   ```

   除此之外，代码生成程序的命令行参数 `base-peer-dirs` 和 `extra-peer-dirs` 指定的相对路径也会被扫描寻找内部版本。

2. 对于第二个输入参数——外部版本定义所在包，同样通过全局注解给出，形式如下：

   ```go
   // +k8s:conversion-gen-external-types=<外部版本类型定义的 import 路径
   ```

   这个标签是可以省略的，默认外部类型定义在第一个标签所在的包。如果希望排除某些类型，可以在其定义之上加如下标签：

   ```go
   // +k8s:conversion-gen=false
   
   ```

3. 第三个参数——生成代码的目标包，需要通过命令行参数给定，默认放置在外部版本定义所在包。

Conversion 代码生成器在运行时会比较参数指定的各包内发现的内部版本类型名称和外部版本类型名称，**为名称一致的内外部类型生成命名**格式如下的两个转换函数：

```go
autoConvert_<pkg1>_<type>_To_<pkg2>_<type>
```

这两个函数分别做两个方向的转换：由内向外和由外向内各一个。生成工具会递归地完成两个类型之间信息的转化：如果源和目标的类型是简单数据类型，不是结构体，则比较二者类型是否匹配，匹配则生成信息 copy 代码，否则生成失败；如果二者类型为复合类型如结构体，则过程如下图所示。

![image-20251120025501760](http://images.liangning7.cn/typora/202511200255135.png)

与 `autoConvert` 函数“配套”，工具还生成了两个名称以“Convert”为前缀的函数，它们的命名模式为：

```go
Convert_<pkg1>_<type>_To_<pkg2>_<type>
```

这两个函数很有用处：开发人员只要在运行代码生成器前，在目标包下手工**创建同名的 Convert 方法，就可以阻止代码生成器生成它们**，这使得开发人员可以**注入自有转换逻辑**。 很多时候这是必要的，因为很多时候代码生成器并不能正确处理两个复杂类型的转换工作。

除了上述 `autoConvert` 和 `Convert` 方法，代码生成器还会生成把 `Convert` 方法向注册表 （`Scheme`）注册的方法，名称为 `RegisterConversions()`。注册表需要这个信息，当系统需要进行转换时才能从注册表中找到正确的方法进行调用。

为了方便项目人员工作，Kubernetes 工程下的 `hack/update-codegen.sh` 文件中定义了如何调用各个代码生成器的 shell 脚本，这给如何调用代码生成器提供参考。

```bash
# 代码: hack/update-codegen.sh
    conversion-gen \
        -v "${KUBE_VERBOSE}" \
        --go-header-file "${BOILERPLATE_FILENAME}" \
        --output-file "${output_file}" \
        $(printf -- " --extra-peer-dirs %s" "${extra_peer_pkgs[@]}") \
        "${tag_pkgs[@]}" \
        "$@"

    if [[ "${DBG_CODEGEN}" == 1 ]]; then
        kube::log::status "Generated conversion code"
    fi
```

### 4.1.4 DeepCopy 生成器

Go 语言由于可以将一个类型的定义和以该类型为接收器的方法定义可以处于不同源文件中。这和 Java 语言不同，灵活性更高。DeepCopy 生成器利用了这个特性。这个生成器可以为指定的包内的所有类型生成一系列深拷贝方法，当然也可以单独为某一个类型生成，只要在合适的地方加注解即可。

在代码生成过程中，如果目标类型已经具有这一系列方法，则生成的代码直接调用它； 没有时，生成器则试着生成基于直接赋值的 copy 实现；如果直接赋值不能做到深拷贝，则生成器按照自己的逻辑给出一个实现。生成的代码会被放在目标类型所在的包下。

启用 DeepCopy 生成器比较简单，只需在包的 package 语句上方加如下标签：

```go
// +k8s:deepcopy=package
```

生成器会默认为这个包内所有类型生成 DeepCopy 系列方法；可以在某个具体类型定义上方加如下注解来将其排除在外：

```go
// +k8s:deepcopy=false
```

类似地，没有在包级别启用该生成器时，可以通过在某个具体类型定义的上方加如下注解，来为该类型生成 DeepCopy：

```go
// +k8s:deepcopy=true
```

Kubernetes 源码中也常常见到如下注解，它的作用是为被修饰的类型生成一个名为 `DeepCopy<接口名>()` 的方法，这里接口名就是注解中等号后所指定的接口名称。

```go
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
```

该方法的返回值类型将是注解所指定的接口类型。标签中可以指定多个类型，用逗号分隔开，这时生成器会为每个接口生成一个如上深拷贝方法。

内部实现如下逻辑：先直接调用 `DeepCopy()`，然后把得到的结果直接返回。`DeepCopy()` 方法得到的副本类型与源实例的类型相同，那么该副本能被当作目标接口类型返回的前提是：源实例类型实现了返回值接口，所以**注解上的接口**并不能随意指定，**一定要确保被修饰的类型实现了它**。

DeepCopy 生成器的生成结果是一个单独文件，所有生成的深拷贝方法均在其中。这一过程中没有改动类型定义文件，从而不必担心其破坏原来的代码。这很酷。

### 4.1.5 Defaulter 生成器

当一个 API 的外部类型对象被创建出来后，其各个属性——也就是类型结构体的字段，还都是初始值——各自类型的零值，可以为它们赋予默认值，这就是 `Defaulter` 系列方法的作用。相对于前两种代码生成器，Defaulter 生成器稍有不同。

首先，需要**使用全局注解**，指出目标包中哪些结构体需要设置默认值，例如在 Kubernetes 工程中常见如下标签：

```go
// +k8s:defaulter-gen=TypeMeta
```

上述注解告诉 `Defaulter` 生成器：请扫描当前包，找到那些有字段 `TypeMeta` 的结构体，为该结构体生成填充默认值的方法。就这个例子而言，API 基座结构体都会内嵌结构体 `metav1.TypeMeta`，那么它们都会有以 `TypeMeta` 为名字的字段，于是当前包下所有 API 的基座结构体都将是 `Defaulter` 生成器服务的对象。除了使用全局注解圈定候选结构体，还可以直接在候选结构体上设置这个注解：

```go
// +k8s:defaulter-gen=true
```

然后，需要逐个编写 `SetDefaults_<候选结构体类型名>` 的函数，在其中对期望设置默认值的字段编写值填充代码。这部分代码一定是人工编写的，不然机器无法知道需要用什么作为那些字段的默认值。Defaulter 代码生成器运行时，将会逐个审查候选结构体，看看包中能否找到 `SetDefaults_<结构体名>()`函数，能则为它生成一个 `SetObjectDefaults_<结构体名>() `方法。这个方法：

1. 先去调用找到的 `SetDefaults_<结构体名>()` 函数。
2. 接下来生成器检查目标结构体所有字段的类型（如果该字段也是结构体，则逐层递归检验并处理之），如果某个子孙字段的类型存在一个 `SetDefaults_<类型名>()` 函数与之对应，就生成代码去调用该函数从而完成对该子孙字段的默认值设定。

以 `v1 apps/Deployment` 资源类型为例， 开发人员手为其手工编写了`SetDefault_Deployment()` 函数，其中为 `Spec.Strategy`, `Spec.Replicas` 等几个子孙字段设置了默认值。函数代码如下所示：

```go
// SetDefaults_Deployment sets additional defaults compared to its counterpart
// in extensions. These addons are:
// - MaxUnavailable during rolling update set to 25% (1 in extensions)
// - MaxSurge value during rolling update set to 25% (1 in extensions)
// - RevisionHistoryLimit set to 10 (not set in extensions)
// - ProgressDeadlineSeconds set to 600s (not set in extensions)
func SetDefaults_Deployment(obj *appsv1.Deployment) {
	// Set DeploymentSpec.Replicas to 1 if it is not set.
	if obj.Spec.Replicas == nil {
		obj.Spec.Replicas = new(int32)
		*obj.Spec.Replicas = 1
	}
	strategy := &obj.Spec.Strategy
	// Set default DeploymentStrategyType as RollingUpdate.
	if strategy.Type == "" {
		strategy.Type = appsv1.RollingUpdateDeploymentStrategyType
	}
	if strategy.Type == appsv1.RollingUpdateDeploymentStrategyType {
		if strategy.RollingUpdate == nil {
			rollingUpdate := appsv1.RollingUpdateDeployment{}
			strategy.RollingUpdate = &rollingUpdate
		}
		if strategy.RollingUpdate.MaxUnavailable == nil {
			// Set default MaxUnavailable as 25% by default.
			maxUnavailable := intstr.FromString("25%")
			strategy.RollingUpdate.MaxUnavailable = &maxUnavailable
		}
		if strategy.RollingUpdate.MaxSurge == nil {
			// Set default MaxSurge as 25% by default.
			maxSurge := intstr.FromString("25%")
			strategy.RollingUpdate.MaxSurge = &maxSurge
		}
	}
	if obj.Spec.RevisionHistoryLimit == nil {
		obj.Spec.RevisionHistoryLimit = new(int32)
		*obj.Spec.RevisionHistoryLimit = 10
	}
	if obj.Spec.ProgressDeadlineSeconds == nil {
		obj.Spec.ProgressDeadlineSeconds = new(int32)
		*obj.Spec.ProgressDeadlineSeconds = 600
	}
}
```

上述函数只为 Deployment 基座结构体的部分字段设置了默认值，但除此之外还有大量其它子孙字段，`Defaulter` 生成器都会去检查它们的类型，最终发现有如下后代字段（`Volumes`，`InitContainers` ， `Containers` ， `EphemeralContainers` ） 或其后代字段的类型具有相应的 `SetDefaults_<类型名>` 函数，于是生成器会生成调用这些函数的代码。以下代码每个 for 循环 中都是对一个子孙字段 `SetDefaults` 函数的调用，为了节省篇幅，这里省略 for 循环体内的代码。

```go
// 代码 4-14 pkg/apis/apps/v1/zz_generated.defaults.go
func SetObjectDefaults_Deployment(in *v1.Deployment) {
	SetDefaults_Deployment(in)
	corev1.SetDefaults_PodSpec(&in.Spec.Template.Spec)
	
	for i := range in.Spec.Template.Spec.Volumes {
		// ...
	}
	
	for i := range in.Spec.Template.Spec.InitContainers {
		// ...
	}
	
	for i := range in.Spec.Template.Spec.Containers {
		// ...
	}
	
	for i := range in.Spec.Template.Spec.EphemeralContainers {
		// ...
	}
	corev1.SetDefaults_ResourceList(&in.Spec.Template.Spec.Overhead)
}
```

### 4.1.6 client-go 代码生成器

为了理解 `client-go` 的作用，先看一下客户端与 API Server 的交互过程，这一过程如下图所示。API Server 通过端点对外暴露其内的 API 资源，客户端利用这些端点对资源进行 CRUD 等操作。一个 Web Server 端点接收的请求和发送的响应都是格式化的内容，对于 API Server，是以 `Json` 或 `Protobuf` 表述的消息。以 `Json` 为例，当用 HTTP GET 向一个资源的端点发出请求读取一个 API 资源时，Server 内部会把该资源的信息从 ETCD 读进内存，以 Go 结构体实例表示，然后通过 Go 的序列化机制把该实例转成 Json 字符串，发回给客户端。

![image-20251120183002854](http://images.liangning7.cn/typora/202511201830190.png)

客户端程序拿到响应结果后是无法直接处理的，需要再转换为客户端所用编程语言的数据结构，如果是 Go 编写的客户端，那就再转换回 Go 结构体实例。有如下问题：

1. 客户端代码需要知道该资源的 Go 基座结构体定义，否则转换无从谈起。这可以通过 Kubernetes 的另一个库——`staging/src/k8s.io/api` 来获得；
2. 从返回结果向 Go 结构体实例的转换以及连带的处理比较繁琐，客户端程序员可以自己做但难保不出错，而这正是 client-go 可以解决的问题。

client-go 按照 API 组和版本，将操作内置 API 的接口组织到一个 client 内，可通过它在 Go 程序内直接这些资源。client 屏蔽了格式转换以及连带操作的复杂性。例如 apps 组有 v1beta1, v1beta2, v1 三个版本，client-go 包含 3 个 client 与之对应。clientset 是在 client 基础上定义出来的，它聚合了某一个 Kubernetes 版本所包含的所有 client，clientset 的初始化需要 API Server 连接信息，从而对接 API Server 完成用户需要的 CRUD 操作。

client-go 没有止步于此，在优化与 API Server 交互上做了更多工作。

* 首先，提供 API 资源的 Lister，它的作用是从 Server 批量获取资源；
* 其次，考虑到众多控制器程序利用 watch 操作从 API Server 获取资源即时信息的需求非常大，而 watch 会造成众多长时连接，处理不妥会拖垮 Server，client-go 又提供了 Informer，其内利用缓存机制化解这一风险。
* 由此可见， client-go 极大降低了客户端程序的开发成本。

从代码结构来看，Lister 和 Informer 比较相似，通过 Lister 来进行讲解。以 v1 版的 Deployment 来举例， 其 源 代 码 位 于 `staging/src/k8s.io/client-go/listers/apps/v1/deployment.go`。

Deployment 的 client 提供两个接口供客户端程序使用：

* `DeploymentNamespaceLister` 接口。提供了 `List()` 和 `Get()` 两个方法，用于获取某一个命名空间内的 Deployment 实例。

  ```go
  // DeploymentNamespaceLister helps list and get Deployments.
  // All objects returned here must be treated as read-only.
  type DeploymentNamespaceLister interface {
  	// List lists all Deployments in the indexer for a given namespace.
  	// Objects returned here must be treated as read-only.
  	List(selector labels.Selector) (ret []*appsv1.Deployment, err error)
  	// Get retrieves the Deployment from the indexer for a given namespace and name.
  	// Objects returned here must be treated as read-only.
  	Get(name string) (*appsv1.Deployment, error)
  	DeploymentNamespaceListerExpansion
  }
  ```

* `DeploymentLister` 接口。提供了 `List()` 方法，具有过滤能力，只返回所有符合条件的 Deployment；还提供一个 `Deployments()` 方法返回 `DeploymentNamespaceLister` 对象，该对象用于在单一命名空间内获取 Deployment 实例。

  ```go
  // DeploymentLister helps list Deployments.
  // All objects returned here must be treated as read-only.
  type DeploymentLister interface {
  	// List lists all Deployments in the indexer.
  	// Objects returned here must be treated as read-only.
  	List(selector labels.Selector) (ret []*appsv1.Deployment, err error)
  	// Deployments returns an object that can list and get Deployments.
  	Deployments(namespace string) DeploymentNamespaceLister
  	DeploymentListerExpansion
  }
  ```

  如果打开另一个 API 资源的 Lister，例如 `core/pod.go`，发现其核心内容和 Deployment 的 Lister 如出一辙，具有类似的两个 Lister 接口，各个接口内的方法也类似:

  1. `PodNamespaceLister` 接口。提供了 `List()` 和 `Get()` 两个方法，用于获取某一个命名空间内的 Pod；
  2. `PodLister` 接口。提供了 `List()` 方法，筛选所有符合条件的 Pod；提供一个 `Pods()` 方法返回 `PodNamespaceLister` 实例，该对象用于在单一命名空间内找 `Deployment`。

实际上，上述 `clientset`，`lister` 和 `informer` 对于每个 API 资源来说代码结构都极为类似， 适合用代码生成来减轻开发工作量。`code-generator` 中提供了与之对应的三种代码生成器： `clientset-gen`,、`lister-gen` 和 `informer-gen`，并且这三个生成器共享一系列注解。当我们需要把某一个内置 API 的某一外部版本加入 clientset 中时，只需在该 API 种类的基座结构体上方加如下注解：

```go
// +genclient
```

这样，clientset 代码生成器就会为：

1. **为该资源生成 typed client 代码**——包含 `Get/List/Create/Update/...` 这些方法，封装了对该资源 REST 接口的调用。
2. **把这个 typed client 挂到 clientset 上**——即在 clientset struct 里新增一个对应的字段/方法（比如 `FooV1()`），返回刚生成的 `typed client`。这样上层代码就能通过 `clientset.FooV1().Bars()` 这种链路访问该 API。

由于上述 CRUD 操作涉及通过端点和 Server 交互，而端点的 URL 结构受资源是否有命名空间影响，所以若这个资源是命名空间无关的，则需额外加如下注解：

```go
// +genclient:nonNamespaced
```

对一个 API 资源可以做的所有操作（也称为 verb）包括：`create`、`update`、`updateStatus`、 `delete`、`deleteCollection`、`get`、`list`、`watch`、`patch`、`apply` 和 `applyStatus`，代码生成器支持为它们生成实现方法。可以通过如下注解来限定或排除某些操作：

```go
// +genclient:onlyVerbs=<...>
// +genclient:skipVerbs=<...>
```

在这些操作中 `updateStatus` 和 `applyStatus` 比较特殊，它们是针对 API 资源的 Status 子资 源的，但有时一个 API 资源没有 Status 子资源，为这两个 verb 生成代码没有意义，对于这种情况可以通过如下标签阻止代码生成：

```go
// +genclient:noStatus
```

`list` 和 `watch` 操作蕴含比较丰富的内容。首先，如果 client-gen 代码生成器在注解上发现它们在列，则会为相应的 client 生成 List 方法和 Watch 方法，它们一个用来从 API Server 拉取一列资源，一个和 API Server 建立长时连接，不断获取目标资源的最新状态。这两个方法是**不带本地缓存**的，每次调用都会对 Server 发出请求。

`lister-gen` 代码生成器和 `informer-gen` 代码生成器是 `client-go` 生成器的辅助工具，当它们运行时，会扫描 `genclient` 标签出现在哪些资源类型定义上，哪些包含了 list 和 watch 操作：lister-gen 为支持 list 的 API 外部版本生成上述的 List 接口等代码。

`informer-gen` 会为支持 list 和 watch 的 API 外部版本生成 Informer 代码。Informer 引入缓存机制减轻 API Server 负担，优化了与 API Server 的交互，当然难免引入编程时的复杂度，特 别是开发人员要格外小心对资源的修改不应直接影响缓存中的资源实例。

> 可能你这里有点混乱了，怎么不带缓存，怎么又引入了缓存：
>
> * **List/Watch（client-gen）**：裸 API 调用，不做缓存；你用多少次就向 server 打多少次请求。
> * **Informer（informer-gen）**：在内部用 `List/Watch` 获取最新数据，同时把结果保存到本地缓存，并通过事件回调通知你的控制逻辑。
> * **Lister（lister-gen）**：只读 Informer 的缓存，提供快速只读访问。

除了以上标准的 verb，client-gen 代码生成器还支持自定义的非标准操作，例如在源码中经常看到如下标签：

```go
// +genclient:method=GetScale,verb=get,subresource=scale,result=k8s.io/api/autoscaling/v1.Scale
```

上述标注定义了一个非标准操作，向 API Server 发送 GET 请求，获取当前 API 资源实例的 Scale 子资源，并要求为该操作生成的方法名为 `GetScale()`。最终生成的 `client-go` 代码可参考 `v1 app/StatefulSet`：

```go
// 代码: staging/src/k8s.io/client-go/kubernetes/typed/apps/v1/statefulset.go
// GetScale takes name of the statefulSet, and returns the corresponding autoscalingv1.Scale object, and an error if there is any.
func (c *statefulSets) GetScale(ctx context.Context, statefulSetName string, options metav1.GetOptions) (result *autoscalingv1.Scale, err error) {
	result = &autoscalingv1.Scale{}
	err = c.GetClient().Get().
		UseProtobufAsDefault().
		Namespace(c.GetNamespace()).
		Resource("statefulsets").
		Name(statefulSetName).
		SubResource("scale").
		VersionedParams(&options, scheme.ParameterCodec).
		Do(ctx).
		Into(result)
	return
}
```

## 4.2 代码生成举例

选取 apps 这个 API 组和其下的 Deployment API 为例，看为代码生成设置的注解、代码生成的产出和干预代码生成的机会。

### 4.2.1 DeepCopy

API 的基座结构体需要 DeepCopy 相关方法。对于内部版本，在包 `pkg/apis/apps/doc.go` 中加全局注解：

```go
// 代码 pkg/apis/apps/doc.go
// +k8s:deepcopy-gen=package

package apps 
```

对于 API 的基座结构体，还需要返回值类型为 `runtime.Object` 的 DeepCopy 方法，需要在这类结构体上加如下注解，以 Deployment 为例：

```go
// 代码: pkg/apis/apps/types.go
// Deployment provides declarative updates for Pods and ReplicaSets.
type Deployment struct {
	metav1.TypeMeta
	// +optional
	metav1.ObjectMeta

	// Specification of the desired behavior of the Deployment.
	// +optional
	Spec DeploymentSpec

	// Most recently observed status of the Deployment.
	// +optional
	Status DeploymentStatus
}
```

上述代码中注解促使一个 go 源文件被生成，位于 `pkg/apis/apps/zz_generated.deepcopy.go`，内含 DeepCopy 相关的源代码。

对于外部版本，DeepCopy 同样需要。注解的设置和内部版本并没有不同。唯一需要注意的是，外部版本的类型定义是作为单独代码库发布的，在 `pkg/apis/apps/<版本号>` 下是找不到它们的源码，要到 `staging/src/k8s.io/api/apps/<版本号>` 下，注解分别加在其内的 `doc.go` 和 `types.go` 中，所生成的代码也是放置在其下。

### 4.2.2 Conversion

API 的内外部版本之间转换是必须的，这里以外部版本 v1 和内部版本的转换为例说明。 在 v1 的包定义文件 `pkg/apis/apps/v1/doc.go` 中有 conversion 相关的两个注解：

```go
// 代码: pkg/apis/apps/v1/doc.go
// +k8s:conversion-gen=k8s.io/kubernetes/pkg/apis/apps
// +k8s:conversion-gen-external-types=k8s.io/api/apps/v1
// +k8s:defaulter-gen=TypeMeta
// +k8s:defaulter-gen-input=k8s.io/api/apps/v1

package v1 // import "k8s.io/kubernetes/pkg/apis/apps/v1"
```

虽然内外部版本的转换是两个方向的，但注解设置只需在外部版本的包上进行。

由于自动生成无法完全实现 Deployment 版本之间的转换，需要手动在`pkg/apps/v1/conversion.go` 下添加代码，如下所示（略过反向转换方法）

```go
// 代码: pkg/apis/apps/v1/conversion.go
func Convert_apps_Deployment_To_v1_Deployment(in *apps.Deployment, out *appsv1.Deployment, s conversion.Scope) error {
	if err := autoConvert_apps_Deployment_To_v1_Deployment(in, out, s); err != nil {
		return err
	}

	out.Annotations = deepCopyStringMap(out.Annotations) // deep copy because we modify it below

	// Copy deprecated rollbackTo field to annotation for roundtrip
	// TODO: remove this conversion after we delete extensions/v1beta1 and apps/v1beta1 Deployment
	if in.Spec.RollbackTo != nil {
		if out.Annotations == nil {
			out.Annotations = make(map[string]string)
		}
		out.Annotations[appsv1.DeprecatedRollbackTo] = strconv.FormatInt(in.Spec.RollbackTo.Revision, 10)
	} else {
		delete(out.Annotations, appsv1.DeprecatedRollbackTo)
	}

	return nil
}
```

这个 Convert 函数将会被所生成的代码调用。自动生成的代码将被放入源文件 `pkg/apis/apps/v1/zz_generated.conversion.go` 中。

### 4.2.3 Defaulter

默认值代码生成只需对**外部版本**进行，所以全局注解设置在`pkg/apis/apps/v1/doc.go` 中。 如 `pkg/apis/apps/v1/doc.go` 所示，它圈定所有具有 TypeMeta 字段的顶层结构体，为它们生成默认值设置方法。为了注入自有逻辑，在 `pkg/apis/apps/v1/defaults.go` 中开发人员为基座结构体编写了 `SetDefaults_<类型>()` 方法。所生成的代码位于 `pkg/apis/apps/v1/zz_generated.defaults.go`。

### 4.2.4 client-go 相关代码

只有 API 的外部版本具有 client-go 相关的代码，所以相应注解会设置在外部版本的基座结构体类型定义上，由 4.1 节可知，Clientset，Lister 和 Informer 全部是局部注解。针对 apps/v1 的 Deployment，其类型定义所在源文件为 `staging/src/k8s.io/api/apps/v1/types.go`，注解部分代码如下所示：

```go
// 代码: staging/src/k8s.io/api/apps/v1/types.go
// +genclient
// +genclient:method=GetScale,verb=get,subresource=scale,result=k8s.io/api/autoscaling/v1.Scale
// +genclient:method=UpdateScale,verb=update,subresource=scale,input=k8s.io/api/autoscaling/v1.Scale,result=k8s.io/api/autoscaling/v1.Scale
// +genclient:method=ApplyScale,verb=apply,subresource=scale,input=k8s.io/api/autoscaling/v1.Scale,result=k8s.io/api/autoscaling/v1.Scale
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +k8s:prerelease-lifecycle-gen:introduced=1.9

// Deployment enables declarative updates for Pods and ReplicaSets.
type Deployment struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Specification of the desired behavior of the Deployment.
	// +optional
	Spec DeploymentSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

	// Most recently observed status of the Deployment.
	// +optional
	Status DeploymentStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

生成的 client-go 代码存放在 staging/src/k8s.io/client-go 下，作为 client-go 库的组成部分独立发布。

## 4.3 触发代码生成

`code-generator` 工程提供了诸多代码生成器，每一个都以一个可执行文件的形式存在， 例如 defaulter 生成器对应可执行文件名为 `defaulter-gen`，将被置于`$GOPATH/bin` 下。编译 Kubernetes 系统前要确保代码生成已经完毕，否则一定失败。成功生成代码需要：

1. 所有代码生成器的可执行文件都已经置于 `$GOPATH/bin` 下；
2. 要确保调用各个生成器时所使用的参数都正确。

由于生成器数量较多，手动满足上述条件比较繁琐的，且对于 CI 工具来说，手动执行上述步骤是行不通的。于是，Kubernetes 工程在 hack 目录下提供了脚本 `update-codegen.sh`， 可直接完成所有代码生成相应的工作。

该脚本代为执行上述两步：它先确保代码生成器的可执行文件已经存在，否则自动使用 go 命令编译 code-generator 并安装——即将可执行文件放入$GOPATH/bin；然后组织参数对 各个代码生成器进行调用。调用 Informer 生成器的核心代码如下所示：

```bash
# 代码: hack/update-codegen.sh
function codegen::informers() {
    if [[ -n "${LINT:-}" ]]; then                                                             
        if [[ "${KUBE_VERBOSE}" -gt 2 ]]; then
            kube::log::status "No linter for informers codegen"
        fi
        return
    fi

    GOPROXY=off go install \
        k8s.io/code-generator/cmd/informer-gen

    local ext_apis=()
    kube::util::read-array ext_apis < <(
        cd "${KUBE_ROOT}/staging/src"
        git_find -z ':(glob)k8s.io/api/**/types.go' \
            | while read -r -d $'\0' F; do dirname "${F}"; done \
            | sort -u)

    kube::log::status "informers: code for ${#ext_apis[@]} targets"
    if [[ "${DBG_CODEGEN}" == 1 ]]; then
        kube::log::status "DBG: running informer-gen for:"
        for api in "${ext_apis[@]}"; do
            kube::log::status "DBG:     $api"
        done
    fi

    (git_grep -l --null \
        -e '^// Code generated by informer-gen. DO NOT EDIT.$' \
        -- \
        ':(glob)staging/src/k8s.io/client-go/**/*.go' \
        || true) \
        | xargs -0 rm -f

    informer-gen \
        -v "${KUBE_VERBOSE}" \
        --go-header-file "${BOILERPLATE_FILENAME}" \
        --output-dir "${KUBE_ROOT}/staging/src/k8s.io/client-go/informers" \
        --output-pkg "k8s.io/client-go/informers" \
        --single-directory \
        --versioned-clientset-package "k8s.io/client-go/kubernetes" \
        --listers-package "k8s.io/client-go/listers" \
        --plural-exceptions "${PLURAL_EXCEPTIONS}" \
        "${ext_apis[@]}" \
        "$@"

    if [[ "${DBG_CODEGEN}" == 1 ]]; then
        kube::log::status "Generated informer code"
    fi
}
```

当 API 资源类型被修改后，开发人员都需要调用该脚本进行重新进行代码生成工作。如果不确定是否需要重新生成，我们可以运行 `hack/verify-codegen.sh` 脚本来检验改动，当改动需要重新生成代码时，这个脚本的输出会提示。

## 4.4 生成 API OpenAPI 规格说明

OpenAPI 是一个 HTTP 程序编程接口描述规范，要解决的问题是兼顾人类可读与机器可读可理解，以统一格式准确描述基于 HTTP 的服务接口。

用 OpenAPI 规范精确描述 REST API 也是有代价的：工作量大，且不容有错。API Server 提供那么多 API 资源，有如此之多的端点，手工为它们撰写 OpenAPI 服务规格说明不可接受。解决办法是通过代码生成来自动生成。生成与消费的过程如下图所示。

![image-20251121220401052](http://images.liangning7.cn/typora/202511212204361.png)

### 4.4.1 加注解

打开任意一个 Kubernetes API 组的外部类型定义包，找到其内 `doc.go` 文件，例如 `staging/src/k8s.io/api/apps/v1/doc.go`，会看到如下注解：

```go
// 代码 staging/src/k8s.io/api/apps/v1/doc.go
// +k8s:deepcopy-gen=package
// +k8s:protobuf-gen=package
// +k8s:openapi-gen=true // 要点①

package v1 // import "k8s.io/api/apps/v1"
```

代码中要点①处的注解标明需要为本包下定义的所有类型生成符合 OpenAPI 的描述，称之为 Kubernetes API 的 Schema。这些类型描述将被用于在 OpenAPI 规格说明中定义 API 端点的参数。

### 4.4.2 执行生成

注解只是标明哪些 Go 类型需要生成 OpenAPI 内 Schema，而真正的生成需要触发，并且要在编译 API Server 为可执行文件前进行。脚本 `update-codegen.sh` 包含了触发 OpenAPI Schema 的代码生成。最终结果是在工程的 `/pkg/generated/openapi/v3` 下生成一个巨大的文件 `zz_generated.openapi.go`，该文件内含用于为每个内置 API 基座结构体生成 OpenAPI Schema 的函数，调用这些函数就可以获得对应内置 API 基座结构体的 Schema。`apps/v1 Deployment` 的 Schema 生成方法的代码如下所示：

```go
// 代码: pkg\generated\openapi\zz_generated.openapi.go
func schema_k8sio_api_apps_v1_Deployment(ref common.ReferenceCallback) common.OpenAPIDefinition {
	return common.OpenAPIDefinition{
		Schema: spec.Schema{
			SchemaProps: spec.SchemaProps{
				Description: "Deployment enables declarative updates for Pods and ReplicaSets.",
				Type:        []string{"object"},
				Properties: map[string]spec.Schema{
					"kind": {
						SchemaProps: spec.SchemaProps{
							Description: "Kind is a string value representing the REST resource this object represents. Servers may infer this from the endpoint the client submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds",
							Type:        []string{"string"},
							Format:      "",
						},
					},
					"apiVersion": {
						SchemaProps: spec.SchemaProps{
							Description: "APIVersion defines the versioned schema of this representation of an object. Servers should convert recognized schemas to the latest internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources",
							Type:        []string{"string"},
							Format:      "",
						},
					},
					"metadata": {
						SchemaProps: spec.SchemaProps{
							Description: "Standard object's metadata. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata",
							Default:     map[string]interface{}{},
							Ref:         ref(metav1.ObjectMeta{}.OpenAPIModelName()),
						},
					},
					"spec": {
						SchemaProps: spec.SchemaProps{
							Description: "Specification of the desired behavior of the Deployment.",
							Default:     map[string]interface{}{},
							Ref:         ref(appsv1.DeploymentSpec{}.OpenAPIModelName()),
						},
					},
					"status": {
						SchemaProps: spec.SchemaProps{
							Description: "Most recently observed status of the Deployment.",
							Default:     map[string]interface{}{},
							Ref:         ref(appsv1.DeploymentStatus{}.OpenAPIModelName()),
						},
					},
				},
			},
		},
		Dependencies: []string{
			appsv1.DeploymentSpec{}.OpenAPIModelName(), appsv1.DeploymentStatus{}.OpenAPIModelName(), metav1.ObjectMeta{}.OpenAPIModelName()},
	}
}
```

### 4.4.3 启动时生成并缓存

经过前面两步，获取了 Kubernetes API 在 OpenAPI 规格说明文档中的 Schema，但 Server 上还并不存在各 API 端点的 OpenAPI 规格说明文档。启动时，Server 会根据已向其注册的端点（go-restful 中的 web service）和上述 Schema 信息，在内存中为每个 API 版本内的所有端点生成对应的 OpenAPI 规格说明文档，并缓存于内存中，等待客户端访问请求的到来。
