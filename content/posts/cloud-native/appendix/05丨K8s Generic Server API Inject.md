+++
title = "K8s Generic Server API Inject"
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

series_order = 8

+++

# K8s Generic Server API Inject

## 1. Sechme 类型注册中心

`runtime.Scheme` 里注册了所有资源的 Go 类型及版本转换函数。每个 API 组在自己的 `install` 包里调用 `Install(legacyschme.Scheme)`，把该组所有版本的类型加入 `Scheme` 中，并设置版本优先级。

以 `pkg/apis/apps/install/install.go` 为例：

```go
// 代码: pkg\apis\apps\install\install.go#L36-L42
// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
	utilruntime.Must(apps.AddToScheme(scheme))
	utilruntime.Must(v1beta1.AddToScheme(scheme))
	utilruntime.Must(v1beta2.AddToScheme(scheme))
	utilruntime.Must(v1.AddToScheme(scheme))
	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion, v1beta2.SchemeGroupVersion, v1beta1.SchemeGroupVersion))
}
```

## 2. `NewDefaultAPIGroupInfo` 读取 Scheme 信息

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L1004-L1027
// NewDefaultAPIGroupInfo returns an APIGroupInfo stubbed with "normal" values
// exposed for easier composition from other packages
func NewDefaultAPIGroupInfo(group string, scheme *runtime.Scheme, parameterCodec runtime.ParameterCodec, codecs serializer.CodecFactory) APIGroupInfo {
	opts := []serializer.CodecFactoryOptionsMutator{}
	if utilfeature.DefaultFeatureGate.Enabled(features.CBORServingAndStorage) {
		opts = append(opts, serializer.WithSerializer(cbor.NewSerializerInfo))
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.StreamingCollectionEncodingToJSON) {
		opts = append(opts, serializer.WithStreamingCollectionEncodingToJSON())
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.StreamingCollectionEncodingToProtobuf) {
		opts = append(opts, serializer.WithStreamingCollectionEncodingToProtobuf())
	}
	if len(opts) != 0 {
		codecs = serializer.NewCodecFactory(scheme, opts...)
	}
	return APIGroupInfo{
		PrioritizedVersions:          scheme.PrioritizedVersionsForGroup(group),
		VersionedResourcesStorageMap: map[string]map[string]rest.Storage{},
		// TODO unhardcode this.  It was hardcoded before, but we need to re-evaluate
		OptionsExternalVersion: &schema.GroupVersion{Version: "v1"},
		Scheme:                 scheme,
		ParameterCodec:         parameterCodec,
		NegotiatedSerializer:   codecs,
	}
}
```

* 调 `scheme.PrioritizedVersionsForGroup(group)` 拿到这个组的全部版本，赋给 `APIGroupInfo.PrioritizedVersions`；
* 把 `scheme`、`parameterCodec`、`codecFactory` 等注入 `APIGroupInfo`，以便后续序列化/解码、默认值、转换等都能使用同一个 Scheme
* 这里会初始化 `VersionedResourcesStorageMap: map[string]map[string]rest.Storage{},`。

## 3. RESTStorageProvider

`RESTStorageProvider` 接口的作用是让每个 API Group（如 `apps`, `batch`, `networking`）独立负责创建自己的后端存储逻辑，并将结果打包成 `APIGroupInfo` 交给主服务器。代码如下：

```go
// 代码: pkg\controlplane\apiserver\apis.go#L42-#L45
// RESTStorageProvider is a factory type for REST storage.
type RESTStorageProvider interface {
	GroupName() string
	NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, error)
}
```

## 4. StorageProvider 利用 Scheme 构造 VersionedResourcesStorageMap

以 apps 组【`pkg\registry\apps\rest\storage_apps.go`】为例：

```go
// 代码: pkg\registry\apps\rest\storage_apps.go#L35-#L113
// StorageProvider is a struct for apps REST storage.
type StorageProvider struct{}

// NewRESTStorage returns APIGroupInfo object.
func (p StorageProvider) NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, error) {
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apps.GroupName, legacyscheme.Scheme, legacyscheme.ParameterCodec, legacyscheme.Codecs) //要点①
	// If you add a version here, be sure to add an entry in `k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go with specific priorities.
	// TODO refactor the plumbing to provide the information in the APIGroupInfo

	if storageMap, err := p.v1Storage(apiResourceConfigSource, restOptionsGetter); err != nil { //要点②
		return genericapiserver.APIGroupInfo{}, err
	} else if len(storageMap) > 0 {
		apiGroupInfo.VersionedResourcesStorageMap[appsapiv1.SchemeGroupVersion.Version] = storageMap //要点③
	}

	return apiGroupInfo, nil
}

func (p StorageProvider) v1Storage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (map[string]rest.Storage, error) {
	storage := map[string]rest.Storage{}

	// deployments
	if resource := "deployments"; apiResourceConfigSource.ResourceEnabled(appsapiv1.SchemeGroupVersion.WithResource(resource)) {
		deploymentStorage, err := deploymentstore.NewStorage(restOptionsGetter)
		if err != nil {
			return storage, err
		}
		storage[resource] = deploymentStorage.Deployment
		storage[resource+"/status"] = deploymentStorage.Status
		storage[resource+"/scale"] = deploymentStorage.Scale
	}
	...
	return storage, nil
}

// GroupName returns name of the group
func (p StorageProvider) GroupName() string {
	return apps.GroupName
}
```

1. `NewRESTStorage` 首先调用 `genericapiserver.NewDefaultAPIGroupInfo` 得到空 map 的 `APIGroupInfo`；【要点①】
2. 接着会调用 `v1Storage()` 函数，构造 `map[string]rest.Storage`；【要点②】
3. 待 `storage` 填好后，再写回到 `apiGroupInfo.VersionedResourcesStorageMap[appsapiv1.SchemeGroupVersion.Version] = storage`。【要点③】

其它 API 组（core、rbac、certificates…）拥有各自的 StorageProvider，同样流程。

## 5. GenericAPIServer 消费 Scheme/Storage 信息

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L780-#L840
// installAPIResources is a private method for installing the REST storage backing each api groupversionresource
func (s *GenericAPIServer) installAPIResources(apiPrefix string, apiGroupInfo *APIGroupInfo, typeConverter managedfields.TypeConverter) error {
	var resourceInfos []*storageversion.ResourceInfo
	for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
		if len(apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version]) == 0 {
			klog.Warningf("Skipping API %v because it has no resources.", groupVersion)
			continue
		}

		apiGroupVersion, err := s.getAPIGroupVersion(apiGroupInfo, groupVersion, apiPrefix) //要点①
		if err != nil {
			return err
		}
		if apiGroupInfo.OptionsExternalVersion != nil {
			apiGroupVersion.OptionsExternalVersion = apiGroupInfo.OptionsExternalVersion
		}
		apiGroupVersion.TypeConverter = typeConverter
		apiGroupVersion.MaxRequestBodyBytes = s.maxRequestBodyBytes

		discoveryAPIResources, r, err := apiGroupVersion.InstallREST(s.Handler.GoRestfulContainer) //要点②

		if err != nil {
			return fmt.Errorf("unable to setup API %v: %v", apiGroupInfo, err)
		}
		resourceInfos = append(resourceInfos, r...)

		// Aggregated discovery only aggregates resources under /apis
		if apiPrefix == APIGroupPrefix {
			s.AggregatedDiscoveryGroupManager.AddGroupVersion(
				groupVersion.Group,
				apidiscoveryv2.APIVersionDiscovery{
					Freshness: apidiscoveryv2.DiscoveryFreshnessCurrent,
					Version:   groupVersion.Version,
					Resources: discoveryAPIResources,
				},
			)
		} else {
			// There is only one group version for legacy resources, priority can be defaulted to 0.
			s.AggregatedLegacyDiscoveryGroupManager.AddGroupVersion(
				groupVersion.Group,
				apidiscoveryv2.APIVersionDiscovery{
					Freshness: apidiscoveryv2.DiscoveryFreshnessCurrent,
					Version:   groupVersion.Version,
					Resources: discoveryAPIResources,
				},
			)
		}

	}

	s.RegisterDestroyFunc(apiGroupInfo.destroyStorage) //要点③

	if s.FeatureGate.Enabled(features.StorageVersionAPI) &&
		s.FeatureGate.Enabled(features.APIServerIdentity) {
		// API installation happens before we start listening on the handlers,
		// therefore it is safe to register ResourceInfos here. The handler will block
		// write requests until the storage versions of the targeting resources are updated.
		s.StorageVersionManager.AddResourceInfo(resourceInfos...)
	}

	return nil
}
```

* ```go
  for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
      if len(apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version]) == 0 {
          continue
      }
      ...
  }
  ```

  这里使用 `VersionedResourcesStorageMap` 判断某个 version 是否有实际 Storage，如果没有就跳过。

* `getAPIGroupVersion`：把 Scheme/Storage 打包成 APIGroupVersion：【要点①】

  ```go
  // 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L952-L964
  func (s *GenericAPIServer) getAPIGroupVersion(apiGroupInfo *APIGroupInfo, groupVersion schema.GroupVersion, apiPrefix string) (*genericapi.APIGroupVersion, error) {
  	storage := make(map[string]rest.Storage)
  	for k, v := range apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version] {
  		if strings.ToLower(k) != k {
  			return nil, fmt.Errorf("resource names must be lowercase only, not %q", k)
  		}
  		storage[k] = v
  	}
  	version := s.newAPIGroupVersion(apiGroupInfo, groupVersion)
  	version.Root = apiPrefix
  	version.Storage = storage
  	return version, nil
  }
  ```

  * 从 `VersionedResourcesStorageMap[groupVersion.Version]` 拿到 `map[string]rest.Storage`

  * `newAPIGroupVersion` 里把 `apiGroupInfo.Scheme` 提供的能力注入：`Creater`、`Defaulter`、`Typer`、`Serializer`、`ParameterCodec` 等，这样 `APIGroupVersion` 就知道如何根据 Scheme 创建对象、默认化、版本转换、编码/解码。

    ```go
    func (s *GenericAPIServer) newAPIGroupVersion(apiGroupInfo *APIGroupInfo, groupVersion schema.GroupVersion) *genericapi.APIGroupVersion {
    
    	allServedVersionsByResource := map[string][]string{}
    	for version, resourcesInVersion := range apiGroupInfo.VersionedResourcesStorageMap {
    		for resource := range resourcesInVersion {
    			if len(groupVersion.Group) == 0 {
    				allServedVersionsByResource[resource] = append(allServedVersionsByResource[resource], version)
    			} else {
    				allServedVersionsByResource[resource] = append(allServedVersionsByResource[resource], fmt.Sprintf("%s/%s", groupVersion.Group, version))
    			}
    		}
    	}
    
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
    
    		Admit:             s.admissionControl,
    		MinRequestTimeout: s.minRequestTimeout,
    		Authorizer:        s.Authorizer,
    	}
    }
    ```

  * 最终，`version.Storage = storage` 即为这一版本的 Storage。

* InstallREST：根据 Storage + Scheme 生成路由

  ```go
  discoveryAPIResources, r, err := apiGroupVersion.InstallREST(s.Handler.GoRestfulContainer)
  ```

  ```go
  // 代码: staging\src\k8s.io\apiserver\pkg\endpoints\groupversion.go#L106-L123
  // InstallREST registers the REST handlers (storage, watch, proxy and redirect) into a restful Container.
  // It is expected that the provided path root prefix will serve all operations. Root MUST NOT end
  // in a slash.
  func (g *APIGroupVersion) InstallREST(container *restful.Container) ([]apidiscoveryv2.APIResourceDiscovery, []*storageversion.ResourceInfo, error) {
  	prefix := path.Join(g.Root, g.GroupVersion.Group, g.GroupVersion.Version)
  	installer := &APIInstaller{
  		group:             g,
  		prefix:            prefix,
  		minRequestTimeout: g.MinRequestTimeout,
  	}
  
  	apiResources, resourceInfos, ws, registrationErrors := installer.Install()
  	versionDiscoveryHandler := discovery.NewAPIVersionHandler(g.Serializer, g.GroupVersion, staticLister{apiResources})
  	versionDiscoveryHandler.AddToWebService(ws)
  	container.Add(ws)
  	aggregatedDiscoveryResources, err := ConvertGroupVersionIntoToDiscovery(apiResources)
  	if err != nil {
  		registrationErrors = append(registrationErrors, err)
  	}
  	return aggregatedDiscoveryResources, removeNonPersistedResources(resourceInfos), utilerrors.NewAggregate(registrationErrors)
  }
  ```

  * `Install` 内部会新建 `APIInstaller`，然后调用 `installer.Install()`，完成实际路由注册

    ```go
    apiResources, resourceInfos, ws, registrationErrors := installer.Install()
    ```

    ```go
    // 代码: staging\src\k8s.io\apiserver\pkg\endpoints\installer.go#L193-L220
    // Install handlers for API resources.
    func (a *APIInstaller) Install() ([]metav1.APIResource, []*storageversion.ResourceInfo, *restful.WebService, []error) {
    	var apiResources []metav1.APIResource
    	var resourceInfos []*storageversion.ResourceInfo
    	var errors []error
    	ws := a.newWebService()
    
    	// Register the paths in a deterministic (sorted) order to get a deterministic swagger spec.
    	paths := make([]string, len(a.group.Storage))
    	var i int = 0
    	for path := range a.group.Storage {
    		paths[i] = path
    		i++
    	}
    	sort.Strings(paths)
    	for _, path := range paths {
    		apiResource, resourceInfo, err := a.registerResourceHandlers(path, a.group.Storage[path], ws)
    		if err != nil {
    			errors = append(errors, fmt.Errorf("error in registering resource: %s, %v", path, err))
    		}
    		if apiResource != nil {
    			apiResources = append(apiResources, *apiResource)
    		}
    		if resourceInfo != nil {
    			resourceInfos = append(resourceInfos, resourceInfo)
    		}
    	}
    	return apiResources, resourceInfos, ws, errors
    }
    ```

    `Install` 会遍历 `g.Storage`，中的每个 `resource/subresource`，调用 `registerResourceHandlers` 生成 `restful.Route` 并添加到新的 `WebService`。

    返回值：

    * `apiResources`：`[]metav1.APIResource`，描述当前版本暴露的所有资源，用于 discovery。
    * `resourceInfos`：存储版本信息（StorageVersion API 用）。
    * `ws`：刚构建好的 `restful.WebService`，里边已经包含所有 route。
    * `registrationErrors`：过程中可能累积的错误列表。

  * 为该版本添加 discovery handler，并把 WebService 挂到容器

    ```go
    versionDiscoveryHandler := discovery.NewAPIVersionHandler(g.Serializer, g.GroupVersion, staticLister{apiResources})
    versionDiscoveryHandler.AddToWebService(ws)
    container.Add(ws)
    ```

    * 创建 `/apis/<group>/<version>` 的 discovery endpoint（返回支持的资源列表），复用与资源同一个 WebService。
    * 将 `ws` 注册进传入的 `restful.Container`，使这些路由对外生效。

  * 构建聚合 discovery 数据 & 过滤 StorageVersion 信息

    ```go
    	aggregatedDiscoveryResources, err := ConvertGroupVersionIntoToDiscovery(apiResources)
    	if err != nil {
    		registrationErrors = append(registrationErrors, err)
    	}
    	return aggregatedDiscoveryResources, removeNonPersistedResources(resourceInfos), utilerrors.NewAggregate(registrationErrors)
    ```

* 辅助信息更新

  * `InstallREST` 返回 `discoveryAPIResources`（供 discovery 展示）和 `resourceInfos`（用于 StorageVersionManager）。函数把它们收集起来，稍后更新 AggregatedDiscovery、注册 storage readiness、记录 StorageVersion 等 
  * 同时调用 `s.RegisterDestroyFunc(apiGroupInfo.destroyStorage)` ，确保停止时销毁 Storage。

## 6. 流程总结

API 分组安装与路由注册的流程如下：

### controlplane.Server 构建 APIGroupInfo

`controlplane.Server` 构建 APIGroupInfo 的逻辑在 `pkg\controlplane\instance.go` 中：

```go
// 代码: pkg\controlplane\instance.go#L316-L384
// New returns a new instance of Master from the given config.
// Certain config fields will be set to a default value if unset.
// Certain config fields must be specified, including:
// KubeletClientConfig
func (c CompletedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
	...
	restStorageProviders, err := c.StorageProviders(client) //要点①
	if err != nil {
		return nil, err
	}

	if err := s.ControlPlane.InstallAPIs(restStorageProviders...); err != nil { //要点②
		return nil, err
	}
	...
	return s, nil
}
```

* 首先通过 `restStorageProviders, err := c.StorageProviders(client)` 构建出 `[]RESTStorageProvider`。

* 然后将构建好的传入 `s.ControlPlane.InstallAPIs` 

  ```go
  // 代码: pkg\controlplane\apiserver\apis.go#L88-L153
  // InstallAPIs will install the APIs for the restStorageProviders if they are enabled.
  func (s *Server) InstallAPIs(restStorageProviders ...RESTStorageProvider) error {
  	nonLegacy := []*genericapiserver.APIGroupInfo{}
  
  	// used later in the loop to filter the served resource by those that have expired.
  	resourceExpirationEvaluatorOpts := genericapiserver.ResourceExpirationEvaluatorOptions{
  		CurrentVersion:                          s.GenericAPIServer.EffectiveVersion.EmulationVersion(),
  		Prerelease:                              s.GenericAPIServer.EffectiveVersion.BinaryVersion().PreRelease(),
  		EmulationForwardCompatible:              s.GenericAPIServer.EmulationForwardCompatible,
  		RuntimeConfigEmulationForwardCompatible: s.GenericAPIServer.RuntimeConfigEmulationForwardCompatible,
  	}
  	resourceExpirationEvaluator, err := genericapiserver.NewResourceExpirationEvaluatorFromOptions(resourceExpirationEvaluatorOpts)
  	if err != nil {
  		return err
  	}
  
  	for _, restStorageBuilder := range restStorageProviders { //要点①
  		groupName := restStorageBuilder.GroupName()
  		apiGroupInfo, err := restStorageBuilder.NewRESTStorage(s.APIResourceConfigSource, s.RESTOptionsGetter) //要点②
  		if err != nil {
  			return fmt.Errorf("problem initializing API group %q: %w", groupName, err)
  		}
  		if len(apiGroupInfo.VersionedResourcesStorageMap) == 0 {
  			// If we have no storage for any resource configured, this API group is effectively disabled.
  			// This can happen when an entire API group, version, or development-stage (alpha, beta, GA) is disabled.
  			klog.Infof("API group %q is not enabled, skipping.", groupName)
  			continue
  		}
  
  		// Remove resources that serving kinds that are removed or not introduced yet at the current version.
  		// We do this here so that we don't accidentally serve versions without resources or openapi information that for kinds we don't serve.
  		// This is a spot above the construction of individual storage handlers so that no sig accidentally forgets to check.
  		err = resourceExpirationEvaluator.RemoveUnavailableKinds(groupName, apiGroupInfo.Scheme, apiGroupInfo.VersionedResourcesStorageMap, s.APIResourceConfigSource)
  		if err != nil {
  			return err
  		}
  		if len(apiGroupInfo.VersionedResourcesStorageMap) == 0 {
  			klog.V(1).Infof("Removing API group %v because it is time to stop serving it because it has no versions per APILifecycle.", groupName)
  			continue
  		}
  
  		klog.V(1).Infof("Enabling API group %q.", groupName)
  
  		if postHookProvider, ok := restStorageBuilder.(genericapiserver.PostStartHookProvider); ok {
  			name, hook, err := postHookProvider.PostStartHook()
  			if err != nil {
  				return fmt.Errorf("error building PostStartHook: %w", err)
  			}
  			s.GenericAPIServer.AddPostStartHookOrDie(name, hook)
  		}
  
  		if len(groupName) == 0 {
  			// the legacy group for core APIs is special that it is installed into /api via this special install method.
  			if err := s.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); err != nil { //要点③
  				return fmt.Errorf("error in registering legacy API: %w", err)
  			}
  		} else {
  			// everything else goes to /apis
  			nonLegacy = append(nonLegacy, &apiGroupInfo)
  		}
  	}
  
  	if err := s.GenericAPIServer.InstallAPIGroups(nonLegacy...); err != nil { //要点④
  		return fmt.Errorf("error in registering group versions: %w", err)
  	}
  	return nil
  }
  ```

  * 要点①，`InstallAPIs` 会调用各个 `RESTStorageProvider`
  * 要点②，通过调用 `restStorageBuilder.NewRESTStorage` 来生成 `APIGroupInfo`，在 `APIGroupInfo` 中填好 `VersionedResourcesStorageMap`、`Scheme`、`Serializer `等。

### 交给 GenericAPIServer 安装

同一个函数里：

* 要点③，`s.GenericAPIServer.InstallLegacyAPIGroup`，专门处理 `groupName == ""` 的核心 legacy 组，把它们安装到 `/api` 前缀。

* 要点④，` s.GenericAPIServer.InstallAPIGroups(nonLegacy...)` 则一次性安装所有非 legacy 组（`groupName != ""`），挂在 `/apis` 前缀。

而对应的 `InstallxxxAPIGroup` 都会调用，`s.installAPIResources` 来进行安装，这样，就全部串联起来了。

### 总结

controlplane 负责准备（构建 APIGroupInfo），Generic Server 负责安装（把这些资源挂到 HTTP server）。
