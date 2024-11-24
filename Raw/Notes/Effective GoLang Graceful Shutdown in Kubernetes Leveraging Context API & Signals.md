---
title: "Effective GoLang Graceful Shutdown in Kubernetes: Leveraging Context API & Signals"
source: https://citymall.engineering/effective-golang-graceful-shutdown-in-kubernetes-leveraging-context-api-signals-47b767ed598
clipped: 2023-10-18
published: 
category: k8s
tags:
  - development
  - golang
read: true
---

![[Raw/Media/Resources/2ec03a19267b359ae08a2b9d1466ceda_MD5.png]]

The above picture narrates the Kubernetes traffic shifting story.

Two points we need to know for a graceful shutdown of a golang service.

1.  Context Cancellations Synced with Framework Shutdown (HTTP/GRPC)
2.  Context Cancellations @ Request Level

The below piece of code should be how you handle the entry and exit of your code.

```go
func main() {  
    mainCtx, stop := signal.NotifyContext(  
        context.Background(),   
        syscall.SIGINT,   
        syscall.SIGTERM,  
    )  
    defer stop()

    var wg sync.WaitGroup  
    wg.Add(1)  
    cs.InitCS(mainCtx, &wg)

    wg.Add(1)  
    go cs.StartServer(mainCtx, &wg)

    wg.Add(1)  
    go cs.StartListener(mainCtx, &wg)

          
    sigs := make(chan os.Signal, 1)  
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

    sig := <-sigs  
    cs.STATUS.Store(0) 

    log.Info().Msgf("Got signal, beginning shutdown %s", sig)  
    wg.Wait()  
    log.Info().Msg("All goroutines finished, shutting down!")  
}
```

From the picture above, we get the idea that we should wait before stopping our HTTP and GRPC servers after a SIGTERM is sent by Kubernetes.

After receiving SIGTERM we should do the following -

1.  Immediately start to fail the readiness probe
2.  Wait for **WaitSeconds** amount of time, where  
    **WaitSeconds = KubeFailThreshold x KubePeriodSeconds + DeltaSeconds**
3.  After WaitSeconds is over, stop and clear up all server-level resources — like grpcServer.GracefulStop(), http.Shutdown(), closing DB connections, clearing in-memory stores, etc.

*You can get* ***KubePeriodSeconds*** *and* ***KubeFailThreshold*** *from your Kubernetes Values.yaml file*

![[Raw/Media/Resources/7a9db3f86eca2b6942992474b1f0f76a_MD5.png]]

***DeltaSeconds*** *is fault tolerance, recommended to use the value of* ***KubePeriodSeconds*** *here*

Here is how you should handle probes and HTTP server shutdown

```go
var STATUS atomic.Int32  
var KUBE_PERIOD_SECONDS = 2  
var KUBE_FAILURE_THRESHOLD = 2  
var DELTA_SECONDS = 2  
var WAIT_SECONDS = (KUBE_FAILURE_THRESHOLD*KUBE_PERIOD_SECONDS + DELTA_SECONDS)

func StartServer(mainCtx context.Context, wg \*sync.WaitGroup) {  
 defer wg.Done()  
 router := gin.Default()

 router.GET("/ping/live", func(c *gin.Context) {  
  c.JSON(http.StatusOK, gin.H{"message": "pong"})  
 })

 router.GET("/ping/ready", func(c *gin.Context) {  
  if STATUS.Load() != 2 {  
   c.JSON(http.StatusServiceUnavailable, gin.H{"message": "Service Unavailable"})  
   return  
  }  
  c.JSON(http.StatusOK, gin.H{"message": "pong"})  
 })

 server := &http.Server{  
  Addr:    ":8981",  
  Handler: router,  
 }
   
 go func() {  
  time.AfterFunc(1*time.Second, func() {  
   STATUS.Add(1)  
  })  
  if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {  
   panic(err)  
  }  
 }()  
 log.Info().Msg("HTTP server starting at port: 8981 ")  
   
 quit := make(chan os.Signal, 1)  
   
 signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)  
 sig := <-quit  
 log.Info().Str("Component", "StartServer").Msgf("Got %v signal. HTTP Server will shut down in %v seconds", sig, WAIT_SECONDS)  
 time.Sleep(time.Duration(WAIT\_SECONDS) * time.Second)

 log.Info().Str("Component", "StartServer").Msg("Shutting down server...")
   
 ctx, cancel := context.WithTimeout(mainCtx, 5*time.Second)  
 defer cancel()

 if err := server.Shutdown(ctx); err != nil {  
  log.Error().Str("Component", "StartServer").Msgf("Server forced to shutdown: %v", err)  
 }

 log.Info().Str("Component", "StartServer").Msg("Server exiting!")  
}
```

And here’s how you should handle grpc Shutdown

```go
func StartListener(mainCtx context.Context, wg \*sync.WaitGroup) {  
 defer wg.Done()  
 port := utils.GetConfig().ServiceGRPCPort  
 log.Info().Msgf("GRPC server starting at port: %v ", port)

 lis, err := net.Listen("tcp", "0.0.0.0:"+port)  
 if err != nil {  
  fmt.Println(err)  
  panic("failed to listen at port " + port)  
 }

 s := grpc.NewServer()

 reflection.Register(s)

 grpcServer := server{}  
 pb.RegisterServiceServer(s, &grpcServer)

   
 go func() {  
  time.AfterFunc(1\*time.Second, func() {  
   STATUS.Add(1)  
  })  
  if err := s.Serve(lis); err != nil {  
   panic(err)  
  }  
 }()

   
 c := make(chan os.Signal, 1)  
   
 signal.Notify(c, os.Interrupt, syscall.SIGTERM)

   
 sig := <-c

 log.Info().Str("Component", "StartListener").Msgf("Got %v signal. GRPC Server will shut down in %v seconds", sig, WAIT\_SECONDS)  
 time.Sleep(time.Duration(WAIT\_SECONDS) \* time.Second)

 log.Info().Str("Component", "StartListener").Msg("Gracefully stopping server...")

   
   
 s.GracefulStop()

 log.Info().Str("Component", "StartListener").Msg("Server has stopped")  
}
```

**IMPORTANT:** Any background workers, cron-jobs, etc should use mainCtx for shutdown. For example, if you have a job that runs in the background and updates data periodically then you should use the following handling.

```go
func updateDataInBackground(ctx context.Context, wg \*sync.WaitGroup) {  
 defer wg.Done()  
 cctx, cancel := context.WithCancel(ctx)  
 defer cancel()  
 ticker := time.NewTicker(30 \* time.Second)

 for {  
  select {  
  case <-cctx.Done():  
   log.Info().Msg("Stopping data update")  
   return  
  case <-ticker.C:  
     
  }  
 }  
}
```

Every resource that we use for serving a single request need to be closed during the shutdown. This can be done easily using the Golang context.

We have to follow a code practice that every function which is directly or indirectly called during the serving of a request, need to have the following signature and handling.

```go
func Operation(ctx context.Context, .....) {  
  cctx, cancel := context.WithCancel(ctx)  
  defer cancel()

      
  controller.SaveLog(cctx, ....)  
}
```

The parent context at the request level is gin.Context if we’re using the `github.com/gin-gonic/gin` framework or grpc context.Context in the case of `google.golang.org/grpc`

For example, in the case of gin your handling should be -

```go
router.POST("/save-log", func(ctx \*gin.Context) {  
  cctx, cancel := context.WithCancel(ctx.Request.Context())  
  defer cancel()

      
    
  result, err := CS.SaveLog(cctx, .......

      
})    
```

in the case of grpc,

```go
func (s \*server) SaveLog(ctx context.Context, req \*pb.SaveLogReq) (\*pb.SaveLogResp, error) {  
 cctx, cancel := context.WithCancel(ctx)  
 defer cancel()  
   
   
 result, err := CS.SaveLog(cctx, .......

    
}
```

**PLEASE MAKE SURE YOU DO NOT USE context.TODO() OR context.Background() ANYWHERE YOU CALL A RESOURCE WHICH ACCEPTS A CONTEXT.**
