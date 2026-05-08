+++
title = "K8s Generic Server Graceful Shutdown"
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

series_order = 7

+++

# K8s Generic Server Graceful Shutdown

> 假设服务器开启了所有可选开关，例如 `ShutdownSendRetryAfter` 开关、 `AuditBackend` 开关等等都打开。官方给出的时序流程图位于：`staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L480-L521`

![image-20251123030606406](http://images.liangning7.cn/typora/202511230306640.png)

这里 Generic Server 的停机流程包括：

1. `stopCh` 触发，`ShutdownInitiated` + 延迟窗口。
2. PreShutdownHooks 完成 + 延迟结束 ⇒ `NotAcceptingNewRequest` ⇒ 前门落下。
3. 非长连接 + watch 分别排空 ⇒ `InFlightRequestsDrained`。
4. 根据模式触发 `stopHttpServerCh` ⇒ listener 停止 ⇒ net/http Shutdown 完成。
5. Audit backend、HTTP server、Destroy 顺序收尾。

## `stopCh` 触发

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L522-L525
func (s preparedGenericAPIServer) RunWithContext(ctx context.Context) error {
	stopCh := ctx.Done()
	delayedStopCh := s.lifecycleSignals.AfterShutdownDelayDuration
	shutdownInitiatedCh := s.lifecycleSignals.ShutdownInitiated
    ...
}
```

* `RunWithContext` 把传入的 context 的 `Done()` 当作 `stopCh`
* 同时抓住两个声明周期信号：
  * `AfterShutdownDelayDuration`
  * `ShutdownInitiated`

 ```go
 	// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L555-L568
 	go func() {
 		defer delayedStopCh.Signal()
 		defer klog.V(1).InfoS("[graceful-termination] shutdown event", "name", delayedStopCh.Name())
 
 		<-stopCh
 
 		// As soon as shutdown is initiated, /readyz should start returning failure.
 		// This gives the load balancer a window defined by ShutdownDelayDuration to detect that /readyz is red
 		// and stop sending traffic to this server.
 		shutdownInitiatedCh.Signal()
 		klog.V(1).InfoS("[graceful-termination] shutdown event", "name", shutdownInitiatedCh.Name())
 
 		time.Sleep(s.ShutdownDelayDuration)
 	}()
 ```

当 `<-stopCh` 时：

* 立即 `shutdownInitiatedCh.Signal()`，这会让 `/readyz` 变红，通知外部 LB 节点即将退出。//TODO
* 随后 `time.Sleep(ShutdownDelayDuration)` 才 `delayedStopCh.Signal()`。这个“休眠窗口”就是给 LB 的摘除时间，避免停机瞬间仍有大量新请求涌入。

## `PreShutdownHooksStopped`  

```go
	// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L703-L716
	<-stopCh //要点①

	// run shutdown hooks directly. This includes deregistering from
	// the kubernetes endpoint in case of kube-apiserver.
	func() { //要点②
		defer func() {
			preShutdownHooksHasStoppedCh.Signal()
			klog.V(1).InfoS("[graceful-termination] pre-shutdown hooks completed", "name", preShutdownHooksHasStoppedCh.Name())
		}()
		err = s.RunPreShutdownHooks()
	}()
	if err != nil {
		return err
	}
```

主函数在要点①处被堵塞，当 `<-stopCh` 被触发之后，就会执行触发 `RunPreShutdownHooks`，等待其完成后 `preShutdownHooksHasStoppedCh.Signal()`

再来看看 `preShutdownHooksHasStoppedCh.Signal()` 会唤醒哪个部分：

```go
	// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L622-L634
	preShutdownHooksHasStoppedCh := s.lifecycleSignals.PreShutdownHooksStopped
	go func() {
		defer klog.V(1).InfoS("[graceful-termination] shutdown event", "name", notAcceptingNewRequestCh.Name())
		defer notAcceptingNewRequestCh.Signal()

		// wait for the delayed stopCh before closing the handler chain
		<-delayedStopCh.Signaled()

		// Additionally wait for preshutdown hooks to also be finished, as some of them need
		// to send API calls to clean up after themselves (e.g. lease reconcilers removing
		// itself from the active servers).
		<-preShutdownHooksHasStoppedCh.Signaled()
	}()
```

可以看到，这里在等待 `signCh` 触发后的 `delayedStopCh.Signal()` 信号，和 `RunPreShutdownHooks` 完成后 `preShutdownHooksHasStoppedCh.Signal()` 信号。

当 `delayedStopCh.Signal()` 和 `preShutdownHooksHasStoppedCh.Signal()` 信号到达后，就会发送 `notAcceptingNewRequestCh.Signal()` 信号，进而进入下流程。

和流程图对比查看：

![image-20251129172324079](http://images.liangning7.cn/typora/202511291723328.png)

现在已经完成了左边的处理，进入到了 `NotAcceptingNewRequest` 流程。

## 流量控制【NotAcceptingNewRequest】

这里的流量控制会根据 `s.ShutdownSendRetryAfter` 来进行配置，在 `stopHttpServerCh` 中，如果没有进行配置，那么在 `stopHttpServer` 中就会监听 `NotAcceptingNewRequest`，如果进行了配置，那么就会改为监听 `InFlightRequestsDrained`，代码如下：

```go
	// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L583-L579
	notAcceptingNewRequestCh := s.lifecycleSignals.NotAcceptingNewRequest
	drainedCh := s.lifecycleSignals.InFlightRequestsDrained
	// Canceling the parent context does not immediately cancel the HTTP server.
	// We only inherit context values here and deal with cancellation ourselves.
	stopHTTPServerCtx, stopHTTPServer := context.WithCancelCause(context.WithoutCancel(ctx))
	go func() {
		defer stopHTTPServer(errors.New("time to stop HTTP server"))

		timeToStopHttpServerCh := notAcceptingNewRequestCh.Signaled()
		if s.ShutdownSendRetryAfter {
			timeToStopHttpServerCh = drainedCh.Signaled()
		}

		<-timeToStopHttpServerCh
	}()
```

* 首先监听 `notAcceptingNewRequestCh`；

  ```go 
  timeToStopHttpServerCh := notAcceptingNewRequestCh.Signaled()
  ```

* 如果进行了配置：`if s.ShutdownSendRetryAfter {`，那么则改为监听`InFlightRequestsDrained`：

  ```go
  timeToStopHttpServerCh = drainedCh.Signaled()
  ```

### 非长连接排空

```go
	// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L636-L659
	// wait for all in-flight non-long running requests to finish
	nonLongRunningRequestDrainedCh := make(chan struct{})
	go func() {
		defer close(nonLongRunningRequestDrainedCh)
		defer klog.V(1).Info("[graceful-termination] in-flight non long-running request(s) have drained")

		// wait for the delayed stopCh before closing the handler chain (it rejects everything after Wait has been called).
		<-notAcceptingNewRequestCh.Signaled()

		// Wait for all requests to finish, which are bounded by the RequestTimeout variable.
		// once NonLongRunningRequestWaitGroup.Wait is invoked, the apiserver is
		// expected to reject any incoming request with a {503, Retry-After}
		// response via the WithWaitGroup filter. On the contrary, we observe
		// that incoming request(s) get a 'connection refused' error, this is
		// because, at this point, we have called 'Server.Shutdown' and
		// net/http server has stopped listening. This causes incoming
		// request to get a 'connection refused' error.
		// On the other hand, if 'ShutdownSendRetryAfter' is enabled incoming
		// requests will be rejected with a {429, Retry-After} since
		// 'Server.Shutdown' will be invoked only after in-flight requests
		// have been drained.
		// TODO: can we consolidate these two modes of graceful termination?
		s.NonLongRunningRequestWaitGroup.Wait()
	}()
```

当这里接收到了 `<-notAcceptingNewRequestCh.Signaled()`，则会 `s.NonLongRunningRequestWaitGroup.Wait()`

注意，这里的 `Wait()`：

```go
// staging\src\k8s.io\apimachinery\pkg\util\waitgroup\waitgroup.go
package waitgroup

import (
	"fmt"
	"sync"
)

// SafeWaitGroup must not be copied after first use.
type SafeWaitGroup struct {
	wg sync.WaitGroup
	mu sync.RWMutex
	// wait indicate whether Wait is called, if true,
	// then any Add with positive delta will return error.
	wait bool
}

// Add adds delta, which may be negative, similar to sync.WaitGroup.
// If Add with a positive delta happens after Wait, it will return error,
// which prevent unsafe Add.
func (wg *SafeWaitGroup) Add(delta int) error {
	wg.mu.RLock()
	defer wg.mu.RUnlock()
	if wg.wait && delta > 0 {
		return fmt.Errorf("add with positive delta after Wait is forbidden")
	}
	wg.wg.Add(delta)
	return nil
}

// Done decrements the WaitGroup counter.
func (wg *SafeWaitGroup) Done() {
	wg.wg.Done()
}

// Wait blocks until the WaitGroup counter is zero.
func (wg *SafeWaitGroup) Wait() {
	wg.mu.Lock()
	wg.wait = true
	wg.mu.Unlock()
	wg.wg.Wait()
}
```

这里的 `SafeWaitGroup` 的 `Wait` 会给 `wg.wait` 设置为 true，所以一旦 `wg.Wait` 之后，再每当新请求到来时，进行 `wg.Add(1)` 时，会引发 `error`。

这里讲解一下当新请求到来时，是如何走到 `wg.Add(1)`

新的请求已进入到 Generic Server，会按照 `DefaultBuildHandlerChain` 中定义的顺序通过一串 HTTP filter。

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\config.go#L1014
func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config) http.Handler {
	...
	handler = genericfilters.WithWaitGroup(handler, c.LongRunningFunc, c.NonLongRunningRequestWaitGroup) //#L1065
    ...
}
```

这里将 `WithWaitGroup` 被加入到 handler 链中，而 `WithWaitGroup` 的定义及实现如下：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\filters\waitgroup.go#L47-L88
// WithWaitGroup adds all non long-running requests to wait group, which is used for graceful shutdown.
func WithWaitGroup(handler http.Handler, longRunning apirequest.LongRunningRequestCheck, wg RequestWaitGroup) http.Handler {
	// NOTE: both WithWaitGroup and WithRetryAfter must use the same exact isRequestExemptFunc 'isRequestExemptFromRetryAfter,
	// otherwise SafeWaitGroup might wait indefinitely and will prevent the server from shutting down gracefully.
	return withWaitGroup(handler, longRunning, wg, isRequestExemptFromRetryAfter)
}

func withWaitGroup(handler http.Handler, longRunning apirequest.LongRunningRequestCheck, wg RequestWaitGroup, isRequestExemptFn isRequestExemptFunc) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		ctx := req.Context()
		requestInfo, ok := apirequest.RequestInfoFrom(ctx)
		if !ok {
			// if this happens, the handler chain isn't setup correctly because there is no request info
			responsewriters.InternalError(w, req, errors.New("no RequestInfo found in the context"))
			return
		}

		if longRunning(req, requestInfo) {
			handler.ServeHTTP(w, req)
			return
		}

		if err := wg.Add(1); err != nil {
			// shutdown delay duration has elapsed and SafeWaitGroup.Wait has been invoked,
			// this means 'WithRetryAfter' has started sending Retry-After response.
			// we are going to exempt the same set of requests that WithRetryAfter are
			// exempting from being rejected with a Retry-After response.
			if isRequestExemptFn(req) {
				handler.ServeHTTP(w, req)
				return
			}

			// When apiserver is shutting down, signal clients to retry
			// There is a good chance the client hit a different server, so a tight retry is good for client responsiveness.
			waitGroupWriteRetryAfterToResponse(w)
			return
		}

		defer wg.Done()
		handler.ServeHTTP(w, req)
	})
}
```

对于每个请求都会走到 `if err := wg.Add(1); err != nil`。

所以这里的逻辑就是，当『阀门』【`<-notAcceptingNewRequestCh.Signaled()` 】被关闭后，就会设置 `s.NonLongRunningRequestWaitGroup.Wait()`，然后再当每次新请求到来后，就会进入到 `if err := wg.Add(1); err != nil` 里的逻辑，即

* **先检查是否豁免**：`isRequestExemptFn(req)` 与 `WithRetryAfter` 使用同一套判断，例如健康检查、认证探针之类的请求，即使停机也允许继续执行。若命中豁免，直接 `handler.ServeHTTP`。
* **否则直接拒绝并告知重试**：调用 `waitGroupWriteRetryAfterToResponse` 写出 `Retry-After: 1` 头和 `{503 ServiceUnavailable}`（当启用 `ShutdownSendRetryAfter` 时外层会改成 429），提示客户端“apiserver 正在停机，请快速重试到其他节点”。

因此，这里实现的就是“闸门已落后，非豁免新请求一律被挡，并收到明确的 Retry-After 信号”。

完成之后，就会发送：`close(nonLongRunningRequestDrainedCh)` 信号。

### watch排空

当 NotAcceptingNewRequest 的信号接收到后，还会进入到清空 `Watch` 的逻辑，代码如下：

```go
	// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L661-#L692
	// wait for all in-flight watches to finish
	activeWatchesDrainedCh := make(chan struct{})
	go func() {
		defer close(activeWatchesDrainedCh)

		<-notAcceptingNewRequestCh.Signaled()
		if s.ShutdownWatchTerminationGracePeriod <= time.Duration(0) {
			klog.V(1).InfoS("[graceful-termination] not going to wait for active watch request(s) to drain")
			return
		}

		// Wait for all active watches to finish
		grace := s.ShutdownWatchTerminationGracePeriod
		activeBefore, activeAfter, err := s.WatchRequestWaitGroup.Wait(func(count int) (utilwaitgroup.RateLimiter, context.Context, context.CancelFunc) {
			qps := float64(count) / grace.Seconds()
			// TODO: we don't want the QPS (max requests drained per second) to
			//  get below a certain floor value, since we want the server to
			//  drain the active watch requests as soon as possible.
			//  For now, it's hard coded to 200, and it is subject to change
			//  based on the result from the scale testing.
			if qps < 200 {
				qps = 200
			}

			ctx, cancel := context.WithTimeout(context.Background(), grace)
			// We don't expect more than one token to be consumed
			// in a single Wait call, so setting burst to 1.
			return rate.NewLimiter(rate.Limit(qps), 1), ctx, cancel
		})
		klog.V(1).InfoS("[graceful-termination] active watch request(s) have drained",
			"duration", grace, "activeWatchesBefore", activeBefore, "activeWatchesAfter", activeAfter, "error", err)
	}()
```

这里的具体逻辑如下：

* **是否等待 watch**：如果 `ShutdownWatchTerminationGracePeriod <= 0`，直接返回，不对 watch 做额外等待（保持旧行为）。
* **否则进入排空流程**：
  * 设 `grace := ShutdownWatchTerminationGracePeriod`，作为最长等待时间。
  * 调用 `s.WatchRequestWaitGroup.Wait` 并传入一个函数来配置节流器：
    * 根据当前活跃 watch 数量 `count`，计算需要的 QPS，让所有 watch 在 `grace` 内排空（`qps = count / graceSeconds`，但不低于 200，以确保 drain 足够快）。
    * 创建 `context.WithTimeout(grace)`，在超时后强制结束。
    * 返回 `rate.NewLimiter`（突发 1）和上面的 context，用于控制同时结束 watch 的速率。
  * `Wait` 返回前后活跃 watch 数量以及可能的 error，随后在日志中打印排空情况。

这里完成之后，就会发送 `defer close(activeWatchesDrainedCh)` 。

## InFlightRequestsDrained

在主函数中，会启动一个协程来监听流量控制里的两个请求：

```go
	// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L694-700
	go func() {
		defer klog.V(1).InfoS("[graceful-termination] shutdown event", "name", drainedCh.Name())
		defer drainedCh.Signal()

		<-nonLongRunningRequestDrainedCh
		<-activeWatchesDrainedCh
	}()
```

在这里接收到了两个信号后，就会发送 `defer drainedCh.Signal()`，正好，这个请求，也就是在设置了 `s.ShutdownSendRetryAfter` 之后，`stopHttpServerCh` 监听的信号。

## Audit 收尾

只有在所有请求排空后才关掉审计后端，防止日志丢失。这里没有在流程图中进行展示，见如下代码：

```go
	// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L719-L724
	// Wait for all requests in flight to drain, bounded by the RequestTimeout variable.
	<-drainedCh.Signaled()

	if s.AuditBackend != nil {
		s.AuditBackend.Shutdown()
		klog.V(1).InfoS("[graceful-termination] audit backend shutdown completed")
	}
```

通过上面的学习，我们知道，`<-drainedCh.Signaled()` 到来的时候，就会排空所有请求，这里就会通过 `s.AuditBackend.Shutdown()` 来关闭掉审计后端。

## HTTP Server 停止

```go
	// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L583-L579
	notAcceptingNewRequestCh := s.lifecycleSignals.NotAcceptingNewRequest
	drainedCh := s.lifecycleSignals.InFlightRequestsDrained
	// Canceling the parent context does not immediately cancel the HTTP server.
	// We only inherit context values here and deal with cancellation ourselves.
	stopHTTPServerCtx, stopHTTPServer := context.WithCancelCause(context.WithoutCancel(ctx))
	go func() {
		defer stopHTTPServer(errors.New("time to stop HTTP server"))

		timeToStopHttpServerCh := notAcceptingNewRequestCh.Signaled()
		if s.ShutdownSendRetryAfter {
			timeToStopHttpServerCh = drainedCh.Signaled()
		}

		<-timeToStopHttpServerCh
	}()
```

这里在收到 `<-timeToStopHttpServerCh` 后，会开始执行 `stopHTTPServer(errors.New("time to stop HTTP server"))` 函数。

这里关闭 HTTP Server 的逻辑是这样的：

1. 在 `RunWithContext` 中，用 `context.WithCancelCause(context.WithoutCancel(ctx))` 生成 `stopHTTPServerCtx` 与 `stopHTTPServer`，然后把该 ctx 传给 `NonBlockingRunWithContext` 去启动 secure HTTP server，代码如下：

   ```go
   // 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L608	
   stoppedCh, listenerStoppedCh, err := s.NonBlockingRunWithContext(stopHTTPServerCtx, shutdownTimeout)
   ```

2. 然后在 `NonBlockingRunWithContext` 内部创建 `internalStopCh`【要点①】，并启动 goroutine 等待 `stopHTTPServerCtx.Done()`，一旦 Done，就 `close(internalStopCh)`【要点②】，代码如下：

   ```go
   // 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L748-L777
   // NonBlockingRunWithContext spawns the secure http server. An error is
   // returned if the secure port cannot be listened on.
   // The returned channel is closed when the (asynchronous) termination is finished.
   func (s preparedGenericAPIServer) NonBlockingRunWithContext(ctx context.Context, shutdownTimeout time.Duration) (<-chan struct{}, <-chan struct{}, error) {
   	// Use an internal stop channel to allow cleanup of the listeners on error.
   	internalStopCh := make(chan struct{}) // 要点①
   	var stoppedCh <-chan struct{}
   	var listenerStoppedCh <-chan struct{}
   	if s.SecureServingInfo != nil && s.Handler != nil {
   		var err error
   		stoppedCh, listenerStoppedCh, err = s.SecureServingInfo.Serve(s.Handler, shutdownTimeout, internalStopCh)
   		if err != nil {
   			close(internalStopCh)
   			return nil, nil, err
   		}
   	}
   
   	// Now that listener have bound successfully, it is the
   	// responsibility of the caller to close the provided channel to
   	// ensure cleanup.
   	go func() { // 要点②
   		<-ctx.Done()
   		close(internalStopCh)
   	}()
   
   	s.RunPostStartHooks(ctx)
   
   	if _, err := systemd.SdNotify(true, "READY=1\n"); err != nil {
   		klog.Errorf("Unable to send systemd daemon successful start message: %v\n", err)
   	}
   
   	return stoppedCh, listenerStoppedCh, nil
   }
   ```

3. 在调用 `s.SecureServingInfo.Serve`，也将 `internalStopCh` 进行传入，最终将 `internalStopCh` 传入 `RunServer`，

   ```go
   // 代码: staging\src\k8s.io\apiserver\pkg\server\secure_serving.go#L215-L266
   // RunServer spawns a go-routine continuously serving until the stopCh is
   // closed.
   // It returns a stoppedCh that is closed when all non-hijacked active requests
   // have been processed.
   // This function does not block
   // TODO: make private when insecure serving is gone from the kube-apiserver
   func RunServer(
   	server *http.Server,
   	ln net.Listener,
   	shutDownTimeout time.Duration,
   	stopCh <-chan struct{},
   ) (<-chan struct{}, <-chan struct{}, error) {
   	if ln == nil {
   		return nil, nil, fmt.Errorf("listener must not be nil")
   	}
   
   	// Shutdown server gracefully.
   	serverShutdownCh, listenerStoppedCh := make(chan struct{}), make(chan struct{})
   	go func() {
   		defer close(serverShutdownCh)
   		<-stopCh
   		ctx, cancel := context.WithTimeout(context.Background(), shutDownTimeout)
   		defer cancel()
   		err := server.Shutdown(ctx)
   		if err != nil {
   			klog.Errorf("Failed to shutdown server: %v", err)
   		}
   	}()
   
   	go func() {
   		defer utilruntime.HandleCrash()
   		defer close(listenerStoppedCh)
   
   		var listener net.Listener
   		listener = tcpKeepAliveListener{ln}
   		if server.TLSConfig != nil {
   			listener = tls.NewListener(listener, server.TLSConfig)
   		}
   
   		err := server.Serve(listener)
   
   		msg := fmt.Sprintf("Stopped listening on %s", ln.Addr().String())
   		select {
   		case <-stopCh:
   			klog.Info(msg)
   		default:
   			panic(fmt.Sprintf("%s due to error: %v", msg, err))
   		}
   	}()
   
   	return serverShutdownCh, listenerStoppedCh, nil
   }
   ```

   这里有两个协程，

   * 第一个协程是优雅关闭 `http server`，很好理解；
   * 第二个协程是关闭 `listener`，这里的一般会阻塞在 `err := server.Serve(listener)`，一旦 `stopCh` 关闭，`server.Shutdown` 会打断 `Serve`，后续的 `Select` 是用来进行判断是否正常退出，如果是 `stopCh` 到来，导致进入 Select，即正常退出，如果 `stopCh` 没有到来，则会进入到 `default` 中，从而 Panic。

## 收尾工作

在 `RunWithContext` 的开头，有一句 `defer` 语句，用来进行最后的资源释放：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L522
func (s preparedGenericAPIServer) RunWithContext(ctx context.Context) error {
	...
	// Clean up resources on shutdown.
	defer s.Destroy() //#L528
    ...
}
```

支持，Generic Server 的关停流程就详细讲解关闭，我们也对 `RunWithContext` 有了足够的了解。
