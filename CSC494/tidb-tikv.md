## Main communication between TiDB and TiKV

1.  `distsql.Select(ctx, sctx, kvReq, fieldTypes, builder)`

    - File: `distsql/distsql.go`
    - Description: Dispatches DAG request to TiKV via gRPC

2.  `store.tikv.CopClient.Send(ctx, req)`

    - File: `store/tikv/coprocessor.go`
    - Description: Sends request to TiKV coprocessor

3.  `copIterator.open(ctx, tryCopLiteWorker)`

    - File: `coprocessor/coprocessor.go`
    - Description: Opens coprocessor iterator

4.  `copIteratorWorker.run(ctx)`

    - File: `store/copr/coprocessor.go`
    - Description: Worker loop that processes coprocessor tasks

5.  `copIteratorWorker.handleTask(ctx, task, respCh) → handleTaskOnce(ctx, task)`

    - File: `store/copr/coprocessor.go`
    - Description: Handles region-level coprocessor task and sends response

6.  `RegionRequestSender.SendReqCtx(bo, req, regionID, timeout, et, opts…)`

    - File: `store/tikv/region_request.go`
    - Description: Top-level send with retry mechanism

7.  `sendReqState.next(bo, req, regionID, timeout, et, opts…)`

    - File: `store/tikv/region_request.go`
    - Description: Single retry iteration with region metadata handling

8.  `sendReqState.send(bo, req, timeout)`

    - File: `store/tikv/region_request.go`
    - Description: Invokes RPC client to transmit request

9.  `RPCClient.sendRequest(ctx, addr, req, timeout)`

    - File: `github.com/tikv/client-go/v2/internal/client/client.go`
    - Description: Handles gRPC connection and makes network call to TiKV/TiFlash

```go
// from tikv/client-go/internal/client/client.go/sendRequest
	switch req.Type {
	case tikvrpc.CmdBatchCop:
		return wrapErrConn(c.getBatchCopStreamResponse(ctx, client, req, timeout, connArray))
	case tikvrpc.CmdCopStream:
		return wrapErrConn(c.getCopStreamResponse(ctx, client, req, timeout, connArray))
	case tikvrpc.CmdMPPConn:
		return wrapErrConn(c.getMPPStreamResponse(ctx, client, req, timeout, connArray))
	}
```

## tikv_driver.go

This file is the main file for the TiKV driver. It contains two main structs: `TiKVDriver` and `tikvStore`.

1. TiKVDriver (Initialized in `OpenWithOptions`)

   - It initialzies a new rpcClient
     ```go
     // from tikv_driver.go
     rpcClient := tikv.NewRPCClient(tikv.WithSecurity(d.security), tikv.WithCodec(codec))
     ```
   - In the `NewRPCClient` function, it initialzies a new `RPCClient` struct, with field `conns` as a map of `string` to `*connArray`

     ```go
     // from tikv_driver.go
     &RPCClient{
        conns: make(map[string]*connArray),
        ...
     }
     ```

   - `conns` is created in `RPCClient.createConnArray`, and then `newConnArray` is called, array is initialized in `connArray.Init`

     ```go
     // from tikv/client-go/internal/client/client.go/Init
     for i := range a.v {
        ...
     conn, err := connArray.monitoredDial(
     	ctx,
     	fmt.Sprintf("%s-%d", a.target, i),
     	addr,
     	opts...,
     )
        ...
     a.v[i] = conn
     ```

     Then `grpc.DialContext` is called inside `connArray.monitoredDial` to create a new connection to the TiKV server.

     ```go
     // from tikv/client-go/internal/client/client.go/monitoredDial
     func (a *connArray) monitoredDial(ctx context.Context, connName, target string, opts ...grpc.DialOption) (conn *monitoredConn, err error) {
        conn = &monitoredConn{
            Name: connName,
        }
        conn.ClientConn, err = grpc.DialContext(ctx, target, opts...)
        if err != nil {
            return nil, err
        }
        a.monitor.AddConn(conn)
        return conn, nil
     }
     ```

2. tikvStore
   - Then a new `tikvStore` is created in `openWithOptions`
   ```go
   // from tikv_driver.go
   s, err = tikv.NewKVStore(uuid, pdClient, spkv, &injectTraceClient{Client: rpcClient}
   ```
   - `tikvclient` is used in the initialization
   ```go
   store.clientMu.client = client.NewReqCollapse(client.NewInterceptedClient(tikvclient))
   ```

## Overall workflow

1.  `clientConn.handleQuery()`

    - File: `server/conn.go`
    - Description: Entry point for SQL queries received over the MySQL protocol. Parses SQL text and begins processing.

2.  `cc.ctx.Parse(ctx, sql)`

    - File: `session/session.go`
    - Description: Parses SQL text into an Abstract Syntax Tree (AST) using TiDB's built-in parser.

3.  `cc.ctx.ExecuteStmt(ctx, stmt)` → `tc.Session.ExecuteStmt(ctx, stmt)`

    - File: `session/session.go`
    - Description: Executes the parsed statement. Manages transaction context and initiates compilation and execution.

4.  `session.ExecuteStmt(ctx, stmt)`

    - File: `session/session.go`
    - Description: Central entry to compile and execute SQL statements, handles session-scoped transaction and state.

5.  `executor.Compile(ctx, stmtNode)`

    - File: `executor/compiler.go`
    - Description: Compiles the AST into a logical and physical plan, preparing it for execution.

6.  `plannercore.Optimize(ctx, sctx, stmtNode, infoSchema)`

    - File: `planner/core/optimizer.go`
    - Description: Performs logical and physical plan optimization (e.g., predicate pushdown, join reordering).

7.  `executorBuilder.build(ctx, plan)`

    - File: `executor/builder.go`
    - Description: Converts the physical plan into an executor tree (e.g., `TableReader`, `IndexLookUpReader`).

8.  `ConstructDAGReq(ctx, plans, storeType)`

    - File: `executor/internal/builder/builder_utils.go`
    - Description: Constructs the `tipb.DAGRequest` protobuf message from the executor plan to be sent to TiKV.

9.  `distsql.Select(ctx, sctx, kvReq, fieldTypes, builder)`

    - File: `distsql/distsql.go`
    - Description: Dispatches the DAG request to TiKV via gRPC. Handles region splitting and concurrency.

10. `store.tikv.CopClient.Send(ctx, req)`

    - File: `store/tikv/coprocessor.go`
    - Description: Sends the request to TiKV coprocessor, handles retries, region errors, and low-level gRPC communication.

11. `executor.Next(ctx)`
    - File: `executor/executor.go`
    - Description: Pulls results from TiKV through the executor. Applies final operations like aggregation or limit, and returns rows to the client.
   
```
const (
	CmdGet CmdType = 1 + iota
	CmdScan
	CmdPrewrite
	CmdCommit
	CmdCleanup
	CmdBatchGet
	CmdBatchRollback
	CmdScanLock
	CmdResolveLock
	CmdGC
	CmdDeleteRange
	CmdPessimisticLock
	CmdPessimisticRollback
	CmdTxnHeartBeat
	CmdCheckTxnStatus
	CmdCheckSecondaryLocks
	CmdFlashbackToVersion
	CmdPrepareFlashbackToVersion
	CmdFlush
	CmdBufferBatchGet

	CmdRawGet CmdType = 256 + iota
	CmdRawBatchGet
	CmdRawPut
	CmdRawBatchPut
	CmdRawDelete
	CmdRawBatchDelete
	CmdRawDeleteRange
	CmdRawScan
	CmdRawGetKeyTTL
	CmdRawCompareAndSwap
	CmdRawChecksum

	CmdUnsafeDestroyRange

	CmdRegisterLockObserver
	CmdCheckLockObserver
	CmdRemoveLockObserver
	CmdPhysicalScanLock

	CmdStoreSafeTS
	CmdLockWaitInfo

	CmdGetHealthFeedback
	CmdBroadcastTxnStatus

	CmdCop CmdType = 512 + iota
	CmdCopStream
	CmdBatchCop
	CmdMPPTask   // TODO: These non TiKV RPCs should be moved out of TiKV client
	CmdMPPConn   // TODO: These non TiKV RPCs should be moved out of TiKV client
	CmdMPPCancel // TODO: These non TiKV RPCs should be moved out of TiKV client
	CmdMPPAlive  // TODO: These non TiKV RPCs should be moved out of TiKV client

	CmdMvccGetByKey CmdType = 1024 + iota
	CmdMvccGetByStartTs
	CmdSplitRegion

	CmdDebugGetRegionProperties CmdType = 2048 + iota
	CmdCompact                          // TODO: These non TiKV RPCs should be moved out of TiKV client
	CmdGetTiFlashSystemTable            // TODO: These non TiKV RPCs should be moved out of TiKV client

	CmdEmpty CmdType = 3072 + iota
)
```
