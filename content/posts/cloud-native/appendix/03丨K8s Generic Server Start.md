+++
title = "K8s Generic Server Start"
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

series_order = 6

+++

# K8s Generic Server Start

通过前面对 Generic Server 的学习，知道了 Generic Server 的启动代码位于：

`preparedGenericAPIServer` 的 `RunWithContext` 方法中，代码位置：`staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L522` 

我们下面来对 `RunWithContext` 方法进行详细解读：

> 这里关于 Generic Server 的关停只做大致了解，具体的 Graceful stop 见下一节

## 1. 初始化与信号布置

```go
	stopCh := ctx.Done()
	delayedStopCh := s.lifecycleSignals.AfterShutdownDelayDuration
	shutdownInitiatedCh := s.lifecycleSignals.ShutdownInitiated

	// Clean up resources on shutdown.
	defer s.Destroy()

	// If UDS profiling is enabled, start a local http server listening on that socket
	if s.UnprotectedDebugSocket != nil {
		go func() {
			defer utilruntime.HandleCrash()
			klog.Error(s.UnprotectedDebugSocket.Run(stopCh))
		}()
	}

	// spawn a new goroutine for closing the MuxAndDiscoveryComplete signal
	// registration happens during construction of the generic api server
	// the last server in the chain aggregates signals from the previous instances
	go func() {
		for _, muxAndDiscoveryCompletedSignal := range s.GenericAPIServer.MuxAndDiscoveryCompleteSignals() {
			select {
			case <-muxAndDiscoveryCompletedSignal:
				continue
			case <-stopCh:
				klog.V(1).Infof("haven't completed %s, stop requested", s.lifecycleSignals.MuxAndDiscoveryComplete.Name())
				return
			}
		}
		s.lifecycleSignals.MuxAndDiscoveryComplete.Signal()
		klog.V(1).Infof("%s has all endpoints registered and discovery information is complete", s.lifecycleSignals.MuxAndDiscoveryComplete.Name())
	}()

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

* ```go
      // 将上游 context 的 Done 通知缓存下来，作为所有协程统一感知的“停机”信号源
      stopCh := ctx.Done()
      // 记录优雅停机的“延迟窗口”信号，确保向负载均衡发出 readiness 失败后还能留出迁移时间
      delayedStopCh := s.lifecycleSignals.AfterShutdownDelayDuration
      // 记录“关机已启动”信号，用于驱动 /readyz 失败和后续的关机阶段
      shutdownInitiatedCh := s.lifecycleSignals.ShutdownInitiated
  
      // 在函数退出时统一释放 Listener、协程等资源，避免重复启停造成泄漏
      // Clean up resources on shutdown.
      defer s.Destroy()
  ```

* ```go
      // 如果启用了 UDS 调试/Profiling，额外起一个本地 HTTP 服务供问题定位，且生命周期与主 stop 信号保持同步
      // If UDS profiling is enabled, start a local http server listening on that socket
      if s.UnprotectedDebugSocket != nil {
          go func() {
              // 捕获调试服务 panic，防止影响主流程的优雅停机
              defer utilruntime.HandleCrash()
              // 复用 stopCh，让调试 socket 在 APIServer 退出时也能自动收敛
              klog.Error(s.UnprotectedDebugSocket.Run(stopCh))
          }()
      }
  ```

* ```go
      // 为所有委托的 GenericAPIServer 聚合“路由 + 发现信息已准备完毕”的信号，保证最终只在全部 ready 时对外宣告
      go func() {
          for _, muxAndDiscoveryCompletedSignal := range s.GenericAPIServer.MuxAndDiscoveryCompleteSignals() {
              select {
              case <-muxAndDiscoveryCompletedSignal:
                  // 某个 delegate 完成时继续等待剩余 delegate，直至整条链路都完成
                  continue
              case <-stopCh:
                  // 若尚未完成就收到停机信号，记录日志便于排查为何路由注册不完整
                  klog.V(1).Infof("haven't completed %s, stop requested", s.lifecycleSignals.MuxAndDiscoveryComplete.Name())
                  return
              }
          }
          // 所有 delegate 都 ready 后才真正 Signal，避免客户端访问到尚未注册的 API
          s.lifecycleSignals.MuxAndDiscoveryComplete.Signal()
          klog.V(1).Infof("%s has all endpoints registered and discovery information is complete", s.lifecycleSignals.MuxAndDiscoveryComplete.Name())
      }()
  ```

* ```go
      // 该协程负责管理“延迟关机”阶段：先宣告 readiness 失败，再等待预设间隔，让流量和请求逐步迁移走
      go func() {
          // 无论中途是否提前返回，都必须发出 delayedStop 信号，驱动后续阶段继续执行
          defer delayedStopCh.Signal()
          // 记录一次关机阶段日志，方便排查优雅停机的每个里程碑
          defer klog.V(1).InfoS("[graceful-termination] shutdown event", "name", delayedStopCh.Name())
  
          // 等待 stopCh 真正关闭，这表示需要启动优雅停机场景
          <-stopCh
  
          // As soon as shutdown is initiated, /readyz should start returning failure.
          // This gives the load balancer a window defined by ShutdownDelayDuration to detect that /readyz is red
          // and stop sending traffic to this server.
          // 第一时间让 readiness 探针失败，提示负载均衡器和客户端“该实例即将消失”，从而尽快切走新流量
          shutdownInitiatedCh.Signal()
          klog.V(1).InfoS("[graceful-termination] shutdown event", "name", shutdownInitiatedCh.Name())
  
          // 在 readiness 已失败的前提下继续等待配置的 ShutdownDelayDuration，给上游充足的时间迁移连接
          time.Sleep(s.ShutdownDelayDuration)
      }()
  ```

## 2. 控制 HTTP server 生命周期、审计后端启动

```go
	// close socket after delayed stopCh
	shutdownTimeout := s.ShutdownTimeout
	if s.ShutdownSendRetryAfter {
		// when this mode is enabled, we do the following:
		// - the server will continue to listen until all existing requests in flight
		//   (not including active long running requests) have been drained.
		// - once drained, http Server Shutdown is invoked with a timeout of 2s,
		//   net/http waits for 1s for the peer to respond to a GO_AWAY frame, so
		//   we should wait for a minimum of 2s
		shutdownTimeout = 2 * time.Second
		klog.V(1).InfoS("[graceful-termination] using HTTP Server shutdown timeout", "shutdownTimeout", shutdownTimeout)
	}

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

	// Start the audit backend before any request comes in. This means we must call Backend.Run
	// before http server start serving. Otherwise the Backend.ProcessEvents call might block.
	// AuditBackend.Run will stop as soon as all in-flight requests are drained.
	if s.AuditBackend != nil {
		if err := s.AuditBackend.Run(drainedCh.Signaled()); err != nil {
			return fmt.Errorf("failed to run the audit backend: %v", err)
		}
	}

	stoppedCh, listenerStoppedCh, err := s.NonBlockingRunWithContext(stopHTTPServerCtx, shutdownTimeout)
	if err != nil {
		return err
	}

	httpServerStoppedListeningCh := s.lifecycleSignals.HTTPServerStoppedListening
	go func() {
		<-listenerStoppedCh
		httpServerStoppedListeningCh.Signal()
		klog.V(1).InfoS("[graceful-termination] shutdown event", "name", httpServerStoppedListeningCh.Name())
	}()
```

* ```go
      // 根据当前是否启用 ShutdownSendRetryAfter 来决定 HTTP server 的关闭超时，保证在“先 drain 再关 listener”模式下至少等待 2 秒以配合 net/http 发 GOAWAY
      // close socket after delayed stopCh
      shutdownTimeout := s.ShutdownTimeout
      if s.ShutdownSendRetryAfter {
          // when this mode is enabled, we do the following:
          // - the server will continue to listen until all existing requests in flight
          //   (not including active long running requests) have been drained.
          // - once drained, http Server Shutdown is invoked with a timeout of 2s,
          //   net/http waits for 1s for the peer to respond to a GO_AWAY frame, so
          //   we should wait for a minimum of 2s
          // 若启用 Retry-After 模式，说明我们想让已有请求全部处理完再发 GOAWAY，因此强制使用 2 秒 shutdownTimeout 以满足协议等待
          shutdownTimeout = 2 * time.Second
          klog.V(1).InfoS("[graceful-termination] using HTTP Server shutdown timeout", "shutdownTimeout", shutdownTimeout)
      }
  ```

* ```go
      // 获取“停止接收新请求”与“所有存量请求排空”的信号，用于后续决定何时真正关闭 HTTP server
      notAcceptingNewRequestCh := s.lifecycleSignals.NotAcceptingNewRequest
      drainedCh := s.lifecycleSignals.InFlightRequestsDrained
      // 这里借助 context.WithoutCancel(ctx) 复制出一个不会随父 ctx 立即取消的上下文，再用 WithCancelCause 注入我们自己的关闭原因
      // Canceling the parent context does not immediately cancel the HTTP server.
      // We only inherit context values here and deal with cancellation ourselves.
      stopHTTPServerCtx, stopHTTPServer := context.WithCancelCause(context.WithoutCancel(ctx))
      go func() {
          // 当满足条件时调用 stopHTTPServer 并带上原因，驱动 HTTP server 进入 Shutdown
          defer stopHTTPServer(errors.New("time to stop HTTP server"))
  
          // 默认在不再接收新请求后就开始 shutdown；若启用 Retry-After 模式则必须等 in-flight 请求排空
          timeToStopHttpServerCh := notAcceptingNewRequestCh.Signaled()
          if s.ShutdownSendRetryAfter {
              timeToStopHttpServerCh = drainedCh.Signaled()
          }
  
          <-timeToStopHttpServerCh
      }()
  ```

* ```go
      // 在 HTTP server 对外接受流量前先启动审计后端，防止请求到来时审计通道还没准备好导致阻塞
      // Start the audit backend before any request comes in. This means we must call Backend.Run
      // before http server start serving. Otherwise the Backend.ProcessEvents call might block.
      // AuditBackend.Run will stop as soon as all in-flight requests are drained.
      if s.AuditBackend != nil {
          if err := s.AuditBackend.Run(drainedCh.Signaled()); err != nil {
              return fmt.Errorf("failed to run the audit backend: %v", err)
          }
      }
  ```

* ```go
      // 以不阻塞的方式启动 HTTP server：成功监听后返回“请求完成”与“监听器关闭”两个通道，供后续优雅停机同步使用
      stoppedCh, listenerStoppedCh, err := s.NonBlockingRunWithContext(stopHTTPServerCtx, shutdownTimeout)
      if err != nil {
          return err
      }
  ```

  > 这里的 `NonBlockingRunWithContext` 是在真正的启动 HTTP Server ，在后续会进行详细讲解。

* ```go
  // 监听底层 listener 停止事件，将其转换成生命周期信号 HTTPServerStoppedListening，方便诊断各阶段耗时
  httpServerStoppedListeningCh := s.lifecycleSignals.HTTPServerStoppedListening
  go func() {
      <-listenerStoppedCh
      httpServerStoppedListeningCh.Signal()
      klog.V(1).InfoS("[graceful-termination] shutdown event", "name", httpServerStoppedListeningCh.Name())
  }()
  ```

## 3. 停止接收新请求 & 排空不同类型的 in-flight 请求

```go
	// we don't accept new request as soon as both ShutdownDelayDuration has
	// elapsed and preshutdown hooks have completed.
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

	go func() {
		defer klog.V(1).InfoS("[graceful-termination] shutdown event", "name", drainedCh.Name())
		defer drainedCh.Signal()

		<-nonLongRunningRequestDrainedCh
		<-activeWatchesDrainedCh
	}()
```

* ```go
      // 只有当 ShutdownDelayDuration 窗口结束且预关机钩子执行完毕，才真正停止接受新请求，避免影响需要自清理的钩子
      // we don't accept new request as soon as both ShutdownDelayDuration has
      // elapsed and preshutdown hooks have completed.
      preShutdownHooksHasStoppedCh := s.lifecycleSignals.PreShutdownHooksStopped
      go func() {
          // 记录生命周期事件并最终发出 notAcceptingNewRequest 信号，让后续协程知道可以开始 drain
          defer klog.V(1).InfoS("[graceful-termination] shutdown event", "name", notAcceptingNewRequestCh.Name())
          defer notAcceptingNewRequestCh.Signal()
  
          // 先等待延迟窗口结束，保证负载均衡器完成流量迁移再关闭 handler 链
          <-delayedStopCh.Signaled()
  
          // 再等预关机钩子结束，因为钩子可能还在调用 API（例如释放租约），不能过早拒绝请求
          <-preShutdownHooksHasStoppedCh.Signaled()
      }()
  ```

* ```go
      // 该通道用于确认所有非长连接请求已 drain，配合 WaitGroup 统一阻塞
      // wait for all in-flight non-long running requests to finish
      nonLongRunningRequestDrainedCh := make(chan struct{})
      go func() {
          defer close(nonLongRunningRequestDrainedCh)
          defer klog.V(1).Info("[graceful-termination] in-flight non long-running request(s) have drained")
  
          // 必须等到“不再接受新请求”信号发出后，才能调用 WaitGroup.Wait() 禁止 handler 再放行新请求
          <-notAcceptingNewRequestCh.Signaled()
  
          // Wait() 会阻塞直到所有受 RequestTimeout 约束的请求完成；期间新请求会被 filter 拦截成 503/Retry-After 或连接被拒
          // TODO 解释了两种模式的差异：普通模式因 Server.Shutdown 已触发导致连接拒绝；Retry-After 模式则返回 429 直到 drain 完
          s.NonLongRunningRequestWaitGroup.Wait()
      }()
  ```

* ```go
      // 该通道负责追踪所有 watch 请求的排空情况，watch 属于长连接需要单独处理
      // wait for all in-flight watches to finish
      activeWatchesDrainedCh := make(chan struct{})
      go func() {
          defer close(activeWatchesDrainedCh)
  
          // 等待停止接受新请求后才处理 watch，否则还可能有新 watch 被创建
          <-notAcceptingNewRequestCh.Signaled()
          if s.ShutdownWatchTerminationGracePeriod <= time.Duration(0) {
              // 若未配置宽限期，直接跳过等待并记录日志，表示不阻塞在 watch drain 上
              klog.V(1).InfoS("[graceful-termination] not going to wait for active watch request(s) to drain")
              return
          }
  
          // 否则根据配置的宽限期，使用 WaitGroup + 限速策略逐步关闭 watch，避免瞬间切断造成客户端风暴
          grace := s.ShutdownWatchTerminationGracePeriod
          activeBefore, activeAfter, err := s.WatchRequestWaitGroup.Wait(func(count int) (utilwaitgroup.RateLimiter, context.Context, context.CancelFunc) {
              // 根据当前活跃 watch 数量计算需要的 drain QPS，确保在 grace 时间内大致处理完
              qps := float64(count) / grace.Seconds()
              // 下限 200，避免 watch 过多时 drain 速度太慢
              if qps < 200 {
                  qps = 200
              }
  
              // 为本次等待创建带超时的 context，保证最多等待 grace 时长
              ctx, cancel := context.WithTimeout(context.Background(), grace)
              // 每次 Wait 只放过 1 个请求，因此限速器 burst=1
              return rate.NewLimiter(rate.Limit(qps), 1), ctx, cancel
          })
          klog.V(1).InfoS("[graceful-termination] active watch request(s) have drained",
              "duration", grace, "activeWatchesBefore", activeBefore, "activeWatchesAfter", activeAfter, "error", err)
      }()
  ```

* ```go
      // 当普通请求和 watch 都 drain 完成后，才对外 Signal InFlightRequestsDrained，告知后续阶段可以继续
      go func() {
          defer klog.V(1).InfoS("[graceful-termination] shutdown event", "name", drainedCh.Name())
          defer drainedCh.Signal()
  
          <-nonLongRunningRequestDrainedCh
          <-activeWatchesDrainedCh
      }()
  ```

## 4. 收尾阶段：执行 pre-shutdown hook、关闭审计 & HTTP server

```go
	klog.V(1).Info("[graceful-termination] waiting for shutdown to be initiated")
	<-stopCh

	// run shutdown hooks directly. This includes deregistering from
	// the kubernetes endpoint in case of kube-apiserver.
	func() {
		defer func() {
			preShutdownHooksHasStoppedCh.Signal()
			klog.V(1).InfoS("[graceful-termination] pre-shutdown hooks completed", "name", preShutdownHooksHasStoppedCh.Name())
		}()
		err = s.RunPreShutdownHooks()
	}()
	if err != nil {
		return err
	}

	// Wait for all requests in flight to drain, bounded by the RequestTimeout variable.
	<-drainedCh.Signaled()

	if s.AuditBackend != nil {
		s.AuditBackend.Shutdown()
		klog.V(1).InfoS("[graceful-termination] audit backend shutdown completed")
	}

	// wait for stoppedCh that is closed when the graceful termination (server.Shutdown) is finished.
	<-listenerStoppedCh
	<-stoppedCh

	klog.V(1).Info("[graceful-termination] apiserver is exiting")
	return nil
```

* ```go
      // 在正式收到 stop 信号前保持阻塞，并打印日志提示当前处于“等待关机被触发”的阶段
      klog.V(1).Info("[graceful-termination] waiting for shutdown to be initiated")
      // 阻塞于 ctx.Done()，一旦控制面决定停机，这里立即放行，开始执行关机序列
      <-stopCh
  ```

* ```go
      // run shutdown hooks directly. This includes deregistering from
      // the kubernetes endpoint in case of kube-apiserver.
      // 进入预关机钩子阶段：这些钩子负责从集群指标、租约、服务注册等地方撤销自身，必须同步完成以免留下脏数据
      func() {
          defer func() {
              // 不论钩子执行结果如何都要广播“pre-shutdown hooks 完成”，方便前面等待该信号的协程（如停止接收新请求）继续推进
              preShutdownHooksHasStoppedCh.Signal()
              klog.V(1).InfoS("[graceful-termination] pre-shutdown hooks completed", "name", preShutdownHooksHasStoppedCh.Name())
          }()
          // 实际执行所有注册的 pre-shutdown hook，若其中任意一步失败会返回错误
          err = s.RunPreShutdownHooks()
      }()
      // 如果任意钩子报错，直接返回 err，阻止继续关机流程，以便调用方感知并处理
      if err != nil {
          return err
      }
  ```

* ```go
      // Wait for all requests in flight to drain, bounded by the RequestTimeout variable.
      // 此时挂起等待 InFlightRequestsDrained 信号，确保所有非长连接和 watch 请求都已经安全完成或超时
      <-drainedCh.Signaled()
  
      // audit 后端需要在所有请求结束后显式关闭，以冲刷遗留的审计日志并释放资源
      if s.AuditBackend != nil {
          s.AuditBackend.Shutdown()
          klog.V(1).InfoS("[graceful-termination] audit backend shutdown completed")
      }
  ```

* ```go
      // listenerStoppedCh 代表底层 socket 停止监听；stoppedCh 表示所有请求处理完成
      // 依次等待它们，保证 HTTP server 的优雅关闭完整结束
      // wait for stoppedCh that is closed when the graceful termination (server.Shutdown) is finished.
      <-listenerStoppedCh
      <-stoppedCh
  
      // 全部阶段完成后记录最终日志并返回 nil，代表 APIServer 成功退出
      klog.V(1).Info("[graceful-termination] apiserver is exiting")
      return nil
  ```

# NonBlockingRunWithContext

主要流程如下：

![image-20251122225334441](http://images.liangning7.cn/typora/202511222253598.png)

## preparedGenericAPIServer → NonBlockingRun

```go
// 代码: staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go
	stoppedCh, listenerStoppedCh, err := s.NonBlockingRunWithContext(stopHTTPServerCtx, shutdownTimeout)
	if err != nil {
		return err
	}
```

这里的 `NonBlockingRunWithContext` 是在真正启动 HTTP Server。对应时序图中的 **preparedGenericAPIServer → NonBlockingRun**

再来看看 `NonBlockingRunWithContext` 方法：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\genericapiserver.go#L748
// NonBlockingRunWithContext spawns the secure http server. An error is
// returned if the secure port cannot be listened on.
// The returned channel is closed when the (asynchronous) termination is finished.
func (s preparedGenericAPIServer) NonBlockingRunWithContext(ctx context.Context, shutdownTimeout time.Duration) (<-chan struct{}, <-chan struct{}, error) {
	// Use an internal stop channel to allow cleanup of the listeners on error.
	internalStopCh := make(chan struct{})
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
	go func() {
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

## SecureServingInfo.Serve

若 `SecureServingInfo` 不为空，`NonBlockingRunWithContext` 会执行 `s.SecureServingInfo.Serve(...)`，然后在 Server 中准备 TLS 配置、动态证书以及真正的 `http.Server` 实例。

进一步，再看一步 Server 的详细信息：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\secure_serving.go#L155
// Serve runs the secure http server. It fails only if certificates cannot be loaded or the initial listen call fails.
// The actual server loop (stoppable by closing stopCh) runs in a go routine, i.e. Serve does not block.
// It returns a stoppedCh that is closed when all non-hijacked active requests have been processed.
// It returns a listenerStoppedCh that is closed when the underlying http Server has stopped listening.
func (s *SecureServingInfo) Serve(handler http.Handler, shutdownTimeout time.Duration, stopCh <-chan struct{}) (<-chan struct{}, <-chan struct{}, error) {
	if s.Listener == nil {
		return nil, nil, fmt.Errorf("listener must not be nil")
	}

	tlsConfig, err := s.tlsConfig(stopCh) //要点①
	if err != nil {
		return nil, nil, err
	}

	secureServer := &http.Server{
		Addr:           s.Listener.Addr().String(),
		Handler:        handler,
		MaxHeaderBytes: 1 << 20,
		TLSConfig:      tlsConfig,

		IdleTimeout:       90 * time.Second, // matches http.DefaultTransport keep-alive timeout
		ReadHeaderTimeout: 32 * time.Second, // just shy of requestTimeoutUpperBound
	}

	if !s.DisableHTTP2 { //要点②
		// At least 99% of serialized resources in surveyed clusters were smaller than 256kb.
		// This should be big enough to accommodate most API POST requests in a single frame,
		// and small enough to allow a per connection buffer of this size multiplied by `MaxConcurrentStreams`.
		const resourceBody99Percentile = 256 * 1024

		http2Options := &http2.Server{
			IdleTimeout: 90 * time.Second, // matches http.DefaultTransport keep-alive timeout
			// shrink the per-stream buffer and max framesize from the 1MB default while still accommodating most API POST requests in a single frame
			MaxUploadBufferPerStream: resourceBody99Percentile,
			MaxReadFrameSize:         resourceBody99Percentile,
		}

		// use the overridden concurrent streams setting or make the default of 250 explicit so we can size MaxUploadBufferPerConnection appropriately
		if s.HTTP2MaxStreamsPerConnection > 0 {
			http2Options.MaxConcurrentStreams = uint32(s.HTTP2MaxStreamsPerConnection)
		} else {
			// match http2.initialMaxConcurrentStreams used by clients
			// this makes it so that a malicious client can only open 400 streams before we forcibly close the connection
			// https://github.com/golang/net/commit/b225e7ca6dde1ef5a5ae5ce922861bda011cfabd
			http2Options.MaxConcurrentStreams = 100
		}

		// increase the connection buffer size from the 1MB default to handle the specified number of concurrent streams
		http2Options.MaxUploadBufferPerConnection = http2Options.MaxUploadBufferPerStream * int32(http2Options.MaxConcurrentStreams)
		// apply settings to the server
		if err := http2.ConfigureServer(secureServer, http2Options); err != nil {
			return nil, nil, fmt.Errorf("error configuring http2: %v", err)
		}
	}

	// use tlsHandshakeErrorWriter to handle messages of tls handshake error
	tlsErrorWriter := &tlsHandshakeErrorWriter{os.Stderr}
	tlsErrorLogger := log.New(tlsErrorWriter, "", 0)
	secureServer.ErrorLog = tlsErrorLogger

	klog.Infof("Serving securely on %s", secureServer.Addr)
	return RunServer(secureServer, s.Listener, shutdownTimeout, stopCh) //要点③
}
```

### 要点① **tlsConfig 构建 + 动态证书控制器**

在要点①处，调用 `s.tlsConfig(stopCh)`，生成 `*tls.Config`。其配置的流程如下：

![image-20251123022644748](http://images.liangning7.cn/typora/202511230226951.png)

我们再来看看源代码是如何实现的？

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\secure_serving.go#L45
// tlsConfig produces the tls.Config to serve with.
func (s *SecureServingInfo) tlsConfig(stopCh <-chan struct{}) (*tls.Config, error) {
	...
	if s.ClientCA != nil || s.Cert != nil || len(s.SNICerts) > 0 {
		dynamicCertificateController := dynamiccertificates.NewDynamicServingCertificateController( // 要点①
			tlsConfig,
			s.ClientCA,
			s.Cert,
			s.SNICerts,
			nil, // TODO see how to plumb an event recorder down in here. For now this results in simply klog messages.
		)

		if s.ClientCA != nil { // 要点②
			s.ClientCA.AddListener(dynamicCertificateController)
		}
		if s.Cert != nil { // 要点②
			s.Cert.AddListener(dynamicCertificateController)
		}
		// generate a context from stopCh. This is to avoid modifying files which are relying on apiserver
		// TODO: See if we can pass ctx to the current method
		ctx, cancel := context.WithCancel(context.Background())
		go func() {
			select {
			case <-stopCh:
				cancel() // stopCh closed, so cancel our context
			case <-ctx.Done():
			}
		}()
		// start controllers if possible
		if controller, ok := s.ClientCA.(dynamiccertificates.ControllerRunner); ok {
			// runonce to try to prime data.  If this fails, it's ok because we fail closed.
			// Files are required to be populated already, so this is for convenience.
			if err := controller.RunOnce(ctx); err != nil {
				klog.Warningf("Initial population of client CA failed: %v", err)
			}

			go controller.Run(ctx, 1) // 要点③
		}
		if controller, ok := s.Cert.(dynamiccertificates.ControllerRunner); ok {
			// runonce to try to prime data.  If this fails, it's ok because we fail closed.
			// Files are required to be populated already, so this is for convenience.
			if err := controller.RunOnce(ctx); err != nil {
				klog.Warningf("Initial population of default serving certificate failed: %v", err)
			}

			go controller.Run(ctx, 1) // 要点③
		}
		for _, sniCert := range s.SNICerts {
			sniCert.AddListener(dynamicCertificateController) // 要点②
			if controller, ok := sniCert.(dynamiccertificates.ControllerRunner); ok {
				// runonce to try to prime data.  If this fails, it's ok because we fail closed.
				// Files are required to be populated already, so this is for convenience.
				if err := controller.RunOnce(ctx); err != nil {
					klog.Warningf("Initial population of SNI serving certificate failed: %v", err)
				}

				go controller.Run(ctx, 1) // 要点③
			}
		}

		// runonce to try to prime data.  If this fails, it's ok because we fail closed.
		// Files are required to be populated already, so this is for convenience.
		if err := dynamicCertificateController.RunOnce(); err != nil {
			klog.Warningf("Initial population of dynamic certificates failed: %v", err)
		}
		go dynamicCertificateController.Run(1, stopCh)

		tlsConfig.GetConfigForClient = dynamicCertificateController.GetConfigForClient
	}

	return tlsConfig, nil
}
```

* 首先在要点①处，通过 `dynamiccertificates.NewDynamicServingCertificateController` 构建出了 DynamicFileCAContent 控制器实例。
* 而后，通过要点②处的代码，将 `Cert`，`SNICerts` 和 `ClientCA` 的控制器的 Listener 设置为 DynamicFileCAContent 控制器。
* 最后，通过要点③处的代码，开始监听。

### 要点②

要点②，主要是对 HTTP2 的一些配置。

### 要点③ **secure_serving.go → RunServer**

`RunServer` 主要做两件事：

* 起一个 goroutine 持续监听 `stopCh`，收到后用 `server.Shutdown(ctx)` 做优雅关闭；
* 再起一个 goroutine 真正调用 server.Serve(listener)，并把 listener 包装成 tcpKeepAliveListener 以及 TLS listener

代码如下：

```go
// 代码: staging\src\k8s.io\apiserver\pkg\server\secure_serving.go#L221
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
	go func() {// 要点①
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

		err := server.Serve(listener) // 要点②

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

在要点①处，进行优雅关闭的处理，在要点②处，`http.Server` 开始对外提供服务，直到 `stopCh` 关闭，再由 `server.Shutdown` 配合 drain 逻辑完成优雅退出。

关闭逻辑如下：

* 关闭逻辑：要点①处的 goroutine 在 `<-stopCh` 后调用 `server.Shutdown(ctx)`，它会优雅地关闭所有活跃连接并停止监听端口。
* 当 `server.Shutdown` 执行时，`server.Serve(listener)` 会收到 `http.ErrServerClosed`，跳到 `msg := ...` 后的 `select`，此时因为 `stopCh` 已关闭，进入 `case <-stopCh` 分支，仅记录日志然后退出，从而结束该 goroutine。
* 如果 Server 是因为其它错误返回（`stopCh` 未关闭），就会走 `default` 分支触发 panic，表示监听意外终止。

