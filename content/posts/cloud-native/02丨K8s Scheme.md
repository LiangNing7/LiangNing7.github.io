+++
title = "K8s Scheme"
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
series_order = 2
+++
API Server 可以分为 API 和 Server 两个部分来看待：

1. Server 的主要目标是提供一个以 go-restful 为基础的 Web 服务器，在设计与实现时不会受 API 的结构影响，每个 API 在 Server 看来都是一组端点、一组 Route，Server 提供方法把 API 转化为 WebService 和 Route；
2. API 可以视作信息的容器，是用户向 Server 提交信息的载体。它专注在自己具有的属性、属性默认值、各个版本之间的转换方式等，而不应该受下层 Server 的设计影响。对 API Server 能力的扩展主要是指不断引入新的 API，让 API Server 服务的范畴得到扩展， 如果说每次引入新的 API 都需要对 Server 本身的代码进行改动，则这种设计显然是不太理想的。

二者的独立促成两个方面的灵活性：第一，设计时不互相影响；第二，扩展时各自可变。但这要求深度解耦 API 和 Server，Scheme 是 Kubernetes 达到这一目的的重要工具。

Scheme 就像一个**类型注册中心**：

* API 向它声明"我是谁"（类型、版本、转换规则）
* Server 向它查询"你是谁"（获取类型元信息生成路由）
* 两者通过 Scheme 通信，永远不直接依赖对方

# 1. 注册表的内容

Scheme 技术上由一个 Go 结构体代表，代码如下所示：

Scheme 的数据结构如下图：

![image-20251021155316141](http://images.liangning7.cn/typora/202510211553423.png)

Scheme 数据结构定义的代码示例如下。

```go
type Scheme struct {
	gvkToType map[schema.GroupVersionKind]reflect.Type
	typeToGVK map[reflect.Type][]schema.GroupVersionKind
	unversionedTypes map[reflect.Type]schema.GroupVersionKind
	unversionedKinds map[string]reflect.Type
	fieldLabelConversionFuncs map[schema.GVK]FieldLabelConversionFunc
	defaulterFuncs map[reflect.Type]func(interface{})
	converter *conversion.Converter
	versionPriority map[string][]string
	observedVersions []schema.GroupVersion
	schemeName string
} 
```

Scheme 数据结构定义代码中部分字段的说明如下：

* `gvkToType`：存储 GVK 与 Type 的映射关系。map 类型，key 为 `schema.GroupVersionKind` 类型，即 GVK 三元组，value 为 reflect.Type 类型。即 GVK 唯一对应的 Go Struct 数据结构。通过该字段中存储的信息，只要知道 GVK 信息，就可以通过反射创建出对应 的结构体以接收数据。

* `typeToGVK`：存储 Type 与 GVK 的映射关系，用于根据 Go Struct 反向查询该数据所属的 GVK。map 类型，key 为 reflect.Type 类型，value 为 GVK 列表，一个 Type 会对应一个或多个 GVK。因为即使一个 Kind 在不同的版本甚至不同的组中，但其结构体字段没有改变，所以获取结构体得到的 reflect.Type 就是一样的。例如，`__internal` 版本的 `extensions.Ingress` 、 `networking.k8s.io.Ingress` ，它们的 GVK 对应同一个 Type； `CreateOptions` 资源对象有 49 个 GVK，在不同组、版本中的 reflect.Type 都是一样的。

* `unversionedTypes`：存储 UnversionedType 与 GVK 的映射关系。key 为 reflect.Type 类型， value 为单个 GVK，这个字段与 typeToGVK 有所不同，value 不再是 GVK 列表，这也正与 Unversioned 的概念相对应，该资源类型永远只保留一个版本，无须进行版本化管 理。目前对应的只有 5 个 unversionedTypes，Go Struct 分别为 `metav1.Status`、 `metav1.APIVersions`、`metav1.APIGroupList`、`metav1.APIGroup`、`metav1.APIResourceList`。

* `unversionedKinds`：存储 Kind 名称与 UnversionedType 的映射关系。key 为字符串类型， 存储的是 5 个 unversionedType 对应的 Kind，为什么这里不使用 GVK 作为 key 的类型 呢？因为 5 个 unversionedType 的组都是""，版本都是 v1，所以无须重复指定组和版本， 只需要将 Kind 作为 key，对应的 5 个 Kind 分别是 Status、APIVersions、APIGroupList、 APIGroup、APIResourceList。

  示例：

  ```go
  // unversionedKinds 的实际内容
  scheme.unversionedKinds = map[string]reflect.Type{
      // Key                Value (Go 类型的反射 Type)
      // ==========================================
      "Status":           reflect.TypeOf(metav1.Status{}),
      "APIVersions":      reflect.TypeOf(metav1.APIVersions{}),
      "APIGroupList":     reflect.TypeOf(metav1.APIGroupList{}),
      "APIGroup":         reflect.TypeOf(metav1.APIGroup{}),
      "APIResourceList":  reflect.TypeOf(metav1.APIResourceList{}),
  }
  ```

  GVK 与 reflect.Type 是多对一的关系，如下图：

  ![image-20251021161346671](http://images.liangning7.cn/typora/202510211613914.png)

* `fieldLabelConversionFuncs`：将 Version 和 Resource 映射到对应的函数中，将该版本中的资源字段标签转换为内部版本字段。

* `defaulterFuncs`：存储设置对象默认值的函数。

* `converter`：转换器，存储所有已注册的转换函数，在初始化时会初始化部分默认转换函 数，是实现资源版本转换的关键。

* `versionPriority`：API 组到这些组中有序版本列表的映射，指示这些版本在 Scheme 中注册的默认优先级。

* `observedVersions`：GV 数组类型，跟踪在资源类型注册期间看到的版本的顺序，不包括内部版本，可用于快速查询某个组或版本在 Scheme 中是否注册过。

* `schemeName`：Scheme 的名称，没有实质含义，主要用于标识该 Scheme，如在出错时 打印相关信息。

# 2. 注册表的构建

注册表内容的填入是在 Server 启动过程中完成的，后续运行时不会更改。整个程序设计非常优雅，体现了 Go 语言优良特性。

## 2.1 Builder 模式

注册表的填充是遵循了构造者模式，经典构造者模式角色关系如图：

![image-20251118143403195](http://images.liangning7.cn/typora/202511181434488.png)

该模式中有 4 个角色，其中产品（Product）在本模式中没有逻辑不必细说，**ConcreteBuilder 是 Builder 这个抽象角色的具体实现，所以二者扮演同一角色**。

Director 和 Builder 各有责任：

1. `Director`：通过定义产品零部件决定规格。例如同品牌同版本的手机，存储大小不同造就出不同的产品，市场售价会不同。Director 决定最终产品包含哪些零部件以及规格，但它不负责具体加工过程。
2. Builder 和 ConcreteBuilder：负责把产品零部件组合成产品，掌握组合流程。Director 决定给什么样的零件，Builder 负责把它们按照正确的方式装起来。一般来说，在构建一个 Builder 的时候，需要给足创建一个产品的必备零部件， 不能少掉哪个，否则 Builder 肯定不能工作；而那些可选零部件会通过 Builder 暴露的方法去指定。

## 2.2 注册表的 Builder

SchemeBuilder 扮演了构造者模式中的 Builder 角色。

```go
// 代码: staging/k8s.io/apimachinery/pkg/runtime/scheme_builder.go
package runtime

// SchemeBuilder collects functions that add things to a scheme. It's to allow
// code to compile without explicitly referencing generated types. You should
// declare one in each package that will have generated deep copy or conversion
// functions.
type SchemeBuilder []func(*Scheme) error

// AddToScheme applies all the stored functions to the scheme. A non-nil error
// indicates that one function failed and the attempt was abandoned.
func (sb *SchemeBuilder) AddToScheme(s *Scheme) error {
	for _, f := range *sb {
		if err := f(s); err != nil {
			return err
		}
	}
	return nil
}

// Register adds a scheme setup function to the list.
func (sb *SchemeBuilder) Register(funcs ...func(*Scheme) error) {
	for _, f := range funcs {
		*sb = append(*sb, f)
	}
}

// NewSchemeBuilder calls Register for you.
func NewSchemeBuilder(funcs ...func(*Scheme) error) SchemeBuilder {
	var sb SchemeBuilder
	sb.Register(funcs...)
	return sb
}
```

以上代码定义了 SchemeBuilder，一共就 33 行，做了两件事情：第一定义类型 `SchemeBuilder`；第二为该类型加方法。

SchemeBuilder 被定义为一个数组，**方法的数组**，在 Go 中方法是可以被当作变量使用的。这个数组里的方法的签名有要求：**只能有一个输入参数，其类型为 Scheme 的引用；返回值类型为 error**，只有在出错时返回值才非 nil。

>  `SchemeBuilder` 定义为 `[]func(*Scheme) error` 的设计是一个**延迟注册**模式，带来以下核心优势：解决代码生成和手写代码的分离问题。

在 Go 中， 任何类型都可以作为方法的接收器，这很像为该类型添加方法，而在面向对象语言中，有类 / 接口才能有方法，这也算 Go 特色。上述代码为 SchemeBuilder 数组定义了三个方法：

1. `NewSchemeBuilder()` 方法：Builder 的工厂方法，通过它获取一个 Builder 实例。设计模式中的 Builder 构造函数接收那些必须的零部件，所以这个方法提供了一个输入参数： 可以放入 SchemeBuilder 数组的函数。SchemeBuilder 使用的“零部件”都是以函数形式存在的；
2. `Register()` 方法：把可选零部件放入 SchemeBuilder 数组中。同样的，这里的“零部件”是函数。
3. `AddToScheme()` 方法：用零部件构造出产品——一个 Scheme 实例。它相当于经典模式中 Builder 的 GetProduct 方法：当零部件都给全了，调用这个方法，用这些零部件填充本方法的传入参数得到完整的 Scheme 实例。其内部实现很简洁，是对 SchemeBuilder 数组内保存的“零部件”做一个遍历，如上所述它们是一个个函数，遍历它们能干什么？自然是调用，以本方法获得的实际参数 scheme 为入参去调用这些函数。而这些函数内部会把自己掌握的信息交给该 Scheme 实例，完成 Scheme 实例的构建。

## 2.3 注册表的 Director-register.go

有了 Builder，接下来看模式中的 Director 是怎么实现的。模式中的 Director 会决定给什么“零部件”，前面看到 SchemeBuilder 接收的零部件是以 Scheme 指针为形式参数的函数，那么在 Director 中我们就应该能看到这些函数。

Kubernetes Scheme 构建中的 Director 有多个，每个 API Group 的每一个内部版本和全部外部版本各有一个 Director，这和模式中 Director 的定位是吻合的：每个 API 版本中的 API 逻辑上都是不同的，所以不能共用同一个。在 API Server 的实现中，这个角色由不同的 `register.go` 文件来承担。

### 2.3.1 内部版本

以 apps 组中内部版本的 `register.go` 文件为例，一窥究竟。代码如下所示：

```go
// 代码: pkg\apis\apps\register.go
package apps

import (
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/kubernetes/pkg/apis/autoscaling"
)

var (
	// SchemeBuilder stores functions to add things to a scheme.
    // 注册表 Builder
	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes) //要点①
	// AddToScheme applies all stored functions t oa scheme.
    // 在本包上暴露 Builder 的 AddToScheme() 方法
	AddToScheme = SchemeBuilder.AddToScheme //要点④
)

// GroupName is the group name use in this package
// 本 package 内用的组名
const GroupName = "apps"

// SchemeGroupVersion is group version used to register these objects
// SchemeGroupVersion 是注册这些对象时使用的组和版本
var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: runtime.APIVersionInternal}

// Kind takes an unqualified kind and returns a Group qualified GroupKind
// 由 API 种类获取 GroupKind
func Kind(kind string) schema.GroupKind {
	return SchemeGroupVersion.WithKind(kind).GroupKind()
}

// Resource takes an unqualified resource and returns a Group qualified GroupResource
// 由 Resource 获取一个 GroupResource
func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

// Adds the list of known types to the given scheme.
// 将一列类型添加到给定的注册表内
func addKnownTypes(scheme *runtime.Scheme) error { //要点②
	// TODO this will get cleaned up with the scheme types are fixed
	scheme.AddKnownTypes(SchemeGroupVersion,
		&DaemonSet{}, //要点③
		&DaemonSetList{},
		&Deployment{},
		&DeploymentList{},
		&DeploymentRollback{},
		&autoscaling.Scale{},
		&StatefulSet{},
		&StatefulSetList{},
		&ControllerRevision{},
		&ControllerRevisionList{},
		&ReplicaSet{},
		&ReplicaSetList{},
	)
	return nil
}
```

关注上述代码要点①处 `SchemeBuilder` 类型实例的构造和要点②处 `addKnownTypes()` 方法。

Builder 实例的获取借助前面介绍的 `NewSchemeBuilder()` 方法，它的唯一形参类型是 **Scheme 指针**，在 `register.go` 中调用该方法时使用的实际参数是 `addKnownTypes()` 方法，这个方法把当前这个 API Group 中的所有 API 都交给目标 Scheme 实例：这里目标 Scheme 实例是通过参数传进来的，Group 中的 API 则是以它们的 Go 结构体实例表示的，见要点③及其下方代码。

Scheme 类型的 `AddKnownTypes()` 方法会填充注册表中 GVK 和 API 基座结构体的双向映射表。上面已经看到 `register.go` 中的 `addKnownTypes()` 方法被交给了 Builder，当 Builder 的 AddToScheme 方法被调用时，它就会被调用。Scheme 类型的 `AddKnowTypes()` 方法源代码如下：

```go
// 代码: staging/k8s.io/apimachinery/pkg/runtime/scheme.go
// AddKnownTypes registers all types passed in 'types' as being members of version 'version'.
// All objects passed to types should be pointers to structs. The name that go reports for
// the struct becomes the "kind" field when encoding. Version may not be empty - use the
// APIVersionInternal constant if you have a type that does not have a formal version.
func (s *Scheme) AddKnownTypes(gv schema.GroupVersion, types ...Object) {
	s.addObservedVersion(gv)
	for _, obj := range types {
		t := reflect.TypeOf(obj)
		if t.Kind() != reflect.Pointer {
			panic("All types must be pointers to structs.")
		}
		t = t.Elem()
		s.AddKnownTypeWithName(gv.WithKind(t.Name()), obj)
	}
}
```

总结一下：只要 Builder 的 AddToScheme 方法被调用，当前 API Group 的内部版本 API 与 GVK 映射关系会被填充到目标 Scheme 实例中。

其次，`register.go` 代码中的要点④，这一行把刚刚获得的 Builder 实例的 `AddToScheme()` 方法暴露为当前包（在本例中就是包 apps）上的同名方法 `AddToScheme()`，这样工程内其它代码就可以通过调用 apps 包下的 `AddToScheme()` 方法间接触发 Builder 填充一个给定的 Scheme 实例。这很重要，因为到目前为止，前面的所有工作只是为某 API 版本制作一个 Builder 的实例，并没有真正去填充某个 Scheme，甚至连目标 Scheme 实例是谁都不知道，需要后期通过 `apps.AddToScheme()` 方法的调用触发对某个 Scheme 的填充。

上例示例代码的 `register.go` 对应的是内部版本，对于 apps 这个 group 来说源码位于 `pkg/apis/apps/register.go`；而外部版本 `register.go` 极为类似，不同的是它位于 `staging/src/k8s.io/api/apps/v1/register.go` 中。

### 2.3.2 外部版本

```go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
)

// GroupName is the group name use in this package
const GroupName = "apps"

// SchemeGroupVersion is group version used to register these objects
var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: "v1"} // 要点①

// 要点②
// Resource takes an unqualified resource and returns a Group qualified GroupResource
func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

var (
	// TODO: move SchemeBuilder with zz_generated.deepcopy.go to k8s.io/api.
	// localSchemeBuilder and AddToScheme will stay in k8s.io/kubernetes.
	SchemeBuilder      = runtime.NewSchemeBuilder(addKnownTypes)
	localSchemeBuilder = &SchemeBuilder // 要点③
	AddToScheme        = localSchemeBuilder.AddToScheme
)

// Adds the list of known types to the given scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&Deployment{},
		&DeploymentList{},
		&StatefulSet{},
		&StatefulSetList{},
		&DaemonSet{},
		&DaemonSetList{},
		&ReplicaSet{},
		&ReplicaSetList{},
		&ControllerRevision{},
		&ControllerRevisionList{},
	)
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion) // 要点④
	return nil
}
```

虽然外部版本与内部版本的 `register` 是极其类似的，主要有如下区别：

1. **版本标识符**：

   ```go
   // internal version
   var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: runtime.APIVersionInternal}
   
   // v1 version  
   var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: "v1"}
   ```

   * **internal**: 使用 `runtime.APIVersionInternal`（内部常量，通常为 ""），**Internal version** 是内部版本，不对外暴露。
   * **v1**: 使用具体的版本号字符串 "v1"，**v1** 是外部版本，有具体的版本号。

2. **辅助函数**：

   * `Kind` 函数：把未带 group 的类型名补成 `apps` 组的 `schema.GroupKind`，供 apiserver 内部生成 REST 映射或策略时直接使用 `apps.Kind("Deployment")` 得到完整 `GroupKind`。
   * `Resource` 函数：把资源名补成 `apps` 组的 `schema.GroupResource`，用于内部存储/策略等对 “deployments、statefulsets…” 的统一处理。

3. **SchemeBuilder 定义**：

   ```go
   // internal version
   var (
   	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
   	AddToScheme = SchemeBuilder.AddToScheme
   )
   
   // v1alpha1 version
   var (
   	SchemeBuilder      = runtime.NewSchemeBuilder(addKnownTypes)
   	localSchemeBuilder = &SchemeBuilder
   	AddToScheme        = localSchemeBuilder.AddToScheme
   )
   ```

   * **Internal version** 直接创建：所有 defaulting、conversion、rest storage 等代码都在同一仓库里维护。该包只需要把自身的类型注册进 scheme，因此用值类型的 `SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)` 即可，`AddToScheme` 会在 apiserver 启动过程中调用，把内部类型加入 scheme；其它 defaulting/conversion 函数（同包或同仓库中）直接在这个 builder 上注册即可，没必要暴露指针
   * **v1 version** 采用延迟注册自定义逻辑【自己手写的 `defaulting / conversion` 函数】，而外部版本只定义**通用的注册逻辑**，而自定义的手写函数位于主仓，并且 Go 的 `runtime.SchemeBuilder` 是一个切片类型（值语义）。若直接暴露 `SchemeBuilder`（值），其它文件 import 后拿到的只是副本，调用 `Register` 只会修改副本，`AddToScheme` 仍使用原始那份，导致额外注册失效。因此外部包定义 `localSchemeBuilder = &SchemeBuilder` 并把 `AddToScheme = localSchemeBuilder.AddToScheme`，这样主仓中的任何文件都能通过指针调用 `localSchemeBuilder.Register(...)` 把函数追加到同一个 builder 上，最终 `AddToScheme` 会顺序执行 `addKnownTypes` 和这些额外函数。

4. **metav1.AddToGroupVersion 的差异**

   ```go
   // v1 版本多了一行
   metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
   ```

   * **v1** 需要调用这个来注册 metav1 相关的元数据到该 GroupVersion
   * **Internal version** 不需要，因为内部版本不关心这些元数据格式

## 2.4 注册表注册默认值设置函数

以外部版本 apps/v1 来讲解，外部版本在 pkg/apis/apps/v1 下有一个 `register.go`，部分代码如下所示：

```go
// 代码: pkg/apis/apps/v1
var (
	localSchemeBuilder = &appsv1.SchemeBuilder
	AddToScheme        = localSchemeBuilder.AddToScheme
)

func init() {
	// We only register manually written functions here. The registration of the
	// generated functions takes place in the generated files. The separation
	// makes the code compile even when the generated files are missing.
	//这里只注册人工撰写的默认值函数，自动生成的已经有被生成的代码注册
	// 如此分开的好处是即使代码生成没有进行，这部分编译还是会成功
	localSchemeBuilder.Register(addDefaultingFuncs)
}
```

在 v1 这个包的初始化过程中，调用了 Builder 方法的 `Register()` 方法，向 Builder 添加了 `addDefaultingFuncs()` 方法，我们也应该能猜到这个方法功用，它将向注册表中添加默认值设置函数。当 v1 版本对应的 Builder 被触发时，v1 版本的默认值设置方法将被注册进目标 Scheme 实例。

## 2.5 注册表注册版本转换函数

Converter 又是在哪里被注册进注册表的呢？实际上和 `addDefaultingFuncs`  一样，在包 v1 的初始化过程中，同样通过 `Register()` 方法交给 Builder，只不过这一部分源码不在 `v1/register.go` 中而是在 `v1/zz_generated.conversion.go` 中，代码如下所示：

```go
// 代码 pkg/apis/apps/v1/zz_generated.conversion.go

func init() {
	localSchemeBuilder.Register(RegisterConversions) //要点①
}
```

默认值设置函数和版本转换函数的代码，虽然不在同一源文件，但它们都隶属于包 v1。v1 在被引用时所有的包初始化——`init()` 函数都会被调用，包括 `v1/register.go` 中的 `init()` 和 `v1/zz_generated.conversion.go` 的 `init()` 函数。而这次，把 Conversion 的注册函数交给了 Builder， 见要点①处。

注册表有四个主要信息，到此已经找到了其中三个的注入地，分别是：GVK 与 API 基座结构体映射关系、默认值设置方法和内外部版本转换函数。至于最后一个主要信息“API Group 内各版本的重要级别”，马上展开讲解其注册地。

## 2.6 触发注册表的构建

经过上述代码准备，每个 API Group 的每个内外部版本都有了一个被 Director 设置好的 Builder，并且该 Builder 的产品构造触发方法 `AddToScheme()` 被暴露到该版本对应的 Go 包上，万事俱备，就等触发构造。但从解耦角度看还有待解决问题：Builder 是暴露在各个版本的包上的，要调用它们必须导入这些版本的包，外部触发程序岂不是要与这些包硬绑定？

为了解决这个问题，在每个 API Group 下，都建立了一个 `install/install.go` 文件，工程里其它代码只要导入了一个 API Group 包的 install 子包，就会触发该 API Group 下各个版本对应的 Scheme Builder，把信息填入一个全部 API Group 共享的 Scheme 实例，以 apps 这个 Group 举例，代码如下：

```go
func init() {
	Install(legacyscheme.Scheme)
}

// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
	utilruntime.Must(apps.AddToScheme(scheme))
	utilruntime.Must(v1beta1.AddToScheme(scheme))
	utilruntime.Must(v1beta2.AddToScheme(scheme))
	utilruntime.Must(v1.AddToScheme(scheme))
	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion, v1beta2.SchemeGroupVersion, v1beta1.SchemeGroupVersion)) // 优先级
}
```

> 可以从：
>
> ```go
> import (
>  "k8s.io/apimachinery/pkg/runtime"
> 
>  // 必须导入所有版本 - 硬绑定！
>  "k8s.io/kubernetes/pkg/apis/apps"
>  "k8s.io/kubernetes/pkg/apis/apps/v1"
>  "k8s.io/kubernetes/pkg/apis/apps/v1beta1"
>  "k8s.io/kubernetes/pkg/apis/apps/v1beta2"
> )
> ```
>
> 转为：
>
> ```go
> import (
>  "testing"
> 
>  // 仅需一个空白导入！
>  _ "k8s.io/kubernetes/pkg/apis/apps/install"
> )
> ```

`install` 包的初始化方法 `init()` 会调用 `Install` 方法，该方法逐个调用每个版本所暴露出的 `AddToScheme` 方法，把各个版本下的 API 信息填写入指定的注册表：`pkg/api/legacyscheme.go` 中定义的名为 Scheme、类型为 `runtime.Scheme` 的变量。除此以外，`Install` 方法还设置了这个 Group 下所有版本的优先级，这也是注册表中的一个信息。

现在已找到了注册表全部四个主要信息的注入点。

现在，每个内置 API Group 都提供了一种方便触发 Builder 工作的方式：导入该 Group 的 Install 子包即可。注册表构建的实际触发是在 API Server 启动时完成的，一经构建，运行时不再对其更改，过程如下图所示。

![image-20251122012623639](http://images.liangning7.cn/typora/202511220126915.png)

API Server 的应用程序从 main 包内代码开始运行，第一件事情就是导入依赖的包，这最终触发了注册表的填充。就如它的名字所揭示的一样，图中提及的 `pkg/controlplane/import_known_versions.go` 文件唯一的目的就是触发内建 API 向注册表的注册。

```go
package controlplane

import (
	// These imports are the API groups the API server will support.
	_ "k8s.io/kubernetes/pkg/apis/admission/install"
	_ "k8s.io/kubernetes/pkg/apis/admissionregistration/install"
	_ "k8s.io/kubernetes/pkg/apis/apiserverinternal/install"
	_ "k8s.io/kubernetes/pkg/apis/apps/install"
	_ "k8s.io/kubernetes/pkg/apis/authentication/install"
	_ "k8s.io/kubernetes/pkg/apis/authorization/install"
	_ "k8s.io/kubernetes/pkg/apis/autoscaling/install"
	_ "k8s.io/kubernetes/pkg/apis/batch/install"
	_ "k8s.io/kubernetes/pkg/apis/certificates/install"
	_ "k8s.io/kubernetes/pkg/apis/coordination/install"
	_ "k8s.io/kubernetes/pkg/apis/core/install"
	_ "k8s.io/kubernetes/pkg/apis/discovery/install"
	_ "k8s.io/kubernetes/pkg/apis/events/install"
	_ "k8s.io/kubernetes/pkg/apis/extensions/install"
	_ "k8s.io/kubernetes/pkg/apis/flowcontrol/install"
	_ "k8s.io/kubernetes/pkg/apis/imagepolicy/install"
	_ "k8s.io/kubernetes/pkg/apis/networking/install"
	_ "k8s.io/kubernetes/pkg/apis/node/install"
	_ "k8s.io/kubernetes/pkg/apis/policy/install"
	_ "k8s.io/kubernetes/pkg/apis/rbac/install"
	_ "k8s.io/kubernetes/pkg/apis/resource/install"
	_ "k8s.io/kubernetes/pkg/apis/scheduling/install"
	_ "k8s.io/kubernetes/pkg/apis/storage/install"
	_ "k8s.io/kubernetes/pkg/apis/storagemigration/install"
)
```
