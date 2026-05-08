+++
title = "K8s Generic Server Options To Configs"
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
series_order = 4

+++

# Generic Server Options To Configs

信息流的大致方向为：由启动参数流转至运行选项（Option）、再由运行选项到 Config。

1. 从启动参数到 Option 的主要步骤为：
   * 首先用户输入命令行参数值，它们是利用 Cobra 定义出的标志(flag)；
   * 然后这些标志被转化为 Option 结构体实例；
   * 接着该 Option 实例会用自己的 `Validate()` 方法校验自身内容。
2. 信息由 Option 流转至 Config 则**由各个子 Server 主导，中间会添加子 Server 需要的专有调整**，例如改变 Generic Server 的某些运行配置值和添加子 Server 专有运行配置项。

注意，Option 流转至 Config 由各个子 Server 主导，所以这里以 kube-apiserver 的构建进行介绍，不过只以其为媒介来介绍 Generic Server Options 流转到 Config 的流程，不详细讲解 kube-apiserver 自己的配置。

## 1. Generic Options

1. `Generic Options` 的定义：

   ```go
   // 代码: staging\src\k8s.io\apiserver\pkg\server\options\server_run_options.go#L48
   // ServerRunOptions contains the options while running a generic api server.
   type ServerRunOptions struct {
   	AdvertiseAddress net.IP
   	...
   }
   ```

2. `Generic Options Flag` 的定义：

   ```go
   // 代码: staging\src\k8s.io\apiserver\pkg\server\options\server_run_options.go#L331
   // AddUniversalFlags adds flags for a specific APIServer to the specified FlagSet
   func (s *ServerRunOptions) AddUniversalFlags(fs *pflag.FlagSet) {
   	// Note: the weird ""+ in below lines seems to be the only way to get gofmt to
   	// arrange these text blocks sensibly. Grrr.
   
   	fs.IPVar(&s.AdvertiseAddress, "advertise-address", s.AdvertiseAddress, ""+
       ...
   }
   ```

   定义了这些 Flag 后，子 Server 将该 `AddUniversalFlags` 嵌入后，就可以通过输入命令行参数值得到配置了。

3. 子 Server  嵌入 `AdduniversalFlags` 

   在 `kube-apiserver` 中，先定义了 `kube-apiserver` 的 Options：

   ```go
   // 代码: cmd\kube-apiserver\app\options\options.go#L39
   // ServerRunOptions runs a kubernetes api server.
   type ServerRunOptions struct {
   	*controlplaneapiserver.Options // embedded to avoid noise in existing consumers
   
   	Extra
   }
   ```

   这个结构体『继承』了 `controlplaneapiserver.Options`，在 `kube-apiserver` 中，会定义 `kube-apiserver` 的 `Flags`，代码如下：

   ```go
   // 代码: cmd\kube-apiserver\app\options\options.go#L100
   // Flags returns flags for a specific APIServer by section name
   func (s *ServerRunOptions) Flags() (fss cliflag.NamedFlagSets) {
   	s.Options.AddFlags(&fss)
       ...
   }
   ```

   `kube-apiserver` 会调用 `s.Options.AddFlags`。下面来看看 `s.Options.AddFlags` ：

   ```go
   // 代码: pkg\controlplane\apiserver\options\options.go#L143
   func (s *Options) AddFlags(fss *cliflag.NamedFlagSets) {
   	// Add the generic flags.
   	s.GenericServerRunOptions.AddUniversalFlags(fss.FlagSet("generic"))
       ...
   }
   ```

   在这里就看到了 Generic Server 的 Flags：`AddUniversalFlags`。

在知道了 Flags 的嵌套关系之后，我们就可以通过 `kube-apiserver` 的 Options 的构建流程，来具体看看 Flags 是如何一步步到 Config 的，主要代码位于：`cmd\kube-apiserver\app\server.go` 的 NewAPIServerCommand 函数中：

```go
// NewAPIServerCommand creates a *cobra.Command object with default parameters
func NewAPIServerCommand() *cobra.Command {
	s := options.NewServerRunOptions() // 要点①
	ctx := genericapiserver.SetupSignalContext()
	featureGate := s.GenericServerRunOptions.ComponentGlobalsRegistry.FeatureGateFor(basecompatibility.DefaultKubeComponent)

	cmd := &cobra.Command{
		Use: "kube-apiserver",
		Long: `The Kubernetes API server validates and configures data
for the api objects which include pods, services, replicationcontrollers, and
others. The API Server services REST operations and provides the frontend to the
cluster's shared state through which all other components interact.`,

		// stop printing usage when the command errors
		SilenceUsage: true,
		PersistentPreRunE: func(*cobra.Command, []string) error {
			if err := s.GenericServerRunOptions.ComponentGlobalsRegistry.Set(); err != nil {
				return err
			}
			// silence client-go warnings.
			// kube-apiserver loopback clients should not log self-issued warnings.
			rest.SetDefaultWarningHandler(rest.NoWarnings{})
			return nil
		},
		RunE: func(cmd *cobra.Command, args []string) error {
			verflag.PrintAndExitIfRequested()
			fs := cmd.Flags() // 要点③
			// Activate logging as soon as possible, after that
			// show flags with the final logging configuration.
			if err := logsapi.ValidateAndApply(s.Logs, featureGate); err != nil {
				return err
			}
			cliflag.PrintFlags(fs)

			// set default options
			completedOptions, err := s.Complete(ctx) // 要点④
			if err != nil {
				return err
			}

			// validate options
			if errs := completedOptions.Validate(); len(errs) != 0 { // 要点⑤
				return utilerrors.NewAggregate(errs)
			}
			// add feature enablement metrics
			featureGate.(featuregate.MutableFeatureGate).AddMetrics()
			// add component version metrics
			s.GenericServerRunOptions.ComponentGlobalsRegistry.AddMetrics()
			return Run(ctx, completedOptions)
		},
		Args: func(cmd *cobra.Command, args []string) error {
			for _, arg := range args {
				if len(arg) > 0 {
					return fmt.Errorf("%q does not take any arguments, got %q", cmd.CommandPath(), args)
				}
			}
			return nil
		},
	}
	cmd.SetContext(ctx)

	fs := cmd.Flags()
	namedFlagSets := s.Flags() // 要点②
	s.Flagz = flagz.NamedFlagSetsReader{
		FlagSets: namedFlagSets,
	}
	verflag.AddFlags(namedFlagSets.FlagSet("global"))
	globalflag.AddGlobalFlags(namedFlagSets.FlagSet("global"), cmd.Name(), logs.SkipLoggingConfigurationFlags())
	for _, f := range namedFlagSets.FlagSets {
		fs.AddFlagSet(f)
	}

	cols, _, _ := term.TerminalSize(cmd.OutOrStdout())
	cliflag.SetUsageAndHelpFunc(cmd, namedFlagSets, cols)

	return cmd
}
```

* 在要点①处，通过 `s := options.NewServerRunOptions()` 此结构里嵌有 controlplane 的 Options 和 kube-apiserver 的 Extra Options；

* 在要点②处，调用 `s.Flags` 嵌入的 flag 定义，用于接收用户定义的配置；
* 在要点③处，通过 `fs := cmd.Flags()` 获取 Cobra 命令的 flag 容器。

通过如上三点，flag 就“流入”了 `*cobra.Command`。

然后主要看上面代码定义的 `RunE` 函数，通过要点④的 Complete 和要点⑤的 Validate 就完成了对 Options 的处理。

我们来看看 Complete 和 Validate 的详情：

1. `Complete`

   ```go
   // 代码: cmd\kube-apiserver\app\options\completion.go#L47
   // Complete set default ServerRunOptions.
   // Should be called after kube-apiserver flags parsed.
   func (s *ServerRunOptions) Complete(ctx context.Context) (CompletedOptions, error) {
   	if s == nil {
   		return CompletedOptions{completedOptions: &completedOptions{}}, nil
   	}
   
   	// process s.ServiceClusterIPRange from list to Primary and Secondary
   	// we process secondary only if provided by user
   	apiServerServiceIP, primaryServiceIPRange, secondaryServiceIPRange, err := getServiceIPAndRanges(s.ServiceClusterIPRanges)
   	if err != nil {
   		return CompletedOptions{}, err
   	}
   	controlplane, err := s.Options.Complete(ctx, []string{"kubernetes.default.svc", "kubernetes.default", "kubernetes"}, []net.IP{apiServerServiceIP}) // 要点①
   	if err != nil {
   		return CompletedOptions{}, err
   	}
   
   	completed := completedOptions{
   		CompletedOptions: controlplane,
   
   		Extra: s.Extra,
   	}
       ...
   }
   ```

   主要通过要点①处的 `s.Options.Complete` 得到 `controlplane` 的 Options。再接着看 `s.Options.Complete`：

   ```go
   // 代码: pkg\controlplane\apiserver\options\options.go#L230
   func (o *Options) Complete(ctx context.Context, alternateDNS []string, alternateIPs []net.IP) (CompletedOptions, error) {
   	if o == nil {
   		return CompletedOptions{completedOptions: &completedOptions{}}, nil
   	}
   
   	completed := completedOptions{
   		Options: *o,
   	}
   
   	if err := completed.GenericServerRunOptions.Complete(); err != nil {
   		return CompletedOptions{}, err
   	}
   	...
   }
   ```

   通过 `completed.GenericServerRunOptions.Complete()` 调用 `Generic Server` 的 `Options` 的 `Complete` 函数。

   `Generic Server` 的 `Options` 的 `Complete` 函数定义如下：

   ```go
   // 代码: staging\src\k8s.io\apiserver\pkg\server\options\server_run_options.go#L412
   // Complete fills missing fields with defaults.
   func (s *ServerRunOptions) Complete() error {
   	return s.ComponentGlobalsRegistry.SetFallback()
   }
   ```

2. `Validate`

   ```go
   // 代码: cmd\kube-apiserver\app\options\validation.go#L134
   // Validate checks ServerRunOptions and return a slice of found errs.
   func (s CompletedOptions) Validate() []error {
   	var errs []error
   
   	errs = append(errs, s.CompletedOptions.Validate()...)
   	errs = append(errs, validateClusterIPFlags(s.Extra)...)
   	errs = append(errs, validateServiceNodePort(s.Extra)...)
   	errs = append(errs, validatePublicIPServiceClusterIPRangeIPFamilies(s.Extra, *s.GenericServerRunOptions)...)
   
   	if s.MasterCount <= 0 {
   		errs = append(errs, fmt.Errorf("--apiserver-count should be a positive number, but value '%d' provided", s.MasterCount))
   	}
   
   	return errs
   }
   ```

   通过 `errs = append(errs, s.CompletedOptions.Validate()...)` 来调用 `Options` 的 `Validate`，该 `Validate` 定义如下：

   ```go
   // 代码: pkg\controlplane\apiserver\options\validation.go#L124
   // Validate checks Options and return a slice of found errs.
   func (s *Options) Validate() []error {
   	var errs []error
   
   	errs = append(errs, s.GenericServerRunOptions.Validate()...)
   	errs = append(errs, s.Etcd.Validate()...)
   	errs = append(errs, validateAPIPriorityAndFairness(s)...)
   	errs = append(errs, s.SecureServing.Validate()...)
   	errs = append(errs, s.Authentication.Validate()...)
   	errs = append(errs, s.Authorization.Validate()...)
   	errs = append(errs, s.Audit.Validate()...)
   	errs = append(errs, s.Admission.Validate()...)
   	errs = append(errs, s.APIEnablement.Validate(legacyscheme.Scheme, apiextensionsapiserver.Scheme, aggregatorscheme.Scheme)...)
   	errs = append(errs, validateTokenRequest(s)...)
   	errs = append(errs, s.Metrics.Validate()...)
   	errs = append(errs, validateUnknownVersionInteroperabilityProxyFlags(s)...)
   	errs = append(errs, validateServiceAccountTokenSigningConfig(s)...)
   	errs = append(errs, validateCoordinatedLeadershipFlags(s)...)
   
   	return errs
   }
   ```

   这个会通过 `errs = append(errs, s.GenericServerRunOptions.Validate()...)` 来调用 `Generic Server` 的 `Options` 的 `Validate`：

   ```go
   // 代码: staging\src\k8s.io\apiserver\pkg\server\options\server_run_options.go
   // Validate checks validation of ServerRunOptions
   func (s *ServerRunOptions) Validate() []error {
   	errors := []error{}
   
   	if s.LivezGracePeriod < 0 {
   		errors = append(errors, fmt.Errorf("--livez-grace-period can not be a negative value"))
   	}
   
   	if s.MaxRequestsInFlight < 0 {
   		errors = append(errors, fmt.Errorf("--max-requests-inflight can not be negative value"))
   	}
   	if s.MaxMutatingRequestsInFlight < 0 {
   		errors = append(errors, fmt.Errorf("--max-mutating-requests-inflight can not be negative value"))
   	}
   	...
   }
   ```

通过上述的流程，就会得到完整的，经过验证的 `Options`。

> 走一遍上述流程你就会发现，这里有三个 Options，
>
> 1. **Generic Server Options（staging/k8s.io/apiserver/.../ServerRunOptions）**
>    定义最基础的 Generic APIServer 运行参数（并发、超时、地址等），这是所有使用 generic-apiserver 框架的组件都要共用的底座配置。
> 2. **controlplane Options（pkg/controlplane/apiserver/options.Options）**
>    这是 kube controlplane（kube-apiserver、aggregator、apiextensions 等）共享的“组合包”。它内嵌 Generic Server Options，并额外打包 etcd、认证授权、审计、feature、Tracing、Metrics 等 controlplane 通用子模块。controlplane Options 负责把“generic + controlplane 通用需求”准备成可直接构建 Generic Server 的配置。
> 3. **kube-apiserver ServerRunOptions（cmd/kube-apiserver/app/options/...）**
>    再往上包一层，把 controlplane Options 嵌进去，同时加上 kube-apiserver 专属的 Extra（ServiceClusterIPRanges、KubeletConfig、EndpointReconciler 等）。

## 2. Generic Config

在 kube-apiserver 的构建 Command 中，`RunE` 会返回 `Run(ctx, completedOptions)`

```go
// 代码: cmd\kube-apiserver\app\server.go#L148
// Run runs the specified APIServer.  This should never exit.
func Run(ctx context.Context, opts options.CompletedOptions) error {
	// To help debugging, immediately log version
	klog.Infof("Version: %+v", utilversion.Get())

	klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

	config, err := NewConfig(opts) // 要点①
	if err != nil {
		return err
	}
	completed, err := config.Complete() // 要点②
	if err != nil {
		return err
	}
	server, err := CreateServerChain(completed)
	if err != nil {
		return err
	}

	prepared, err := server.PrepareRun()
	if err != nil {
		return err
	}

	return prepared.Run(ctx)
}
```

这里会使用 `CompletedOptions` 来生成 Config，先通过 `NewConfig` 生成 `config`，然后调用 `Complete`，生成 `completedConfig`。

1. `NewConfig`

   ```go
   // 代码: cmd\kube-apiserver\app\config.go#L74
   // NewConfig creates all the resources for running kube-apiserver, but runs none of them.
   func NewConfig(opts options.CompletedOptions) (*Config, error) {
   	c := &Config{
   		Options: opts,
   	}
   
   	genericConfig, versionedInformers, storageFactory, err := controlplaneapiserver.BuildGenericConfig( // 要点①
   		opts.CompletedOptions,
   		[]*runtime.Scheme{legacyscheme.Scheme, apiextensionsapiserver.Scheme, aggregatorscheme.Scheme},
   		controlplane.DefaultAPIResourceConfigSource(),
   		generatedopenapi.GetOpenAPIDefinitions,
   	)
   	if err != nil {
   		return nil, err
   	}
       ...
   }
   ```

   要点①，这里会通过 `BuildGenericConfig`，来生成 `genericConfig`，`BuildGenericConfig` 的定义如下：

   ```go
   // 代码: pkg\controlplane\apiserver\config.go#L116
   // BuildGenericConfig takes the generic controlplane apiserver options and produces
   // the genericapiserver.Config associated with it. The genericapiserver.Config is
   // often shared between multiple delegated apiservers.
   func BuildGenericConfig(
   	s options.CompletedOptions,
   	schemes []*runtime.Scheme,
   	resourceConfig *serverstorage.ResourceConfig,
   	getOpenAPIDefinitions func(ref openapicommon.ReferenceCallback) map[string]openapicommon.OpenAPIDefinition,
   ) (
   	genericConfig *genericapiserver.Config, // 要点①
   	versionedInformers clientgoinformers.SharedInformerFactory,
   	storageFactory *serverstorage.DefaultStorageFactory,
   	lastErr error,
   ) {
   	genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)
   	genericConfig.Flagz = s.Flagz
   	genericConfig.MergedResourceConfig = resourceConfig
   
   	if lastErr = s.GenericServerRunOptions.ApplyTo(genericConfig); lastErr != nil { // 要点②
   		return
   	}
       ...
   }
   ```

   这里首先通过要点①创建 genericConfig 的空壳，然后通过要点②处的 `ApplyTo` 函数，通过 `Options` 来填充 `Config`，这样就可以得到 Config。

2. `Complete`

   ```go
   // 代码: cmd\kube-apiserver\app\config.go#L61
   func (c *Config) Complete() (CompletedConfig, error) {
   	return CompletedConfig{&completedConfig{
   		Options: c.Options,
   
   		Aggregator:    c.Aggregator.Complete(),
   		KubeAPIs:      c.KubeAPIs.Complete(),
   		ApiExtensions: c.ApiExtensions.Complete(),
   
   		ExtraConfig: c.ExtraConfig,
   	}}, nil
   }
   ```

   通过 `KubeAPIs: c.KubeAPIs.Complete()`，是 controlplane（核心 kube-apiserver）那部分配置，同样调用 `.Complete()`， 让底层 Generic Config（及 controlplane Extra）进入完成态。查看该 `c.KubeAPIs.Complete()` 函数：

   ```go
   // 代码: pkg\controlplane\instance.go#L255
   // Complete fills in any fields not set that are required to have valid data. It's mutating the receiver.
   func (c *Config) Complete() CompletedConfig {
   	if c.ControlPlane.PeerEndpointReconcileInterval == 0 && c.EndpointReconcilerConfig.Interval != 0 {
   		// default this to the endpoint reconciler value before the generic
   		// controlplane completion can kick in
   		c.ControlPlane.PeerEndpointReconcileInterval = c.EndpointReconcilerConfig.Interval
   	}
   
   	cfg := completedConfig{
   		c.ControlPlane.Complete(), // 要点①
   		&c.Extra,
   	}
   	...
   }
   ```

   在要点①处，通过 `c.ControlPlane.Complete()` 来调用让 `controlPlane` 的 Config 进入完成态。再来查看该 `c.ControlPlane.Complete()` 应该就可以看见 Generic Config 的 Complete 函数了：

   ```go
   // 代码: pkg\controlplane\apiserver\completion.go
   func (c *Config) Complete() CompletedConfig {
   	cfg := completedConfig{
   		c.Generic.Complete(c.VersionedInformers), // 要点①
   		&c.Extra,
   	}
   
   	discoveryAddresses := discovery.DefaultAddresses{DefaultAddress: cfg.Generic.ExternalAddress}
   	cfg.Generic.DiscoveryAddresses = discoveryAddresses
   
   	if cfg.Extra.PeerEndpointReconcileInterval == 0 {
   		cfg.Extra.PeerEndpointReconcileInterval = DefaultPeerEndpointReconcileInterval
   	}
   
   	return CompletedConfig{&cfg}
   }
   ```

   果然，在要点①处就调用了 Generic Config 的 Complete 函数。

   该 Complete 函数如下：

   ```go
   // 代码: staging\src\k8s.io\apiserver\pkg\server\config.go#L700
   // Complete fills in any fields not set that are required to have valid data and can be derived
   // from other fields. If you're going to `ApplyOptions`, do that first. It's mutating the receiver.
   func (c *Config) Complete(informers informers.SharedInformerFactory) CompletedConfig {
   	if c.FeatureGate == nil {
   		c.FeatureGate = utilfeature.DefaultFeatureGate
   	}
   	if len(c.ExternalAddress) == 0 && c.PublicAddress != nil {
   		c.ExternalAddress = c.PublicAddress.String()
   	}
       ...
   }
   ```

   然后也可以看到 Generic Config 的定义：

   ```go
   // 代码: staging\src\k8s.io\apiserver\pkg\server\config.go#L109
   // Config is a structure used to configure a GenericAPIServer.
   // Its members are sorted roughly in order of importance for composers.
   type Config struct {
   	// SecureServing is required to serve https
   	SecureServing *SecureServingInfo
   
   	// Authentication is the configuration for authentication
   	Authentication AuthenticationInfo
   
   	// Authorization is the configuration for authorization
   	Authorization AuthorizationInfo
       ...
   }
   ```

再次回到 Run 函数：

```go
// 代码: cmd\kube-apiserver\app\server.go
// Run runs the specified APIServer.  This should never exit.
func Run(ctx context.Context, opts options.CompletedOptions) error {
	// To help debugging, immediately log version
	klog.Infof("Version: %+v", utilversion.Get())

	klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

	config, err := NewConfig(opts)
	if err != nil {
		return err
	}
	completed, err := config.Complete()
	if err != nil {
		return err
	}
	server, err := CreateServerChain(completed)
	if err != nil {
		return err
	}

	prepared, err := server.PrepareRun()
	if err != nil {
		return err
	}

	return prepared.Run(ctx)
}
```

经过上述的过程之后，就得到了 `completed`，然后就可以用 `completed` 来进行后续步骤。
