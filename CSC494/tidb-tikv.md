### tikv_driver.go

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

   - `grpc.DialContext` is a function that creates a new gRPC client and connects to a gRPC server.
