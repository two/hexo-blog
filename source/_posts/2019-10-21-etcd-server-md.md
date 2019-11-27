title: etcd server 启动
date: 2019-10-21 11:01:48
tags: [etcd Go]
---

# 入口

文件入口:
```go
// main.go

package main

import "go.etcd.io/etcd/etcdmain"

func main() {
    etcdmain.Main()
}
```

## 开始启动过程
1. 平台支持检查
2. 启动参数解析
3. 启动服务或者 proxy

```go
// etcdmain/main.go

func Main() {
    // 检查支持的平台
    checkSupportArch()

    // 参数解析
    if len(os.Args) > 1 {
        cmd := os.Args[1]
        if covArgs := os.Getenv("ETCDCOV_ARGS"); len(covArgs) > 0 {
            args := strings.Split(os.Getenv("ETCDCOV_ARGS"), "\xe7\xcd")[1:]
            rootCmd.SetArgs(args)
            cmd = "grpc-proxy" // 如果设置了 ETCDCOV_ARGS 环境变量，就是以 grpc-proxy 方式启动
        }
        switch cmd {
        case "gateway", "grpc-proxy": // 如果设置了 cmd变量，就以变量的形式启动,包括 gateway, grpc-proxy 两种方式
            if err := rootCmd.Execute(); err != nil { // rootCmd.Execute 就等于调用 `etcd gateway` 或者 `etcd grpc-proxy`, 其它的参数不支持，调用的是 `etcd`
                fmt.Fprint(os.Stderr, err)
                os.Exit(1)
            }
            return
        }
    }

    startEtcdOrProxyV2()
}
```

## 配置检查 & 启动 etcdServer

```go
// etcdmain/etcd.go

func startEtcdOrProxyV2() {
    grpc.EnableTracing = false

    cfg := newConfig()                             // 默认配置项
    defaultInitialCluster := cfg.ec.InitialCluster // 对应配置文件中的 ETCD_INITIAL_CLUSTER 变量，默认的集群节点配置

    ...

    defer func() {
        logger := cfg.ec.GetLogger()
        if logger != nil {
            logger.Sync()
        }
    }()

    // 重新修改一下 cluster 配置中的 cluster 相关的信息，比如获取当前计算机的 hostname 代替 0.0.0.0 或 localhost
    defaultHost, dhErr := (&cfg.ec).UpdateDefaultClusterFromName(defaultInitialCluster)
    ...

    var stopped <-chan struct{}
    var errc <-chan error

    which := identifyDataDirOrDie(cfg.ec.GetLogger(), cfg.ec.Dir) // 检查存储数据的目录, 返回数据存储的类型是成员还是 proxy 等
    if which != dirEmpty {                                        // 节点目录不为空，证明不是第一次使用，恢复之前的配置
        ...
        switch which {
        case dirMember: // 如果是成员类型，需要开启一个 etcd 服务
            stopped, errc, err = startEtcd(&cfg.ec)
        case dirProxy: // 如果是proxy, 则开启一个 proxy 服务
            err = startProxy(cfg)
        default:
            ...
        }
    } else { // 如果为空，则根据参数启动服务
        shouldProxy := cfg.isProxy()
        if !shouldProxy { // 如果不是 proxy, 则启动一个正常的 server
            stopped, errc, err = startEtcd(&cfg.ec)
            if derr, ok := err.(*etcdserver.DiscoveryError); ok && derr.Err == v2discovery.ErrFullCluster {
                if cfg.shouldFallbackToProxy() {
                    ...
                    shouldProxy = true
                }
            } 
        ...
        if shouldProxy { // 如果是 proxy ,则启动一个 proxy 服务
            err = startProxy(cfg)
        }
    }
    ...
    osutil.HandleInterrupts(lg) // 接收外界信号

    // At this point, the initialization of etcd is done.
    // The listeners are listening on the TCP ports and ready
    // for accepting connections. The etcd instance should be
    // joined with the cluster and ready to serve incoming
    // connections.
    notifySystemd(lg) // 把型号发送给正在运行的 etcd 守护进程

    select { // 进入阻塞状态，除非出现错误或者服务关闭
    case lerr := <-errc:
        // fatal out on listener errors
        if lg != nil {
            lg.Fatal("listener failed", zap.Error(lerr))
        } else {
            plog.Fatal(lerr)
        }
    case <-stopped:
    }
    osutil.Exit(0)
}

// startEtcd runs StartEtcd in addition to hooks needed for standalone etcd.
func startEtcd(cfg *embed.Config) (<-chan struct{}, <-chan error, error) {
    e, err := embed.StartEtcd(cfg) // 根据配置开启一个 etcd server
    if err != nil {
        return nil, nil, err
    }
    osutil.RegisterInterruptHandler(e.Close) // 注册通过信号关闭时的回调函数
    select {                                 // 进入阻塞,除非接收到下面的信号
    case <-e.Server.ReadyNotify(): // wait for e.Server to join the cluster  阻塞，直到当前 server 注册到了集群中
    case <-e.Server.StopNotify(): // publish aborted from 'ErrStopped' 注册失败
    }
    return e.Server.StopNotify(), e.Err(), nil
}

```

##  开启 peer , client 和 metrics server

1. etcd Server 启动会开起多个 server
2. peer server 用于 etcd 集群之间的选举，探活, 默认端口 2380
3. client 用于监听客户端请求，处理客户端读写, 默认端口 2379
4. metrics  用户监控集群状态，需要手动指定端口，否则不开启

```go
// embed/etcd.go

// StartEtcd 回开启一个 etcd server, 并且接收 HTTP 请求，但是这个函数并不会保证加入到了集群中
// 加入集群是由  Etcd.Server.ReadyNotify() 来实现的
// StartEtcd launches the etcd server and HTTP handlers for client/server communication.
// The returned Etcd.Server is not guaranteed to have joined the cluster. Wait
// on the Etcd.Server.ReadyNotify() channel to know when it completes and is ready for use.
func StartEtcd(inCfg *Config) (e *Etcd, err error) {
    if err = inCfg.Validate(); err != nil { // 检查一些参数是否合法
        return nil, err
    }
    serving := false
    e = &Etcd{cfg: *inCfg, stopc: make(chan struct{})}
    cfg := &e.cfg
    defer func() {
        if e == nil || err == nil {
            return
        }
        if !serving {
            // errored before starting gRPC server for serveCtx.serversC
            for _, sctx := range e.sctxs {
                close(sctx.serversC)
            }
        }
        e.Close()
        e = nil
    }()
    // 开启 peer server, 默认端口 2380
    if e.Peers, err = configurePeerListeners(cfg); err != nil { // 节点数据赋值
        return e, err
    }
    // 开启 client server,  默认端口 2379，并且支持多协议(grpc,http, https)
    if e.sctxs, err = configureClientListeners(cfg); err != nil { // client 数据赋值
        return e, err
    }

    for _, sctx := range e.sctxs { // 当前 server ctx 记录
        e.Clients = append(e.Clients, sctx.l)
    }

    // 注册 token
    memberInitialized := true
    if !isMemberInitialized(cfg) {
        memberInitialized = false
        urlsmap, token, err = cfg.PeerURLsMapAndToken("etcd")
        if err != nil {
            return e, fmt.Errorf("error setting up initial cluster: %v", err)
        }
    }

    // AutoCompactionRetention defaults to "0" if not set.
    if len(cfg.AutoCompactionRetention) == 0 {
        cfg.AutoCompactionRetention = "0"
    }
    autoCompactionRetention, err := parseCompactionRetention(cfg.AutoCompactionMode, cfg.AutoCompactionRetention)
    if err != nil {
        return e, err
    }

    backendFreelistType := parseBackendFreelistType(cfg.ExperimentalBackendFreelistType)
    // 根据配置新建一个 server 对象
    if e.Server, err = etcdserver.NewServer(srvcfg); err != nil {
        return e, err
    }

    // buffer channel 保证服务关闭的时候不会阻塞
    // buffer channel so goroutines on closed connections won't wait forever
    e.errc = make(chan error, len(e.Peers)+len(e.Clients)+2*len(e.sctxs))

    // newly started member ("memberInitialized==false")
    // does not need corruption check
    if memberInitialized {
        if err = e.Server.CheckInitialHashKV(); err != nil {
            // set "EtcdServer" to nil, so that it does not block on "EtcdServer.Close()"
            // (nothing to close since rafthttp transports have not been started)
            e.Server = nil
            return e, err
        }
    }
    e.Server.Start() // 开启一个 etcd server

    // 与每个 peer 保持通信
    if err = e.servePeers(); err != nil {
        return e, err
    }
    //  开启 server 与每个 client 保持通信, 可以同时支持多重协议
    if err = e.serveClients(); err != nil {
        return e, err
    }
    // 与每个 metrics 保持通信
    if err = e.serveMetrics(); err != nil {
        return e, err
    }
    ...
    serving = true
    return e, nil
}
```

`e.Server.Start()`函数实现:

```go
// Start performs any initialization of the Server necessary for it to
// begin serving requests. It must be called before Do or Process.
// Start must be non-blocking; any long-running server functionality
// should be implemented in goroutines.
func (s *EtcdServer) Start() {
    s.start()
    // 下面的函数将在 server 关闭后等执行完成
    s.goAttach(func() { s.adjustTicks() })
    s.goAttach(func() { s.publish(s.Cfg.ReqTimeout()) })
    s.goAttach(s.purgeFile)
    s.goAttach(func() { monitorFileDescriptor(s.getLogger(), s.stopping) })
    s.goAttach(s.monitorVersions)
    s.goAttach(s.linearizableReadLoop)
    s.goAttach(s.monitorKVHash)
}

// start prepares and starts server in a new goroutine. It is no longer safe to
// modify a server's fields after it has been sent to Start.
// This function is just used for testing.
func (s *EtcdServer) start() {
    ...
    // 通过一个 goroutine 开启服务
    // TODO: if this is an empty log, writes all peer infos
    // into the first entry
    go s.run()
}

func (s *EtcdServer) run() {
    lg := s.getLogger()
    sn, err := s.r.raftStorage.Snapshot()

    // asynchronously accept apply packets, dispatch progress in-order
    sched := schedule.NewFIFOScheduler()
    ...
    s.r.start(rh)

    defer func() {
        s.wgMu.Lock() // block concurrent waitgroup adds in goAttach while stopping
        close(s.stopping)
        s.wgMu.Unlock()
        s.cancel()

        sched.Stop()

        // wait for gouroutines before closing raft so wal stays open
        s.wg.Wait()

        s.SyncTicker.Stop()

        // must stop raft after scheduler-- etcdserver can leak rafthttp pipelines
        // by adding a peer after raft stops the transport
        s.r.stop()

        // kv, lessor and backend can be nil if running without v3 enabled
        // or running unit tests.
        if s.lessor != nil {
            s.lessor.Stop()
        }
        if s.kv != nil {
            s.kv.Close()
        }
        if s.authStore != nil {
            s.authStore.Close()
        }
        if s.be != nil {
            s.be.Close()
        }
        if s.compactor != nil {
            s.compactor.Stop()
        }
        close(s.done)
    }()
    var expiredLeaseC <-chan []*lease.Lease
    if s.lessor != nil {
        expiredLeaseC = s.lessor.ExpiredLeasesC()
    }
    for {
        select {
        case ap := <-s.r.apply(): // 数据更新
            f := func(context.Context) { s.applyAll(&ep, &ap) }
            sched.Schedule(f)
        case leases := <-expiredLeaseC: // 租期过期处理
            s.goAttach(func() {
            ...
             })
        case err := <-s.errorc: // 出现错误，退出 server
            ...
            return
        case <-getSyncC(): // 定期同步数据
            if s.v2store.HasTTLKeys() {
                s.sync(s.Cfg.ReqTimeout())
            }
        case <-s.stop: // 停止信号，停止 server
            return
        }
    }
}

// etcdserver/raft.go.start
// raft node 启动，保持心跳
// start prepares and starts raftNode in a new goroutine. It is no longer safe
// to modify the fields after it has been started.
func (r *raftNode) start(rh *raftReadyHandler) {
}

```

## 多协议支持
如何做到监听一个端口, 开启多个协议的 server 进行处理呢？ 借助 `github.com/soheilhy/cmux` 来完成的。
```go

func (e *Etcd) serveClients() (err error) {
    if !e.cfg.ClientTLSInfo.Empty() {
        if e.cfg.logger != nil {
            e.cfg.logger.Info(
                "starting with client TLS",
                zap.String("tls-info", fmt.Sprintf("%+v", e.cfg.ClientTLSInfo)),
                zap.Strings("cipher-suites", e.cfg.CipherSuites),
            )
        } else {
            plog.Infof("ClientTLS: %s", e.cfg.ClientTLSInfo)
        }
    }

    // Start a client server goroutine for each listen address
    var h http.Handler
    if e.Config().EnableV2 {
        if len(e.Config().ExperimentalEnableV2V3) > 0 {
            srv := v2v3.NewServer(e.cfg.logger, v3client.New(e.Server), e.cfg.ExperimentalEnableV2V3)
            h = v2http.NewClientHandler(e.GetLogger(), srv, e.Server.Cfg.ReqTimeout())
        } else {
            h = v2http.NewClientHandler(e.GetLogger(), e.Server, e.Server.Cfg.ReqTimeout())
        }
    } else {
        mux := http.NewServeMux()
        etcdhttp.HandleBasic(mux, e.Server)
        h = mux
    }

    gopts := []grpc.ServerOption{}
    if e.cfg.GRPCKeepAliveMinTime > time.Duration(0) {
        gopts = append(gopts, grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
            MinTime:             e.cfg.GRPCKeepAliveMinTime,
            PermitWithoutStream: false,
        }))
    }
    if e.cfg.GRPCKeepAliveInterval > time.Duration(0) &&
        e.cfg.GRPCKeepAliveTimeout > time.Duration(0) {
        gopts = append(gopts, grpc.KeepaliveParams(keepalive.ServerParameters{
            Time:    e.cfg.GRPCKeepAliveInterval,
            Timeout: e.cfg.GRPCKeepAliveTimeout,
        }))
    }
    // start client servers in each goroutine
    for _, sctx := range e.sctxs {
        go func(s *serveCtx) { // 这里正式启动多个 client server， 包括不同的协议
            e.errHandler(s.serve(e.Server, &e.cfg.ClientTLSInfo, h, e.errHandler, gopts...))
        }(sctx)
    }
    return nil
}

// 上面函数中 s.serve 函数的实现
// serve accepts incoming connections on the listener l,
// creating a new service goroutine for each. The service goroutines
// read requests and then call handler to reply to them.
// 这里使用了 github.com/soheilhy/cmux 来实现链接复用器
// 同一个链接可以根据其协议分发到不同的 server 协议处理
// 支持 grpc, ssh, http, https等
// 同一个链接依次只能是一个协议
// 根据协议头的前几个字段来判断协议
func (sctx *serveCtx) serve(
    s *etcdserver.EtcdServer,
    tlsinfo *transport.TLSInfo,
    handler http.Handler,
    errHandler func(error),
    gopts ...grpc.ServerOption) (err error) {
    logger := defaultLog.New(ioutil.Discard, "etcdhttp", 0)
    <-s.ReadyNotify()

    if sctx.lg == nil {
        plog.Info("ready to serve client requests")
    }

    m := cmux.New(sctx.l)
    v3c := v3client.New(s)
    servElection := v3election.NewElectionServer(v3c)
    servLock := v3lock.NewLockServer(v3c)

    var gs *grpc.Server
    defer func() {
        if err != nil && gs != nil {
            gs.Stop()
        }
    }()
    // http 请求
    if sctx.insecure {
        gs = v3rpc.Server(s, nil, gopts...)
        v3electionpb.RegisterElectionServer(gs, servElection)
        v3lockpb.RegisterLockServer(gs, servLock)
        if sctx.serviceRegister != nil {
            sctx.serviceRegister(gs)
        }
        // 匹配 HTTP2 协议
        grpcl := m.Match(cmux.HTTP2())
        // 启动 grpc server
        go func() { errHandler(gs.Serve(grpcl)) }()

        var gwmux *gw.ServeMux
        if s.Cfg.EnableGRPCGateway {
            gwmux, err = sctx.registerGateway([]grpc.DialOption{grpc.WithInsecure()})
            if err != nil {
                return err
            }
        }

        httpmux := sctx.createMux(gwmux, handler)

        srvhttp := &http.Server{
            Handler:  createAccessController(sctx.lg, s, httpmux),
            ErrorLog: logger, // do not log user error
        }
        // 匹配 HTTP1 协议
        httpl := m.Match(cmux.HTTP1())
        // 启动 client HTTP Server
        go func() { errHandler(srvhttp.Serve(httpl)) }()
        sctx.serversC <- &servers{grpc: gs, http: srvhttp}
        ...
    }
    // https 请求
    if sctx.secure {
    ...
    }
    close(sctx.serversC)
    return m.Serve()
}
```
