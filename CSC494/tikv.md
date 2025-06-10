## Overall workflow

1.  `Server.new<L: LockManager, F: KvFormat>`

    - File: `src/server/server.rs`
    - Description: The TiKV server, it hosts various internal components, including gRPC, the raftstore router and a snapshot worker.

2.  `builder_factory.create_builder(&self, env: Arc<Environment>)`

    - File: `src/server/server.rs`
    - Description: It registers several services to the server.(including tikv)

3.  `create_tikv<S: Tikv + Send + Clone + 'static>(s: S)`

    - File: `target/debug/build/kvproto-25d2ecb1d8748d50/out/protos/tikvpb_grpc.rs`
    - Description: It add all the handlers to the server. Tikv is a trait that defines all the methods that the server needs to implement.
    - Example:
      ```rust
        builder = builder.add_unary_handler(&METHOD_TIKV_KV_GET, move |ctx, req, resp| {
        instance.kv_get(ctx, req, resp)
        });
      ```
      ```rust
        const METHOD_TIKV_KV_GET: ::grpcio::Method<super::kvrpcpb::GetRequest, super::kvrpcpb::GetResponse> = ::grpcio::Method {
            ty: ::grpcio::MethodType::Unary,
            name: "/tikvpb.Tikv/KvGet",
            req_mar: ::grpcio::Marshaller { ser: ::grpcio::pb_ser, de: ::grpcio::pb_de },
            resp_mar: ::grpcio::Marshaller { ser: ::grpcio::pb_ser, de: ::grpcio::pb_de },
        };
      ```

4.  `Tikv.kv_get(&mut self, ctx: ::grpcio::RpcContext, _req: super::kvrpcpb::GetRequest, sink: ::grpcio::UnarySink<super::kvrpcpb::GetResponse>)`

    - File: `target/debug/build/kvproto-25d2ecb1d8748d50/out/protos/tikvpb_grpc.rs`
    - Description: It handles the `kv_get` request.

5.  `impl<E: Engine, L: LockManager, F: KvFormat> Tikv for Service<E, L, F> `

    - File: `src/server/service/kv.rs`
    - Description: It implements the `Tikv` trait for the `Service` struct.
    - Code:
      ```rust
        handle_request!(kv_get, future_get, GetRequest, GetResponse, has_time_detail);
      ```

6.  `macro_rules! handle_request`

    - File: `src/server/service/kv.rs`
    - Description: It handles the request and response.
    - Code:

      ```rust
        // 7. Call the future‐producing function (passed in as `$future_name`)
        let resp = $future_name(&self.storage, req);

        // 8. Build an async‐task that awaits the future, then sends the response
        let task = async move {
                    let resp = resp.await?;
                    let elapsed = begin_instant.saturating_elapsed();
                    set_total_time!(resp, elapsed, $time_detail);
                    sink.success(resp).await?;
                    GRPC_MSG_HISTOGRAM_STATIC
                        .$fn_name
                        .get(resource_group_priority)
                        .observe(elapsed.as_secs_f64());
                    record_request_source_metrics(source, elapsed);
                    ServerResult::Ok(())
                }

      ```

7.  `future_get<E: Engine, L: LockManager, F: KvFormat>(
    storage: &Storage<E, L, F>,
    mut req: GetRequest,
)`

    - File: `src/server/service/kv.rs`
    - Description: It handles the `kv_get` request.
    - Code:
      ```rust
        fn future_get<E: Engine, L: LockManager, F: KvFormat>(
            storage: &Storage<E, L, F>,
            mut req: GetRequest,
        ) -> impl Future<Output = ServerResult<GetResponse>> {
            let tracker = GLOBAL_TRACKERS.insert(Tracker::new(RequestInfo::new(
                req.get_context(),
                RequestType::KvGet,
                req.get_version(),
            )));
            set_tls_tracker_token(tracker);
            let start = Instant::now();
            let v = storage.get(
                req.take_context(),
                Key::from_raw(req.get_key()),
                req.get_version().into(),
            );

            async move {
                let v = v.await;
                let duration = start.saturating_elapsed();
                let mut resp = GetResponse::default();
                if let Some(err) = extract_region_error(&v) {
                    resp.set_region_error(err);
                } else {
                    match v {
                        Ok((val, stats)) => {
                            let exec_detail_v2 = resp.mut_exec_details_v2();
                            stats
                                .stats
                                .write_scan_detail(exec_detail_v2.mut_scan_detail_v2());
                            GLOBAL_TRACKERS.with_tracker(tracker, |tracker| {
                                tracker.write_scan_detail(exec_detail_v2.mut_scan_detail_v2());
                                tracker.merge_time_detail(exec_detail_v2.mut_time_detail_v2());
                            });
                            set_time_detail(exec_detail_v2, duration, &stats.latency_stats);
                            match val {
                                Some(val) => resp.set_value(val),
                                None => resp.set_not_found(true),
                            }
                        }
                        Err(e) => resp.set_error(extract_key_error(&e)),
                    }
                }
                GLOBAL_TRACKERS.remove(tracker);
                Ok(resp)
            }
        }
        
      ```

8.  `storage.get(&self, ctx: &mut Context, req: GetRequest) -> Result<GetResponse>`

    - File: `src/storage/mod.rs`
    - Description: It handles the `kv_get` request.

