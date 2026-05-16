+++
title = "Tags"
date = 2026-05-16
draft = false

categories = [
    "cloud-native"
]

tags = [
    "Go",
    "云原生",
    "Kubernetes"
]

series = [
    "K8s-Codegen"
]
series_order = 2

+++

# Tag

代码生成器通过 Tags 来识别一个包是否需要生成代码和生成代码的方式，Kubernetes 提供的 Tags 可以分为两种：

- **全局 Tags**：定义在每个包的 `doc.go` 文件中，对整个包中的类型自动生成代码 。  
- **局部 Tags**：定义在 Go 语言的类型声明上方，只对指定的类型自动生成代码 。

> Tags 的定义规则通常为“`//+tag-name`”或“`//+tag-name=value`”格式，它们的定义在注释中 。

## 全局 Tags

全局 Tags 定义在 `pkg/apis/<group>/<version>/doc.go` 中，代码示例如下：

```go
//+k8s: deepcopy-gen-package
// +groupName=example.com
package v1
```

全局 Tags 告诉 `deepcopy-gen` 代码生成器，为该包中的每个类型自动生成 DeepCopy 函数 。其中，“`//+groupName`”定义了资源组名称，资源组名称一般使用域名形式命名 。

## 局部 Tags

局部 Tags 定义在 Go 语言的类型声明上方，代码示例如下：

代码路径：`pkg/apis/core/types.go`

```go
// +genclient
// +k8s: deepcopy-gen: interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Pod is a collection of containers, used as either input (create, update) or as output (list, get).
type Pod struct {
    metav1.TypeMeta
    // +optional
    metav1.ObjectMeta
    // +optional
    Spec PodSpec
    // +optional
    Status PodStatus
}
```

例如，Pod 资源类型的局部 Tags 定义在 Pod 资源类型的上方，该类型有两个代码生成器，分别为 `genclient` 和 `deepcopy-gen` 。其中，`genclient` (即 `client-gen`) 代码生成器为这个资源类型自动生成对应的 Client 代码；`deepcopy-gen` 代码生成器为这个资源类型自动生成 DeepCopy 相关函数。

> **注意：** 关于 Tags 的位置，局部 Tags 一般定义在类型声明的上方，但如果该类型有注释信息，则局部 Tags 的定义和类型声明的注释信息之间至少要有一个空行 。例如：  
>
> ```go
> // +tag
> 
> // comment-block
> type Foo struct {
> }
> ```
>
> 这是因为 Kubernetes 的 API 文档生成器会根据类型声明的注释信息 (comment-block) 生成文档 。为了避免 Tags 信息出现在文档中，所以将 Tags 定义在注释的上方并空一行 。
