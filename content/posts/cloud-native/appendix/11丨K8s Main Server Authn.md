+++
title = "K8s Main Server Authn"
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

series_order = 14

+++

# K8s Main Server Authn

Kubernetes 提供了如此丰富的登录验证策略，那么它们在登录验证过程中如何和谐共生呢？API Server 的登录验证机制基于插件化的思想，将各个策略分别构建成不同登录插件，在进行验证时逐个调用，只要有一个登录插件成功认证了请求用户，则登录成功。

> NOTICE:
>
> 通过与否的准则与准入控制机制不同：准入控制机制下一个请求只要被一个准入控制器拒绝，请求就失败了；而登录认证是只需在一种方式下成功便通过。

API Server 登录认证机制的构建过程如下图所示。

![image-20251210195614031](http://images.liangning7.cn/typora/202512101956090.png)

从配置信息流转上看与准入控制机制的构建过程十分类似。但它与准入控制机制也有重要不同，登录认证是在过滤器中实现的。回顾 Generic Server 的创建过程，在构建请求过滤器链时有一个名为 Authentication 的过滤器，其作用就是提供登录认证服务。过滤器在请求刚刚进入 API Server 就会被执行，这的确是登录、鉴权以及其它安全保护逻辑理想的触发时机。

## 运行选项和命令行参数

API Server 在启动时第一项工作就是做一个类型为 ServerRunOptions 的变量，它是后续制作可用命令行参数和 Server 运行配置的数据来源，需要搞清楚这个数据源中关于登录认证的信息是怎么得来的。以 `cmd/kube-apiserver/app/server.go` 源文件中 API Server 启动命令生成方法 `NewAPIServerCommand()` 为入口，找到该变量是经如下方法制作并返回的：

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
		Admission:               kubeoptions.NewAdmissionOptions(),
        // 要点①
		Authentication:          kubeoptions.NewBuiltInAuthenticationOptions().WithAll(),
        ...
    }
    ...
}
```

要点①表明，用户认证相关选项数据是 `NewBuiltInAuthenticationOptions()` 方法，以及对其返回值的 WithAll()方法调用得来的。技术上说，在这之后还有可能对Authentication 内信息做进一步修改，但实际上 Authentication 属性的内容后续就不会修改了，只要搞清楚 `NewBuiltInAuthenticationOptions()` 和 `WithAll()` 方法，就清楚了登录认证配置从何而来。

```go
// 代码: pkg\kubeapiserver\options\authentication.go#L163-L182
// NewBuiltInAuthenticationOptions create a new BuiltInAuthenticationOptions, just set default token cache TTL
func NewBuiltInAuthenticationOptions() *BuiltInAuthenticationOptions {
	return &BuiltInAuthenticationOptions{
		TokenSuccessCacheTTL: 10 * time.Second,
		TokenFailureCacheTTL: 0 * time.Second,
	}
}

// WithAll set default value for every build-in authentication option
func (o *BuiltInAuthenticationOptions) WithAll() *BuiltInAuthenticationOptions {
	return o.
		WithAnonymous().
		WithBootstrapToken().
		WithClientCert().
		WithOIDC().
		WithRequestHeader().
		WithServiceAccounts().
		WithTokenFile().
		WithWebHook()
}
```

`NewBuiltInAuthenticationOptions` 方法做了一个类型为 `BuiltInAuthenticationOptions` 结构体实例并返回。

`WithAll` 方法向接收者——也就是 `NewBuiltInAuthenticationOptions` 的返回值添加各种登录认证插件，包括：

1. 匿名登录认证策略：由 `WithAnonymous()` 方法加入。
2. 启动引导认证策略：由 `WithBootstrapToken()` 方法加入。
3. X509 证书认证策略：由 `WithClientCert()` 方法加入。
4. OpenID Connect 认证策略：由 `WithOIDC()` 方法加入。
5. 代理认证策略：由 `WithRequestHeader()` 方法加入。
6. Service Account 认证策略：由 `WithServiceAccounts()` 方法加入。
7. 静态令牌验认证略：由 `WithTokenFile()` 方法加入。
8. Webhook 验认证略：由 `WithWebHook()` 方法加入。

以上就是运行选项中 Authentication 属性信息的来源。

这一信息在运行选项的 Complete [补全] 阶段会被做一个小修改：禁止匿名登录策略。运行选项补全发生在 `Complete()` 方法中，这里 Authentication 属性的 `ApplyAuthorization()` 方法会被调到，而它只做了一件事情：在一定条件下禁止匿名登录策略，调用链如下：

```go
// 代码: cmd\kube-apiserver\app\options\completion.go#L47-L92
// Complete set default ServerRunOptions.
// Should be called after kube-apiserver flags parsed.
func (s *ServerRunOptions) Complete(ctx context.Context) (CompletedOptions, error) {
	...
	controlplane, err := s.Options.Complete(ctx, []string{"kubernetes.default.svc", "kubernetes.default", "kubernetes"}, []net.IP{apiServerServiceIP})
	if err != nil {
		return CompletedOptions{}, err
	}
    ...
}
// ⬇
// 调用 s.Options.Complete
// 代码: pkg\controlplane\apiserver\options\options.go#L221-290
func (o *Options) Complete(ctx context.Context, alternateDNS []string, alternateIPs []net.IP) (CompletedOptions, error) {
	...
    // put authorization options in final state
	completed.Authorization.Complete()
	// adjust authentication for completed authorization
	completed.Authentication.ApplyAuthorization(completed.Authorization)
    ...
}

// ⬇
// 代码: pkg\kubeapiserver\options\authentication.go#L807-L818
// ApplyAuthorization will conditionally modify the authentication options based on the authorization options
func (o *BuiltInAuthenticationOptions) ApplyAuthorization(authorization *BuiltInAuthorizationOptions) {
	if o == nil || authorization == nil || o.Anonymous == nil {
		return
	}

	// authorization ModeAlwaysAllow cannot be combined with AnonymousAuth.
	// in such a case the AnonymousAuth is stomped to false and you get a message
	if o.Anonymous.Allow && sets.NewString(authorization.Modes...).Has(authzmodes.ModeAlwaysAllow) {
		klog.Warningf("AnonymousAuth is not allowed with the AlwaysAllow authorizer. Resetting AnonymousAuth to false. You should use a different authorizer")
		o.Anonymous.Allow = false
	}
}
```

大多数登录认证插件都有自己的配置，例如 OpenID Connect 策略需要设置校验 `JWT` 凭据时用到的密钥或证书，这些配置需要管理员在启动 API Server 时通过命令行参数设置；另外通过命令行参数也可以启用或禁用一些登录认证插件。这些命令行参数都由 `ServerRunOptions.Authentication` 字段来承载。该字段类型为 `BuiltInAuthenticationOptions` 结构体。

```go
// 代码: cmd\kube-apiserver\app\options\options.go#L39-L43
// ServerRunOptions runs a kubernetes api server.
type ServerRunOptions struct {
	*controlplaneapiserver.Options // embedded to avoid noise in existing consumers

	Extra
}
```

它具有方法 `AddFlags()`，这个方法把所有可用登录插件的命令行参数加入到 Cobra 框架中，

```go
// 代码: cmd\kube-apiserver\app\options\options.go#L100-L157
// Flags returns flags for a specific APIServer by section name
func (s *ServerRunOptions) Flags() (fss cliflag.NamedFlagSets) {
	s.Options.AddFlags(&fss)
    ...
}

// 调用 s.Options.AddFlags
// ⬇
// 代码: pkg\controlplane\apiserver\options\options.go#L143-L219
func (s *Options) AddFlags(fss *cliflag.NamedFlagSets) {
	// Add the generic flags.
	s.GenericServerRunOptions.AddUniversalFlags(fss.FlagSet("generic"))
	s.Etcd.AddFlags(fss.FlagSet("etcd"))
	s.SecureServing.AddFlags(fss.FlagSet("secure serving"))
	s.Audit.AddFlags(fss.FlagSet("auditing"))
	s.Features.AddFlags(fss.FlagSet("features"))
	s.Authentication.AddFlags(fss.FlagSet("authentication"))
    ...
}
// 调用 s.Authentication.AddFlags
// ⬇
// pkg/kubeapiserver/options/authentication.go#L316-L463
func (o *BuiltInAuthenticationOptions) AddFlags(fs *pflag.FlagSet) {
    if o == nil {
        return
    }

    // 注册各个认证插件的命令行参数
    
    // 1. 匿名认证
    if o.Anonymous != nil {
        fs.BoolVar(&o.Anonymous.Allow, "anonymous-auth", o.Anonymous.Allow,
            "Enables anonymous requests to the secure port of the API server...")
    }

    // 2. Bootstrap Token
    if o.BootstrapToken != nil {
        fs.BoolVar(&o.BootstrapToken.Enable, "enable-bootstrap-token-auth", ...)
    }

    // 3. 客户端证书
    if o.ClientCert != nil {
        o.ClientCert.AddFlags(fs)  // --client-ca-file
    }

    // 4. OIDC
    if o.OIDC != nil {
        fs.StringVar(&o.OIDC.IssuerURL, "oidc-issuer-url", ...)
        fs.StringVar(&o.OIDC.ClientID, "oidc-client-id", ...)
        fs.StringVar(&o.OIDC.CAFile, "oidc-ca-file", ...)
        fs.StringVar(&o.OIDC.UsernameClaim, "oidc-username-claim", ...)
        fs.StringVar(&o.OIDC.UsernamePrefix, "oidc-username-prefix", ...)
        fs.StringVar(&o.OIDC.GroupsClaim, "oidc-groups-claim", ...)
        fs.StringVar(&o.OIDC.GroupsPrefix, "oidc-groups-prefix", ...)
        fs.StringSliceVar(&o.OIDC.SigningAlgs, "oidc-signing-algs", ...)
        fs.Var(&o.OIDC.RequiredClaims, "oidc-required-claim", ...)
    }

    // 5. 请求头认证
    if o.RequestHeader != nil {
        o.RequestHeader.AddFlags(fs)  // --requestheader-*
    }

    // 6. ServiceAccount
    if o.ServiceAccounts != nil {
        fs.StringArrayVar(&o.ServiceAccounts.KeyFiles, "service-account-key-file", ...)
        fs.BoolVar(&o.ServiceAccounts.Lookup, "service-account-lookup", ...)
        fs.StringArrayVar(&o.ServiceAccounts.Issuers, "service-account-issuer", ...)
        fs.StringVar(&o.ServiceAccounts.JWKSURI, "service-account-jwks-uri", ...)
        fs.DurationVar(&o.ServiceAccounts.MaxExpiration, "service-account-max-token-expiration", ...)
        fs.BoolVar(&o.ServiceAccounts.ExtendExpiration, "service-account-extend-token-expiration", ...)
    }

    // 7. Token 文件
    if o.TokenFile != nil {
        fs.StringVar(&o.TokenFile.TokenFile, "token-auth-file", ...)
    }

    // 8. Webhook
    if o.WebHook != nil {
        fs.StringVar(&o.WebHook.ConfigFile, "authentication-token-webhook-config-file", ...)
        fs.StringVar(&o.WebHook.Version, "authentication-token-webhook-version", ...)
        fs.DurationVar(&o.WebHook.CacheTTL, "authentication-token-webhook-cache-ttl", ...)
    }
}
```

Cobra 负责把用户的相关输入赋值给 `ServerRunOptions.Authentication`。

## 从运行选项到运行配置

运行选项结构体（ServerRunOptions）面向命令行，负责用组织包含命令行输入信息在内的所有选项配置信息；而主 Server 运行配置结构体(`controlplan.Config`)面向 Server，选项信息是它的主要信息来源，辅以一些其它逻辑决定的信息。登录认证信息也有一个从运行选项到运行配置转移的一个过程。

登录认证机制完全是由 `Generic Server` 提供的，主 Server 直接把这部分工作交给自己的底座 Generic Server；类似地，主 Server 的运行配置结构体通过 Generic Server 的运行配置结构体（`genericapiserver.Config`）代持 Authentication 信息，代码上可看到 `controlplan.Config` 直接定义了一个属性 GenericConfig 来嵌入 `Generic Server` 运行配置：

```go
// 代码: pkg\controlplane\apiserver\config.go#L65-L68
// Config defines configuration for the master
type Config struct {
	Generic *genericapiserver.Config
	Extra
}
```

登录认证参数从命令行选项到运行参数的转移过程与准入控制参数的过程如出一辙，这里重温一下。方法 `buildGenericConfig()` 以之前得到的 `ServerRunOptions` 结构体实例为入参，构造一个 Generic Server 的 Config 结构体实例；这个 Config 结构体实例会成为主 Server 运行配置的一部分——也就是前面看到的 `controlplan.Config` 结构体的 GenericConfig 字段。

```markdown
┌─────────────────────────────────────────────────────────────────────────────┐
│                          命令行参数解析阶段                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  $ kube-apiserver --anonymous-auth=true \                                   │
│                   --oidc-issuer-url=https://issuer.example.com \            │
│                   --service-account-key-file=/path/to/sa.key                │
│                                                                             │
│  pflag 通过指针直接修改 ServerRunOptions.Authentication 的字段值              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                    第一步：Run() 函数入口                                      │
│                    cmd/kube-apiserver/app/server.go:147                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  func Run(ctx context.Context, opts options.CompletedOptions) error {      │
│      config, err := NewConfig(opts)  ────────────────┐                     │
│      completed, err := config.Complete()             │                     │
│      server, err := CreateServerChain(completed)     │                     │
│      prepared, err := server.PrepareRun()            │                     │
│      return prepared.Run(ctx)                        │                     │
│  }                                                   │                     │
└──────────────────────────────────────────────────────┼─────────────────────┘
                                                       │
                                                       ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                    第二步：NewConfig() 创建配置                                │
│                    cmd/kube-apiserver/app/config.go:74                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  func NewConfig(opts options.CompletedOptions) (*Config, error) {          │
│      c := &Config{Options: opts}                                           │
│                                                                             │
│      genericConfig, versionedInformers, storageFactory, err :=             │
│          controlplaneapiserver.BuildGenericConfig(  ─────────┐             │
│              opts.CompletedOptions,                          │             │
│              schemes,                                        │             │
│              resourceConfig,                                 │             │
│              getOpenAPIDefinitions,                          │             │
│          )                                                   │             │
│                                                              │             │
│      kubeAPIs, serviceResolver, pluginInitializer, err :=    │             │
│          CreateKubeAPIServerConfig(opts, genericConfig, ...) │             │
│                                                              │             │
│      return c, nil                                           │             │
│  }                                                           │             │
└──────────────────────────────────────────────────────────────┼─────────────┘
                                                               │
                                                               ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│              第三步：BuildGenericConfig() 构建通用配置                          │
│              pkg/controlplane/apiserver/config.go:115                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  func BuildGenericConfig(s options.CompletedOptions, ...) (...) {          │
│      genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)       │
│                                                                             │
│      // 应用各种配置                                                         │
│      s.GenericServerRunOptions.ApplyTo(genericConfig)                      │
│      s.SecureServing.ApplyTo(&genericConfig.SecureServing, ...)            │
│      s.Features.ApplyTo(genericConfig, ...)                                │
│      s.APIEnablement.ApplyTo(genericConfig, ...)                           │
│      s.EgressSelector.ApplyTo(genericConfig)                               │
│      s.Traces.ApplyTo(genericConfig.EgressSelector, genericConfig)         │
│                                                                             │
│      // 🔥 关键：应用认证配置                                                │
│      s.Authentication.ApplyTo(  ─────────────────────────┐                 │
│          ctx,                                            │                 │
│          &genericConfig.Authentication,  // 目标配置      │                 │
│          genericConfig.SecureServing,                    │                 │
│          genericConfig.EgressSelector,                   │                 │
│          genericConfig.OpenAPIConfig,                    │                 │
│          genericConfig.OpenAPIV3Config,                  │                 │
│          clientgoExternalClient,                         │                 │
│          versionedInformers,                             │                 │
│          genericConfig.APIServerID,                      │                 │
│      )                                                   │                 │
│                                                          │                 │
│      return genericConfig, versionedInformers, storageFactory, nil         │
│  }                                                       │                 │
└──────────────────────────────────────────────────────────┼─────────────────┘
                                                           │
                                                           ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│           第四步：Authentication.ApplyTo() 应用认证配置                         │
│           pkg/kubeapiserver/options/authentication.go:647                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  func (o *BuiltInAuthenticationOptions) ApplyTo(                           │
│      ctx context.Context,                                                  │
│      authInfo *genericapiserver.AuthenticationInfo,  // 目标               │
│      ...,                                                                  │
│  ) error {                                                                 │
│      // 步骤 4.1：转换选项为配置                                             │
│      authenticatorConfig, err := o.ToAuthenticationConfig()  ──────┐       │
│      if err != nil {                                               │       │
│          return err                                                │       │
│      }                                                             │       │
│                                                                    │       │
│      // 步骤 4.2：应用客户端证书配置                                 │       │
│      if authenticatorConfig.ClientCAContentProvider != nil {       │       │
│          authInfo.ApplyClientCert(...)                             │       │
│      }                                                             │       │
│                                                                    │       │
│      // 步骤 4.3：设置请求头配置                                     │       │
│      authInfo.RequestHeaderConfig = authenticatorConfig.RequestHeaderConfig│
│      authInfo.APIAudiences = o.APIAudiences                        │       │
│                                                                    │       │
│      // 步骤 4.4：设置 ServiceAccount Token Getter                 │       │
│      authenticatorConfig.ServiceAccountTokenGetter =               │       │
│          serviceaccountcontroller.NewGetterFromClient(...)         │       │
│                                                                    │       │
│      // 步骤 4.5：设置 Bootstrap Token 认证器                       │       │
│      if authenticatorConfig.BootstrapToken {                       │       │
│          authenticatorConfig.BootstrapTokenAuthenticator =         │       │
│              bootstrap.NewTokenAuthenticator(...)                  │       │
│      }                                                             │       │
│                                                                    │       │
│      // 步骤 4.6：创建实际的认证器                                   │       │
│      authenticator, updateFunc, openAPIV2Defs, openAPIV3Schemes, err :=    │
│          authenticatorConfig.New(ctx)  ────────────────────────┐   │       │
│      if err != nil {                                           │   │       │
│          return err                                            │   │       │
│      }                                                         │   │       │
│                                                                │   │       │
│      // 步骤 4.7：赋值给服务器配置                               │   │       │
│      authInfo.Authenticator = authenticator  // 🎯 最终赋值     │   │       │
│                                                                │   │       │
│      return nil                                                │   │       │
│  }                                                             │   │       │
└────────────────────────────────────────────────────────────────┼───┼───────┘
                                                                 │   │
                    ┌────────────────────────────────────────────┘   │
                    ↓                                                │
┌─────────────────────────────────────────────────────────────────────────────┐
│        第五步：ToAuthenticationConfig() 转换选项为配置                          │
│        pkg/kubeapiserver/options/authentication.go:465                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  func (o *BuiltInAuthenticationOptions) ToAuthenticationConfig()           │
│      (kubeauthenticator.Config, error) {                                   │
│                                                                             │
│      ret := kubeauthenticator.Config{                                      │
│          TokenSuccessCacheTTL: o.TokenSuccessCacheTTL,                     │
│          TokenFailureCacheTTL: o.TokenFailureCacheTTL,                     │
│      }                                                                     │
│                                                                             │
│      // 转换 Bootstrap Token                                               │
│      if o.BootstrapToken != nil {                                          │
│          ret.BootstrapToken = o.BootstrapToken.Enable                     │
│      }                                                                     │
│                                                                             │
│      // 转换客户端证书                                                       │
│      if o.ClientCert != nil {                                              │
│          ret.ClientCAContentProvider, _ =                                  │
│              o.ClientCert.GetClientCAContentProvider()                     │
│      }                                                                     │
│                                                                             │
│      // 转换 OIDC 配置                                                      │
│      if o.OIDC != nil && len(o.OIDC.IssuerURL) > 0 {                      │
│          jwtAuthenticator := apiserver.JWTAuthenticator{                   │
│              Issuer: apiserver.Issuer{                                     │
│                  URL:       o.OIDC.IssuerURL,                             │
│                  Audiences: []string{o.OIDC.ClientID},                    │
│              },                                                            │
│              ClaimMappings: apiserver.ClaimMappings{                       │
│                  Username: apiserver.PrefixedClaimOrExpression{            │
│                      Prefix: ptr.To(usernamePrefix),                       │
│                      Claim:  o.OIDC.UsernameClaim,                        │
│                  },                                                        │
│              },                                                            │
│          }                                                                 │
│          ret.AuthenticationConfig.JWT = []apiserver.JWTAuthenticator{      │
│              jwtAuthenticator,                                             │
│          }                                                                 │
│          ret.OIDCSigningAlgs = o.OIDC.SigningAlgs                         │
│      }                                                                     │
│                                                                             │
│      // 转换匿名认证                                                         │
│      if o.Anonymous != nil {                                               │
│          ret.Anonymous = apiserver.AnonymousAuthConfig{                    │
│              Enabled: o.Anonymous.Allow,                                   │
│          }                                                                 │
│      }                                                                     │
│                                                                             │
│      // 转换请求头认证                                                       │
│      if o.RequestHeader != nil {                                           │
│          ret.RequestHeaderConfig, _ =                                      │
│              o.RequestHeader.ToAuthenticationRequestHeaderConfig()         │
│      }                                                                     │
│                                                                             │
│      // 转换 ServiceAccount 配置                                            │
│      if o.ServiceAccounts != nil {                                         │
│          ret.ServiceAccountIssuers = o.ServiceAccounts.Issuers            │
│          ret.ServiceAccountLookup = o.ServiceAccounts.Lookup              │
│          // 加载公钥文件                                                     │
│          for _, keyFile := range o.ServiceAccounts.KeyFiles {              │
│              publicKeys, _ := keyutil.PublicKeysFromFile(keyFile)         │
│              ret.ServiceAccountPublicKeysGetter =                          │
│                  serviceaccount.StaticPublicKeysGetter(publicKeys)        │
│          }                                                                 │
│      }                                                                     │
│                                                                             │
│      // 转换 Token 文件                                                     │
│      if o.TokenFile != nil {                                               │
│          ret.TokenAuthFile = o.TokenFile.TokenFile                        │
│      }                                                                     │
│                                                                             │
│      // 转换 Webhook                                                        │
│      if o.WebHook != nil {                                                 │
│          ret.WebhookTokenAuthnConfigFile = o.WebHook.ConfigFile           │
│          ret.WebhookTokenAuthnVersion = o.WebHook.Version                 │
│          ret.WebhookTokenAuthnCacheTTL = o.WebHook.CacheTTL               │
│      }                                                                     │
│                                                                             │
│      return ret, nil                                                       │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                 返回 kubeauthenticator.Config                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  type Config struct {                                                      │
│      Anonymous                      apiserver.AnonymousAuthConfig          │
│      BootstrapToken                 bool                                   │
│      TokenAuthFile                  string                                 │
│      AuthenticationConfig           *apiserver.AuthenticationConfiguration │
│      OIDCSigningAlgs                []string                               │
│      ServiceAccountLookup           bool                                   │
│      ServiceAccountIssuers          []string                               │
│      ServiceAccountPublicKeysGetter serviceaccount.PublicKeysGetter        │
│      WebhookTokenAuthnConfigFile    string                                 │
│      RequestHeaderConfig            *RequestHeaderConfig                   │
│      TokenSuccessCacheTTL           time.Duration                          │
│      TokenFailureCacheTTL           time.Duration                          │
│      // ... 其他字段                                                        │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│          第六步：authenticatorConfig.New() 创建实际认证器                       │
│          pkg/kubeapiserver/authenticator/config.go:103                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  func (config Config) New(ctx context.Context) (                           │
│      authenticator.Request,                                                │
│      updateFunc,                                                           │
│      *spec.SecurityDefinitions,                                            │
│      spec3.SecuritySchemes,                                                │
│      error,                                                                │
│  ) {                                                                       │
│      var authenticators []authenticator.Request                            │
│      var tokenAuthenticators []authenticator.Token                         │
│                                                                             │
│      // 1. 添加请求头认证器                                                  │
│      if config.RequestHeaderConfig != nil {                                │
│          requestHeaderAuthenticator :=                                     │
│              headerrequest.NewDynamicVerifyOptionsSecure(...)              │
│          authenticators = append(authenticators,                           │
│              authenticator.WrapAudienceAgnosticRequest(                    │
│                  config.APIAudiences,                                      │
│                  requestHeaderAuthenticator,                               │
│              ))                                                            │
│      }                                                                     │
│                                                                             │
│      // 2. 添加 X509 客户端证书认证器                                        │
│      if config.ClientCAContentProvider != nil {                            │
│          certAuth := x509.NewDynamic(                                      │
│              config.ClientCAContentProvider.VerifyOptions,                 │
│              x509.CommonNameUserConversion,                                │
│          )                                                                 │
│          authenticators = append(authenticators, certAuth)                 │
│      }                                                                     │
│                                                                             │
│      // 3. 添加 Token 文件认证器                                            │
│      if len(config.TokenAuthFile) > 0 {                                    │
│          tokenAuth, _ := newAuthenticatorFromTokenFile(                    │
│              config.TokenAuthFile,                                         │
│          )                                                                 │
│          tokenAuthenticators = append(tokenAuthenticators,                 │
│              authenticator.WrapAudienceAgnosticToken(                      │
│                  config.APIAudiences,                                      │
│                  tokenAuth,                                                │
│              ))                                                            │
│      }                                                                     │
│                                                                             │
│      // 4. 添加 ServiceAccount 认证器                                       │
│      if config.ServiceAccountPublicKeysGetter != nil {                     │
│          serviceAccountAuth, _ := newServiceAccountAuthenticator(          │
│              config.ServiceAccountIssuers,                                 │
│              config.ServiceAccountPublicKeysGetter,                        │
│              config.APIAudiences,                                          │
│              config.ServiceAccountTokenGetter,                             │
│          )                                                                 │
│          tokenAuthenticators = append(tokenAuthenticators,                 │
│              serviceAccountAuth,                                           │
│          )                                                                 │
│      }                                                                     │
│                                                                             │
│      // 5. 添加 Bootstrap Token 认证器                                      │
│      if config.BootstrapTokenAuthenticator != nil {                        │
│          tokenAuthenticators = append(tokenAuthenticators,                 │
│              authenticator.WrapAudienceAgnosticToken(                      │
│                  config.APIAudiences,                                      │
│                  config.BootstrapTokenAuthenticator,                       │
│              ))                                                            │
│      }                                                                     │
│                                                                             │
│      // 6. 添加 OIDC/JWT 认证器                                             │
│      if len(config.AuthenticationConfig.JWT) > 0 {                         │
│          jwtAuthenticator, _ := newJWTAuthenticator(                       │
│              config.AuthenticationConfig.JWT,                              │
│              config.APIAudiences,                                          │
│              config.OIDCSigningAlgs,                                       │
│          )                                                                 │
│          tokenAuthenticators = append(tokenAuthenticators,                 │
│              jwtAuthenticator,                                             │
│          )                                                                 │
│      }                                                                     │
│                                                                             │
│      // 7. 添加 Webhook 认证器                                              │
│      if len(config.WebhookTokenAuthnConfigFile) > 0 {                      │
│          webhookAuth, _ := newWebhookTokenAuthenticator(config)            │
│          tokenAuthenticators = append(tokenAuthenticators, webhookAuth)    │
│      }                                                                     │
│                                                                             │
│      // 8. 将所有 token 认证器组合并添加缓存                                 │
│      if len(tokenAuthenticators) > 0 {                                     │
│          tokenAuth := tokenunion.New(tokenAuthenticators...)               │
│          tokenAuth = tokencache.New(                                       │
│              tokenAuth,                                                    │
│              true,                                                         │
│              config.TokenSuccessCacheTTL,                                  │
│              config.TokenFailureCacheTTL,                                  │
│          )                                                                 │
│          authenticators = append(authenticators,                           │
│              bearertoken.New(tokenAuth),                                   │
│              websocket.NewProtocolAuthenticator(tokenAuth),                │
│          )                                                                 │
│      }                                                                     │
│                                                                             │
│      // 9. 添加匿名认证器（最后）                                            │
│      if config.Anonymous.Enabled {                                         │
│          authenticators = append(authenticators,                           │
│              anonymous.NewAuthenticator(),                                 │
│          )                                                                 │
│      }                                                                     │
│                                                                             │
│      // 10. 组合所有认证器                                                  │
│      authenticator := union.New(authenticators...)                         │
│      authenticator = group.NewAuthenticatedGroupAdder(authenticator)       │
│                                                                             │
│      return authenticator, updateFunc, &securityDefs, securitySchemes, nil │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                    返回认证器链（Authenticator Chain）                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  authenticator.Request (union 组合)                                         │
│    │                                                                       │
│    ├─ RequestHeaderAuthenticator      // 请求头认证                         │
│    │                                                                       │
│    ├─ X509Authenticator                // 客户端证书认证                    │
│    │                                                                       │
│    ├─ BearerTokenAuthenticator         // Bearer Token 认证                │
│    │   └─ TokenCache                   // Token 缓存层                     │
│    │       └─ TokenUnion                // Token 认证器联合                 │
│    │           ├─ TokenFileAuthenticator         // Token 文件认证         │
│    │           ├─ ServiceAccountAuthenticator    // ServiceAccount 认证    │
│    │           ├─ BootstrapTokenAuthenticator    // Bootstrap Token 认证   │
│    │           ├─ JWTAuthenticator (OIDC)        // OIDC/JWT 认证         │
│    │           └─ WebhookAuthenticator           // Webhook 认证           │
│    │                                                                       │
│    └─ AnonymousAuthenticator          // 匿名认证（最后一个）                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│              第七步：赋值给服务器配置（回到 ApplyTo）                           │
│              pkg/kubeapiserver/options/authentication.go:731                │
├─────────────────────────────────────────────────────────────────────────────┤
│  authInfo.Authenticator = authenticator  // 🎯 最终赋值                     │
│                                                                             │
│  // authInfo 是 genericConfig.Authentication 的引用                         │
│  // 所以实际上是：                                                           │
│  // genericConfig.Authentication.Authenticator = authenticator             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                  最终结果：genericapiserver.Config                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  type Config struct {                                                      │
│      ...                                                                   │
│      Authentication AuthenticationInfo {                                   │
│          Authenticator: authenticator.Request,  // 🎯 认证器链             │
│          APIAudiences:  []string,                                          │
│          RequestHeaderConfig: *RequestHeaderConfig,                        │
│      }                                                                     │
│      Authorization AuthorizationInfo {                                     │
│          Authorizer: authorizer.Authorizer,                                │
│      }                                                                     │
│      ...                                                                   │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                    服务器启动，使用认证器处理请求                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  每个 HTTP 请求到达时：                                                      │
│    1. 经过认证中间件                                                         │
│    2. 调用 genericConfig.Authentication.Authenticator.AuthenticateRequest()│
│    3. 认证器链按顺序尝试认证                                                  │
│    4. 第一个成功的认证器返回用户信息                                          │
│    5. 如果所有认证器都失败，匿名认证器返回匿名用户（如果启用）                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 从运行配置到 Generic Server 过滤器

```markdown
┌─────────────────────────────────────────────────────────────────────────────┐
│                  第一步：Run() → CreateServerChain()                         │
│                  cmd/kube-apiserver/app/server.go                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  func Run(ctx context.Context, opts options.CompletedOptions) error {      │
│      config, err := NewConfig(opts)                                        │
│      completed, err := config.Complete()                                   │
│      server, err := CreateServerChain(completed)  // 👈 创建服务器链         │
│      prepared, err := server.PrepareRun()                                  │
│      return prepared.Run(ctx)                                              │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│              第二步：CreateServerChain() 创建三层服务器                         │
│              cmd/kube-apiserver/app/server.go:176                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  func CreateServerChain(config CompletedConfig) (*APIAggregator, error) {  │
│      // 1. APIExtensions Server (CRD)                                      │
│      apiExtensionsServer, _ := config.ApiExtensions.New(...)               │
│                                                                             │
│      // 2. KubeAPIServer (核心 API) 👈 关键                                 │
│      kubeAPIServer, _ := config.KubeAPIs.New(apiExtensionsServer)          │
│                                                                             │
│      // 3. Aggregator Server (聚合层)                                       │
│      aggregatorServer, _ := CreateAggregatorServer(...)                    │
│                                                                             │
│      return aggregatorServer, nil                                          │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│           第三步：多层 New() 调用链（创建 GenericAPIServer）                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  config.KubeAPIs.New(delegationTarget)                                     │
│    ↓ pkg/controlplane/instance.go:316                                      │
│  CompletedConfig.New(delegationTarget)                                     │
│    ↓ 调用                                                                   │
│  c.ControlPlane.New("KubeAPIServer", delegationTarget)                     │
│    ↓ pkg/controlplane/apiserver/server.go:95                               │
│  completedConfig.New("KubeAPIServer", delegationTarget)                    │
│    ↓ 调用                                                                   │
│  c.Generic.New("KubeAPIServer", delegationTarget)                          │
│    ↓ staging/src/k8s.io/apiserver/pkg/server/config.go:766                │
│  completedConfig.New("KubeAPIServer", delegationTarget)  // 👈 最终到达     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│      第四步：BuildHandlerChainFunc 的赋值下一步需要使用
│      staging/src/k8s.io/apiserver/pkg/server/config.go:396                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  func NewConfig(codecs serializer.CodecFactory) *Config {                  │
│      return &Config{                                                       │
│          Serializer: codecs,                                               │
│          BuildHandlerChainFunc: DefaultBuildHandlerChain,  // 🔥 默认赋值   │
│          // ... 其他字段                                                    │
│      }                                                                     │
│  }                                                                         │
│                                                                             │
│  // 说明：                                                                  │
│  // 1. BuildHandlerChainFunc 是一个函数类型字段                              │
│  // 2. 默认值是 DefaultBuildHandlerChain 函数                               │
│  // 3. 可以被覆盖（如 aggregator 会包装它）                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│         第五步：GenericAPIServer.New() 使用 BuildHandlerChainFunc             │
│         staging/src/k8s.io/apiserver/pkg/server/config.go:766               │
├─────────────────────────────────────────────────────────────────────────────┤
│  func (c completedConfig) New(name string, delegationTarget DelegationTarget)│
│      (*GenericAPIServer, error) {                                          │
│                                                                             │
│      // 🔥 创建 handler chain builder 闭包                                  │
│      handlerChainBuilder := func(handler http.Handler) http.Handler {      │
│          return c.BuildHandlerChainFunc(handler, c.Config)                 │
│          //     👆 调用 DefaultBuildHandlerChain(handler, c.Config)        │
│      }                                                                     │
│                                                                             │
│      // 创建 APIServerHandler，传入 builder                                 │
│      apiServerHandler := NewAPIServerHandler(                              │
│          name,                                                             │
│          c.Serializer,                                                     │
│          handlerChainBuilder,  // 👈 传递闭包                               │
│          delegationTarget.UnprotectedHandler(),                            │
│      )                                                                     │
│                                                                             │
│      s := &GenericAPIServer{                                               │
│          Handler: apiServerHandler,  // 包含完整的过滤器链                   │
│          // ... 其他字段                                                    │
│      }                                                                     │
│                                                                             │
│      return s, nil                                                         │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│          第六步：NewAPIServerHandler() 调用 builder                           │
│          staging/src/k8s.io/apiserver/pkg/server/handler.go                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  func NewAPIServerHandler(                                                 │
│      name string,                                                          │
│      s runtime.NegotiatedSerializer,                                       │
│      handlerChainBuilder HandlerChainBuilderFn,                            │
│      notFoundHandler http.Handler,                                         │
│  ) *APIServerHandler {                                                     │
│                                                                             │
│      // 创建 director (路由分发器)                                          │
│      director := director{                                                 │
│          name:               name,                                         │
│          goRestfulContainer: gorestfulContainer,                           │
│          nonGoRestfulMux:    nonGoRestfulMux,                              │
│      }                                                                     │
│                                                                             │
│      return &APIServerHandler{                                             │
│          // 🔥 调用 builder 构建完整的过滤器链                               │
│          FullHandlerChain: handlerChainBuilder(director),                  │
│          //                👆                                              │
│          //                调用 c.BuildHandlerChainFunc(director, c.Config)│
│          //                即 DefaultBuildHandlerChain(director, c.Config) │
│          //                                                                │
│          GoRestfulContainer: gorestfulContainer,                           │
│          NonGoRestfulMux:    nonGoRestfulMux,                              │
│          Director:           director,                                     │
│      }                                                                     │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│       第七步：DefaultBuildHandlerChain() 构建认证过滤器                         │
│       staging/src/k8s.io/apiserver/pkg/server/config.go:1014               │
├─────────────────────────────────────────────────────────────────────────────┤
│  func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config)         │
│      http.Handler {                                                        │
│                                                                             │
│      handler := apiHandler  // director (路由分发器)                        │
│                                                                             │
│      // 从内到外构建过滤器链（洋葱模型）                                       │
│                                                                             │
│      // ... 授权过滤器                                                       │
│      handler = genericapifilters.WithAuthorization(                        │
│          handler, c.Authorization.Authorizer, c.Serializer)                │
│                                                                             │
│      // ... 流控过滤器                                                       │
│      handler = genericfilters.WithPriorityAndFairness(...)                 │
│                                                                             │
│      // ... 伪装过滤器                                                       │
│      handler = genericapifilters.WithImpersonation(...)                    │
│                                                                             │
│      // ... 审计过滤器                                                       │
│      handler = genericapifilters.WithAudit(...)                            │
│                                                                             │
│      // 认证失败处理器                                                       │
│      failedHandler := genericapifilters.Unauthorized(c.Serializer)         │
│      failedHandler = genericapifilters.WithFailedAuthenticationAudit(...)  │
│                                                                             │
│      // 🔥🔥🔥 认证过滤器（关键！）                                           │
│      handler = genericapifilters.WithAuthentication(                       │
│          handler,                                                          │
│          c.Authentication.Authenticator,  // 👈 从 Config 取出认证器        │
│          failedHandler,                                                    │
│          c.Authentication.APIAudiences,                                    │
│          c.Authentication.RequestHeaderConfig,                             │
│      )                                                                     │
│                                                                             │
│      // ... 其他外层过滤器                                                   │
│      handler = genericfilters.WithCORS(handler, ...)                       │
│      handler = genericfilters.WithTimeoutForNonLongRunningRequests(...)    │
│      handler = genericapifilters.WithRequestInfo(...)                      │
│      handler = genericfilters.WithPanicRecovery(...)                       │
│                                                                             │
│      return handler  // 返回完整的过滤器链                                   │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│          第八步：WithAuthentication() 创建认证过滤器                            │
│          staging/src/k8s.io/apiserver/pkg/endpoints/filters/               │
│          authentication.go:46                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  func WithAuthentication(                                                  │
│      handler http.Handler,                                                 │
│      auth authenticator.Request,  // 👈 认证器（从 Config 传入）            │
│      failed http.Handler,                                                  │
│      apiAuds authenticator.Audiences,                                      │
│      requestHeaderConfig *RequestHeaderConfig,                             │
│  ) http.Handler {                                                          │
│                                                                             │
│      return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {│
│          // 设置 API Audiences                                              │
│          if len(apiAuds) > 0 {                                             │
│              req = req.WithContext(                                        │
│                  authenticator.WithAudiences(req.Context(), apiAuds))      │
│          }                                                                 │
│                                                                             │
│          // 🔥 调用认证器进行认证                                            │
│          resp, ok, err := auth.AuthenticateRequest(req)                    │
│          //                👆                                              │
│          //                认证器链按顺序尝试认证：                          │
│          //                1. RequestHeaderAuthenticator                   │
│          //                2. X509Authenticator                            │
│          //                3. BearerTokenAuthenticator                     │
│          //                   └─ TokenCache                                │
│          //                       └─ TokenUnion                            │
│          //                           ├─ TokenFileAuthenticator            │
│          //                           ├─ ServiceAccountAuthenticator       │
│          //                           ├─ BootstrapTokenAuthenticator       │
│          //                           ├─ JWTAuthenticator (OIDC)           │
│          //                           └─ WebhookAuthenticator              │
│          //                4. AnonymousAuthenticator                       │
│                                                                             │
│          // 认证失败处理                                                     │
│          if err != nil || !ok {                                            │
│              failed.ServeHTTP(w, req)  // 返回 401                         │
│              return                                                        │
│          }                                                                 │
│                                                                             │
│          // 验证 Audiences                                                  │
│          if !audiencesAreAcceptable(apiAuds, resp.Audiences) {             │
│              failed.ServeHTTP(w, req)                                      │
│              return                                                        │
│          }                                                                 │
│                                                                             │
│          // 🎯 认证成功：删除 Authorization 头                              │
│          req.Header.Del("Authorization")                                   │
│                                                                             │
│          // 删除前端代理头                                                   │
│          headerrequest.ClearAuthenticationHeaders(...)                     │
│                                                                             │
│          // 🎯 将用户信息存入请求上下文                                       │
│          req = req.WithContext(                                            │
│              genericapirequest.WithUser(req.Context(), resp.User))         │
│                                                                             │
│          // 🎯 调用下一个处理器（授权过滤器）                                 │
│          handler.ServeHTTP(w, req)                                         │
│      })                                                                    │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                    第九步：HTTP 请求处理流程                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  客户端发送 HTTP 请求                                                        │
│    ↓                                                                       │
│  到达 GenericAPIServer.Handler.FullHandlerChain                            │
│    ↓                                                                       │
│  经过过滤器链（从外到内）：                                                   │
│    1. WithPanicRecovery          // Panic 恢复                             │
│    2. WithRequestInfo            // 解析请求信息                           │
│    3. WithTimeoutForNonLongRunningRequests // 超时                         │
│    4. WithCORS                   // CORS                                   │
│    5. 🔥 WithAuthentication      // 认证（关键！）                          │
│       │                                                                    │
│       ├─ 调用 auth.AuthenticateRequest(req)                                │
│       │   └─ 认证器链按顺序尝试认证                                          │
│       │                                                                    │
│       ├─ 认证成功：                                                         │
│       │   - 删除 Authorization 头                                          │
│       │   - 将用户信息存入 context                                          │
│       │   - 调用下一个处理器                                                │
│       │                                                                    │
│       └─ 认证失败：                                                         │
│           - 返回 401 Unauthorized                                          │
│                                                                             │
│    6. WithAudit                  // 审计                                   │
│    7. WithImpersonation          // 伪装                                   │
│    8. WithPriorityAndFairness    // 流控                                   │
│    9. WithAuthorization          // 授权                                   │
│    ↓                                                                       │
│  到达业务逻辑处理器 (director)                                               │
│    ↓                                                                       │
│  路由到具体的 API Handler                                                   │
│    ↓                                                                       │
│  返回响应                                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

