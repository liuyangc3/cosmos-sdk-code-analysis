# RPC设计解析

## API 服务端

当从命令行启动节点时，`RunE` 调用[startInProcess](https://github.com/cosmos/cosmos-sdk/blob/v0.46.7/server/start.go#L143)
```go
RunE: func(cmd *cobra.Command, _ []string) error {
	...
	startInProcess(serverCtx, clientCtx, appCreator)
}
```

关键代码如下（有省略），https://github.com/cosmos/cosmos-sdk/blob/v0.46.7/server/start.go#L405
```go
var apiSrv *api.Server
apiSrv = api.New(clientCtx, ctx.Logger.With("module", "api-server"))
app.RegisterAPIRoutes(apiSrv, config.API)
go func() {
	if err := apiSrv.Start(config); err != nil {
		errCh <- err
	}
}()
```
整个过程分为3步:
* 创建 api http server
* 注册路由，即 grpc http handler
* 启动 http server

app.RegisterAPIRoutes 将 handler 注入 grpc http server mux，一般是在 application 侧实现, 例如 gaiad 的[实现](https://github.com/cosmos/gaia/blob/v7.1.0/app/app.go#L799) 如下
```go
// RegisterAPIRoutes registers all application module routes with the provided
// API server.
func (app *GaiaApp) RegisterAPIRoutes(apiSvr *api.Server, apiConfig config.APIConfig) {
	clientCtx := apiSvr.ClientCtx
	rpc.RegisterRoutes(clientCtx, apiSvr.Router)
	// Register legacy tx routes.
	authrest.RegisterTxRoutes(clientCtx, apiSvr.Router)
	// Register new tx routes from grpc-gateway.
	authtx.RegisterGRPCGatewayRoutes(clientCtx, apiSvr.GRPCGatewayRouter)
	// Register new tendermint queries routes from grpc-gateway.
	tmservice.RegisterGRPCGatewayRoutes(clientCtx, apiSvr.GRPCGatewayRouter)

	// Register legacy and grpc-gateway routes for all modules.
	ModuleBasics.RegisterRESTRoutes(clientCtx, apiSvr.Router)
	ModuleBasics.RegisterGRPCGatewayRoutes(clientCtx, apiSvr.GRPCGatewayRouter)

	// register swagger API from root so that other applications can override easily
	if apiConfig.Swagger {
		RegisterSwaggerAPI(apiSvr.Router)
	}
}
```

可以看到注入的 handler 有 module 的 route 例如 `auth/tx`， 还有 tmservice， 或者 Swagger，其中 
- authtx 处理 uri 为`/cosmos/tx/v1beta1/txs/*` 和 `/cosmos/tx/v1beta1/simulate` 的请求
- tmservice 处理 uri 为 `/cosmos/base/tendermint/v1beta1/*` 的请求


`Start` 负责启动 server， 其关键代码如下：
```go
// Start starts the API server. Internally, the API server leverages Tendermint's
// JSON RPC server. Configuration options are provided via config.APIConfig
// and are delegated to the Tendermint JSON RPC server. The process is
// non-blocking, so an external signal handler must be used.
func (s *Server) Start(cfg config.Config) error {
	...
	listener, err := tmrpcserver.Listen(cfg.API.Address, tmCfg)
	if err != nil {
		s.mtx.Unlock()
		return err
	}

	s.registerGRPCGatewayRoutes()
	s.listener = listener

	s.logger.Info("starting API server...")
	return tmrpcserver.Serve(s.listener, s.Router, s.logger, tmCfg)
}

func (s *Server) registerGRPCGatewayRoutes() {
	s.Router.PathPrefix("/").Handler(s.GRPCGatewayRouter)
}
```

`cfg.API.Address` 默认的是`"tcp://0.0.0.0:1317"`，另外可以看到，uri为`/`的请求都交给 `GRPCGatewayRouter` hanlder 处理。


## GRPC 服务端

TODO 

在node启动时会调用`node.startRPC()`方法来监听rpc请求，`node.startRPC()`方法同时会实现以下各种服务的注册。`startRPC()`方法会调用如下代码：
```go
func RegisterRPCFuncs(mux *http.ServeMux, funcMap map[string]*RPCFunc, cdc *amino.Codec, logger log.Logger) {
	// HTTP endpoints
	for funcName, rpcFunc := range funcMap {
		mux.HandleFunc("/"+funcName, makeHTTPHandler(rpcFunc, cdc, logger))
	}

	// JSONRPC endpoints
	mux.HandleFunc("/", handleInvalidJSONRPCPaths(makeJSONRPCHandler(funcMap, cdc, logger)))
}
```
这样rpc服务可以有两种方式进行访问，一种是jsonrpc的方法访问服务，这种方法uri都是`/`，通过接受post的json数据进行通信，tendermint代码封装的client就是用这种方法。另外一种是通过不同的uri进行访问，方便网页进行访问等。rpc服务可以同时监听两个端口，采用以下不同的两种协议：


1. 普通http协议。具体注册的方法如下：
```go
// tendermint/rpc/core/routes.go
var Routes = map[string]*rpc.RPCFunc{
	// info API
	"health":               rpc.NewRPCFunc(Health, ""),
	"status":               rpc.NewRPCFunc(Status, ""),
	"net_info":             rpc.NewRPCFunc(NetInfo, ""),
	"blockchain":           rpc.NewRPCFunc(BlockchainInfo, "minHeight,maxHeight"),
	"genesis":              rpc.NewRPCFunc(Genesis, ""),
	"block":                rpc.NewRPCFunc(Block, "height"),
	"block_results":        rpc.NewRPCFunc(BlockResults, "height"),
	"commit":               rpc.NewRPCFunc(Commit, "height"),
	"tx":                   rpc.NewRPCFunc(Tx, "hash,prove"),
	"tx_search":            rpc.NewRPCFunc(TxSearch, "query,prove,page,per_page"),
	"validators":           rpc.NewRPCFunc(Validators, "height"),
	"dump_consensus_state": rpc.NewRPCFunc(DumpConsensusState, ""),
	"consensus_state":      rpc.NewRPCFunc(ConsensusState, ""),
	"consensus_params":     rpc.NewRPCFunc(ConsensusParams, "height"),
	"unconfirmed_txs":      rpc.NewRPCFunc(UnconfirmedTxs, "limit"),
	"num_unconfirmed_txs":  rpc.NewRPCFunc(NumUnconfirmedTxs, ""),
	// broadcast API
	"broadcast_tx_commit": rpc.NewRPCFunc(BroadcastTxCommit, "tx"),
	"broadcast_tx_sync":   rpc.NewRPCFunc(BroadcastTxSync, "tx"),
	"broadcast_tx_async":  rpc.NewRPCFunc(BroadcastTxAsync, "tx"),
	// abci API
	"abci_query": rpc.NewRPCFunc(ABCIQuery, "path,data,height,prove"),
	"abci_info":  rpc.NewRPCFunc(ABCIInfo, ""),
}
```
配置文件config.toml的`unsafe`字段默认配置为false,设置true时会注册如下api：
```go
// tendermint/rpc/core/routes.go
func AddUnsafeRoutes() {
	// control API
	Routes["dial_seeds"] = rpc.NewRPCFunc(UnsafeDialSeeds, "seeds")
	Routes["dial_peers"] = rpc.NewRPCFunc(UnsafeDialPeers, "peers,persistent")
	Routes["unsafe_flush_mempool"] = rpc.NewRPCFunc(UnsafeFlushMempool, "")

	// profiler API
	Routes["unsafe_start_cpu_profiler"] = rpc.NewRPCFunc(UnsafeStartCPUProfiler, "filename")
	Routes["unsafe_stop_cpu_profiler"] = rpc.NewRPCFunc(UnsafeStopCPUProfiler, "")
	Routes["unsafe_write_heap_profile"] = rpc.NewRPCFunc(UnsafeWriteHeapProfile, "filename")
}
```

2. websocket协议。注册方法如下：
```
//tendermint/rpc/core/routes.go
    var Routes = map[string]*rpc.RPCFunc{
	// subscribe/unsubscribe are reserved for websocket events.
	"subscribe":       rpc.NewWSRPCFunc(Subscribe, "query"),
	"unsubscribe":     rpc.NewWSRPCFunc(Unsubscribe, "query"),
	"unsubscribe_all": rpc.NewWSRPCFunc(UnsubscribeAll, ""),
    }
```
    访问此服务的uri都是`/websocket`。`rpc/lib/server/handlers.go:WebsocketManager`类对进入的普通http请求进行升级为websocket协议，并处理后续的业务逻辑处理。任何`/websocket`的请求都会启动两个goroutine来处理websocket的读写问题。

* GRPC协议。默认配置是不开启此服务的。注册的服务也只有以下两个：
```
service BroadcastAPI {
  rpc Ping(RequestPing) returns (ResponsePing) ;
  rpc BroadcastTx(RequestBroadcastTx) returns (ResponseBroadcastTx) ;
}
```
实现此服务的包地址是：`tendermint/rpc/grpc`

## RPC客户端
不管客户端是读取信息还是发送交易都需要`client/context`包。`context`包封装了`tendermint/rpc/client`包的`HTTP`结构体：
```
type HTTP struct {
	remote string
	rpc    *rpcclient.JSONRPCClient
	*WSEvents
}
```

`WSEvents`主要结构如下：
```
type WSEvents struct {
	cmn.BaseService
	ws       *rpcclient.WSClient
}
```
`WSEvents`需要调用`start()`方法才能完成websocket的连接。

* `tendermint/rpc/lib/client`包
这个包主要包含三种client：
1. `JSONRPCClient` 通过`PostForm`方法把json结构的数据发送给server来进行通信。
1. `URIClient` 通过访问不同的uri来进行通信。cosmos项目中没有用到。
1. `WSClient` 用来建立websocket连接，实现sub/pub功能。
