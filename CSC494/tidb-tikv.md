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

10. `RPCClient.getCopStreamResponse(ctx, client, req, timeout, connArray)`

    - File: `github.com/tikv/client-go/v2/internal/client/client.go`
    - Description: Handles coprocessor request

11. `tikvrpc.CallRPC(ctx1, client, req)`

    - File: `github.com/tikv/client-go/v2/internal/tikvrpc/tikvrpc.go`
    - Description: Handles RPC call to TiKV

```go
switch req.Type {
	case CmdGet:
		resp.Resp, err = client.KvGet(ctx, req.Get())
	case CmdScan:
		resp.Resp, err = client.KvScan(ctx, req.Scan())
	case CmdPrewrite:
		resp.Resp, err = client.KvPrewrite(ctx, req.Prewrite())
	case CmdPessimisticLock:
		resp.Resp, err = client.KvPessimisticLock(ctx, req.PessimisticLock())
	case CmdPessimisticRollback:
		resp.Resp, err = client.KVPessimisticRollback(ctx, req.PessimisticRollback())
	case CmdCommit:
		resp.Resp, err = client.KvCommit(ctx, req.Commit())
	case CmdCleanup:
		resp.Resp, err = client.KvCleanup(ctx, req.Cleanup())
	case CmdBatchGet:
		resp.Resp, err = client.KvBatchGet(ctx, req.BatchGet())
	case CmdBatchRollback:
		resp.Resp, err = client.KvBatchRollback(ctx, req.BatchRollback())
	case CmdScanLock:
		resp.Resp, err = client.KvScanLock(ctx, req.ScanLock())
	case CmdResolveLock:
		resp.Resp, err = client.KvResolveLock(ctx, req.ResolveLock())
	case CmdGC:
		resp.Resp, err = client.KvGC(ctx, req.GC())
	case CmdDeleteRange:
		resp.Resp, err = client.KvDeleteRange(ctx, req.DeleteRange())
	case CmdRawGet:
		resp.Resp, err = client.RawGet(ctx, req.RawGet())
	case CmdRawBatchGet:
		resp.Resp, err = client.RawBatchGet(ctx, req.RawBatchGet())
	case CmdRawPut:
		resp.Resp, err = client.RawPut(ctx, req.RawPut())
	case CmdRawBatchPut:
		resp.Resp, err = client.RawBatchPut(ctx, req.RawBatchPut())
	case CmdRawDelete:
		resp.Resp, err = client.RawDelete(ctx, req.RawDelete())
	case CmdRawBatchDelete:
		resp.Resp, err = client.RawBatchDelete(ctx, req.RawBatchDelete())
	case CmdRawDeleteRange:
		resp.Resp, err = client.RawDeleteRange(ctx, req.RawDeleteRange())
	case CmdRawScan:
		resp.Resp, err = client.RawScan(ctx, req.RawScan())
	case CmdUnsafeDestroyRange:
		resp.Resp, err = client.UnsafeDestroyRange(ctx, req.UnsafeDestroyRange())
	case CmdGetKeyTTL:
		resp.Resp, err = client.RawGetKeyTTL(ctx, req.RawGetKeyTTL())
	case CmdRawCompareAndSwap:
		resp.Resp, err = client.RawCompareAndSwap(ctx, req.RawCompareAndSwap())
	case CmdRawChecksum:
		resp.Resp, err = client.RawChecksum(ctx, req.RawChecksum())
	case CmdRegisterLockObserver:
		resp.Resp, err = client.RegisterLockObserver(ctx, req.RegisterLockObserver())
	case CmdCheckLockObserver:
		resp.Resp, err = client.CheckLockObserver(ctx, req.CheckLockObserver())
	case CmdRemoveLockObserver:
		resp.Resp, err = client.RemoveLockObserver(ctx, req.RemoveLockObserver())
	case CmdPhysicalScanLock:
		resp.Resp, err = client.PhysicalScanLock(ctx, req.PhysicalScanLock())
	case CmdCop:
		resp.Resp, err = client.Coprocessor(ctx, req.Cop())
	case CmdMPPTask:
		resp.Resp, err = client.DispatchMPPTask(ctx, req.DispatchMPPTask())
	case CmdMPPAlive:
		resp.Resp, err = client.IsAlive(ctx, req.IsMPPAlive())
	case CmdMPPConn:
		var streamClient tikvpb.Tikv_EstablishMPPConnectionClient
		streamClient, err = client.EstablishMPPConnection(ctx, req.EstablishMPPConn())
		resp.Resp = &MPPStreamResponse{
			Tikv_EstablishMPPConnectionClient: streamClient,
		}
	case CmdMPPCancel:
		// it cannot use the ctx with cancel(), otherwise this cmd will fail.
		resp.Resp, err = client.CancelMPPTask(ctx, req.CancelMPPTask())
	case CmdCopStream:
		var streamClient tikvpb.Tikv_CoprocessorStreamClient
		streamClient, err = client.CoprocessorStream(ctx, req.Cop())
		resp.Resp = &CopStreamResponse{
			Tikv_CoprocessorStreamClient: streamClient,
		}
	case CmdBatchCop:
		var streamClient tikvpb.Tikv_BatchCoprocessorClient
		streamClient, err = client.BatchCoprocessor(ctx, req.BatchCop())
		resp.Resp = &BatchCopStreamResponse{
			Tikv_BatchCoprocessorClient: streamClient,
		}
	case CmdMvccGetByKey:
		resp.Resp, err = client.MvccGetByKey(ctx, req.MvccGetByKey())
	case CmdMvccGetByStartTs:
		resp.Resp, err = client.MvccGetByStartTs(ctx, req.MvccGetByStartTs())
	case CmdSplitRegion:
		resp.Resp, err = client.SplitRegion(ctx, req.SplitRegion())
	case CmdEmpty:
		resp.Resp, err = &tikvpb.BatchCommandsEmptyResponse{}, nil
	case CmdCheckTxnStatus:
		resp.Resp, err = client.KvCheckTxnStatus(ctx, req.CheckTxnStatus())
	case CmdCheckSecondaryLocks:
		resp.Resp, err = client.KvCheckSecondaryLocks(ctx, req.CheckSecondaryLocks())
	case CmdTxnHeartBeat:
		resp.Resp, err = client.KvTxnHeartBeat(ctx, req.TxnHeartBeat())
	case CmdStoreSafeTS:
		resp.Resp, err = client.GetStoreSafeTS(ctx, req.StoreSafeTS())
	case CmdLockWaitInfo:
		resp.Resp, err = client.GetLockWaitInfo(ctx, req.LockWaitInfo())
	case CmdCompact:
		resp.Resp, err = client.Compact(ctx, req.Compact())
	case CmdFlashbackToVersion:
		resp.Resp, err = client.KvFlashbackToVersion(ctx, req.FlashbackToVersion())
	case CmdPrepareFlashbackToVersion:
		resp.Resp, err = client.KvPrepareFlashbackToVersion(ctx, req.PrepareFlashbackToVersion())
	case CmdGetTiFlashSystemTable:
		resp.Resp, err = client.GetTiFlashSystemTable(ctx, req.GetTiFlashSystemTable())
	case CmdFlush:
		resp.Resp, err = client.KvFlush(ctx, req.Flush())
	case CmdBufferBatchGet:
		resp.Resp, err = client.KvBufferBatchGet(ctx, req.BufferBatchGet())
	case CmdGetHealthFeedback:
		resp.Resp, err = client.GetHealthFeedback(ctx, req.GetHealthFeedback())
	case CmdBroadcastTxnStatus:
		resp.Resp, err = client.BroadcastTxnStatus(ctx, req.BroadcastTxnStatus())
	default:
		return nil, errors.Errorf("invalid request type: %v", req.Type)
	}
```

## tikv_driver initialization

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

## TvGet Trace (low->high)

1. `RPCClient.sendRequest(ctx, addr, req, timeout)`

   - File: `github.com/tikv/client-go/v2/internal/client/client.go`
   - Description: Handles gRPC connection and makes network call to TiKV/TiFlash

2. `RegionRequestSender.sendReqToRegion(bo, rpcCtx, req, timeout)`

   - File: `store/tikv/client-go/v2/internal/locate/region_request.go`
   - Description: Sends request to the correct region
   - Note: req is a `tikvrpc.Request` struct

3. `RegionRequestSender.SendReqCtx(bo, req, regionID, timeout, et, opts…)`

   - File: `store/tikv/client.go/v2/internal/locate/region_request.go`
   - Description: SendReqCtx sends a request to tikv server and return response and RPCCtx of this RPC.

4. `ClientHelper.SendReqCtx(bo, req, regionID, timeout, et, directStoreAddr, opts…)`

   - File: `store/tikv/client-go/v2/txnkv/txnsnapshot/client_helper.go`
   - Description: SendReqCtx wraps the SendReqCtx function and use the resolved lock result in the kvrpcpb.Context.

5. `KVSnapshot.get(ctx, bo, k)`

   - File: `store/tikv/client-go/v2/txnkv/txnsnapshot/snapshot.go`
   - Description: Get gets the value for the given key.

6. `KVSnapshot.get(ctx, bo, k)`

   - File: `store/tikv/client-go/v2/txnkv/txnsnapshot/snapshot.go`
   - Description: Get gets the value for the given key.

   ```go
     req := tikvrpc.NewReplicaReadRequest(tikvrpc.CmdGet,
     	&kvrpcpb.GetRequest{
     		Key:     k,
     		Version: s.version,
     	}, s.mu.replicaRead, &s.replicaReadSeed, kvrpcpb.Context{
     		Priority:         s.priority.ToPB(),
     		NotFillCache:     s.notFillCache,
     		TaskId:           s.mu.taskID,
     		ResourceGroupTag: s.mu.resourceGroupTag,
     		IsolationLevel:   s.isolationLevel.ToPB(),
     		ResourceControlContext: &kvrpcpb.ResourceControlContext{
     			ResourceGroupName: s.mu.resourceGroupName,
     		},
     		BusyThresholdMs: uint32(s.mu.busyThreshold.Milliseconds()),
     	})
   ```

7. `KVSnapshot.Get(ctx, k)`

   - File: `store/tikv/client-go/v2/txnkv/txnsnapshot/snapshot.go`
   - Description:
     - Check the cached values first.
     - If not found, call `get`
     - Update the cache with the new value

8. `KVSnapshot.Get(ctx, k)`

   - File: `store/tikv/client-go/v2/txnkv/txnsnapshot/kv_snapshot.go`
   - Description: Get gets the value for the given key.

9. `PointGetExecutor.get(ctx, key)`

   - File: `executor/point_get.go`
   - Description:
     - Empty key → immediately “not found.”
     - Txn MemBuffer → your own uncommitted writes and deletes.
     - Pessimistic-lock cache → in pessimistic mode, values you locked earlier.
     - Table-point-get cache → if the table is under a read-lock and EnablePointGetCache is true, consults an in-memory table snapshot before TiKV.
     - Snapshot layer → calls KVSnapshot.Get

10. `PointGetExecutor.getAndLock(ctx, key)`


- File: `executor/point_get.go`

11. `PointGetExecutor.Next(ctx, req)`

- File: `executor/point_get.go`
- Description: Next implements the Executor interface.

12. `executorBuilder.build(plan)`

- File: `executor/builder.go`
- Description: Builds an executor from a logical plan.

```go
case *plannercore.PointGetPlan:
    return b.buildPointGet(v)
```

13. `ExecStmt.PointGet(ctx)`

- File: `executor/adapter.go`
- Description: PointGet short path for point exec directly from plan, keep only necessary steps

14. `session.ExecuteStmt(ctx, stmt)`

- File: `session/session.go`
- Description: ExecuteStmt executes a statement.

```go
// Transform abstract syntax tree to a physical plan(stored in executor.ExecStmt).
compiler := executor.Compiler{Ctx: s}
stmt, err := compiler.Compile(ctx, stmtNode)


if stmt.PsStmt != nil { // point plan short path
    recordSet, err = stmt.PointGet(ctx)
    s.txn.changeToInvalid()
} else {
    recordSet, err = runStmt(ctx, s, stmt)
}
```

15. `TiDBContext.ExecuteStmt(ctx, stmt)`

- File: `server/driver_tidb.go`
- Description: ExecuteStmt implements QueryCtx interface.

```go
if s, ok := stmt.(*ast.NonTransactionalDMLStmt); ok {
    rs, err = session.HandleNonTransactionalDML(ctx, s, tc.Session)
} else {
    rs, err = tc.Session.ExecuteStmt(ctx, stmt)
}
```

16. `clientConn.handleStmt(ctx, stmt, warns, lastStmt)`

- File: `server/conn.go`
- Description: The first return value indicates whether the call of handleStmt has no side effect and can be retried.

17. `clientConn.handleQuery(ctx, sql)`

- File: `server/conn.go`
- Description: handleQuery executes the sql query string and writes result set or result ok to the client.

```go
// Parse parses the SQL text into an Abstract Syntax Tree (AST).
stmts, err = cc.ctx.Parse(ctx, sql)
}
```

1.  `clientConn.dispatch(ctx, data)`

- File: `server/conn.go`
- Description: dispatch handles client request based on command which is the first byte of the data.

```go
case mysql.ComQuery: // Most frequently used command.
    // For issue 1989
    // Input payload may end with byte '\0', we didn't find related mysql document about it, but mysql
    // implementation accept that case. So trim the last '\0' here as if the payload an EOF string.
    // See http://dev.mysql.com/doc/internals/en/com-query.html
    if len(data) > 0 && data[len(data)-1] == 0 {
        data = data[:len(data)-1]
        dataStr = string(hack.String(data))
    }
    return cc.handleQuery(ctx, dataStr)
```

19. `clientConn.Run(ctx)`

- File: `server/conn.go`
- Description: Run reads client query and writes query result to client in for loop
=======
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

