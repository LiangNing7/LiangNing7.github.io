+++
title = "K8s Main Server Admission Prepare"
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

series_order = 13

+++

# K8s Main Server Admission Prepare

在梳理一下主 Server 的处理方式后，可见主 Server 的构造分为两个阶段：准备 Server 运行参数与创建主 Server 实例。准入控制器是在第一个阶段准备完毕，并在第二个阶段交给其底座 Generic Server。在第一阶段准备准入控制相关运行参数时经历两个环节：

1. 第一个环节是根据用户的命令行输入决定出最终控制器列表，该列表最终会落入选项结构体（Option）；
2. 第二环节是根据选项结构体中的控制器列表，生成 Server 运行配置中准入控制器列表【见 [K8s Generic Server Admission](./08丨K8s Generic Server Admission.md#调用链)】

![image-20251210171042467](http://images.liangning7.cn/typora/202512101710632.png)

## 运行选项和命令行参数

在API Server启动的最初阶段，一个类型为ServerRunOptions结构体实例就被构造出来，称它为运行选项，它也是生成命令行可用参数的基础。构造运行选项的部分代码如下：

```go
// 代码: cmd\kube-apiserver\app\options\options.go#L66-L97
// NewServerRunOptions creates and returns ServerRunOptions according to the given featureGate and effectiveVersion of the server binary to run.
func NewServerRunOptions() *ServerRunOptions {
	s := ServerRunOptions{
		Options: controlplaneapiserver.NewOptions(), // 要点①

		Extra: Extra{
        	...
        }
    }
}
```

Admission 字段位于要点①处的 `Options: controlplaneapiserver.NewOptions()` 中。

```go
// 代码: pkg\controlplane\apiserver\options\options.go#L112-L141
// NewOptions creates a new ServerRunOptions object with default parameters
func NewOptions() *Options {
	s := Options{
		GenericServerRunOptions: genericoptions.NewServerRunOptions(),
		Etcd:                    genericoptions.NewEtcdOptions(storagebackend.NewDefaultConfig(kubeoptions.DefaultEtcdPathPrefix, nil)),
		SecureServing:           kubeoptions.NewSecureServingOptions(),
		Audit:                   genericoptions.NewAuditOptions(),
		Features:                genericoptions.NewFeatureOptions(),
		Admission:               kubeoptions.NewAdmissionOptions(), // 要点①
        ...
    }
	...
}
```

要点①处 Admission 字段由 `kubeOptions.NewAdmissionOptions()` 方法生成的。

## Admission 的生成

Admission 字段的生成是调用的 `kubeoptions.NewAdmissionOptions()` 进行生成的，而 `kubeoptions.NewAdmissionOptions()` 中又调用了 `genericoptions.NewAdmissionOptions()` 生成。

这样就有三层架构，通用 API Server 层 ==> Kubernetes 特定层 ==> 控制平面选项层。

### 通用 API Server 层

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\options\admission.go#L86-L99
func NewAdmissionOptions() *AdmissionOptions {
	options := &AdmissionOptions{
		Plugins:    admission.NewPlugins(), // 空的插件注册表
		Decorators: admission.Decorators{admission.DecoratorFunc(admissionmetrics.WithControllerMetrics)},
        // 注入 admit 插件
		RecommendedPluginOrder: []string{lifecycle.PluginName, mutatingadmissionpolicy.PluginName, mutatingwebhook.PluginName, validatingadmissionpolicy.PluginName, validatingwebhook.PluginName},
		DefaultOffPlugins:      sets.Set[string]{},
	}
	server.RegisterAllAdmissionPlugins(options.Plugins)
	return options
}
```

通过 `server.RegisterAllAdmissionPlugins` 注册的插件（5个通用插件）：

```go
// 代码:staging\src\k8s.io\apiserver\pkg\server\plugins.go#L29-L36
// RegisterAllAdmissionPlugins registers all admission plugins
func RegisterAllAdmissionPlugins(plugins *admission.Plugins) {
	lifecycle.Register(plugins)
	validatingwebhook.Register(plugins)
	mutatingwebhook.Register(plugins)
	validatingadmissionpolicy.Register(plugins)
	mutatingadmissionpolicy.Register(plugins)
}
```

1. `lifecycle` - 防止在被删除的命名空间中创建资源
2. `mutatingwebhook` - 外部变更 Webhook
3. `validatingwebhook` - 外部验证 Webhook
4. `mutatingadmissionpolicy` - 策略驱动的变更
5. `validatingadmissionpolicy` - 策略驱动的验证

在 `server` 中注册插件后直接返回 options.

这是通用层，任何基于 k8s apiserver 的项目都可以复用。

### Kubernetes 特定层

```go
// 代码: pkg\kubeapiserver\options\admission.go#L54-L66
func NewAdmissionOptions() *AdmissionOptions {
	options := genericoptions.NewAdmissionOptions()
	// register all admission plugins
	RegisterAllAdmissionPlugins(options.Plugins)
	// set RecommendedPluginOrder
	options.RecommendedPluginOrder = AllOrderedPlugins
	// set DefaultOffPlugins
	options.DefaultOffPlugins = DefaultOffAdmissionPlugins()

	return &AdmissionOptions{
		GenericAdmission: options,
	}
}
```

通过 `RegisterAllAdmissionPlugins` 注册的插件：

```go
// 代码: pkg\kubeapiserver\options\plugins.go#L120-L152
// RegisterAllAdmissionPlugins registers all admission plugins.
// The order of registration is irrelevant, see AllOrderedPlugins for execution order.
func RegisterAllAdmissionPlugins(plugins *admission.Plugins) {
	admit.Register(plugins) // DEPRECATED as no real meaning
	alwayspullimages.Register(plugins)
	...
	podtopologylabels.Register(plugins)
}
```

### 控制平面选项层

最后就可以得到：

```go
&Options{
    Admission: kubeoptions.NewAdmissionOptions(),  // 包装好的完整准入配置
    // ... 其他配置项
}
```

这一层只是组装，将 Admission 和其他配置（认证、授权、审计等）组合成完整的 API Server 配置。
