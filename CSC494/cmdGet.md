1. `RegionRequestSender.sendReqToRegion `

- File: `client-go/internal/locate/region_request.go`
- Log:

```
[req.Type=Get] [req="region_id:10011 region_epoch:<conf_ver:1 version:71 > peer:<id:10012 store_id:1 > max_execution_duration_ms:30000 task_id:51713 resource_group_tag:\"\\n \\354\\021\\366\\252\\363\\307\\n\\331d\\347\\\\\\250\\225\\207/\\343\\337\\313\\357\\222\\323\\344\\304;\\340\\200'\\243v\\272\\200%\\022 \\371\\222a\\344\\033j\\276D@\\347R0\\357\\346\\000\\010\\327C&\\004K\\362\\023p\\304\\005R(\\251\\036\\3430\\030\\001 \\205\\001\" request_source:\"leader_external_Select\" busy_threshold_ms:1000 resource_control_context:<resource_group_name:\"default\" > cluster_id:7508966257383359189 "]
```

- Description: This log shows the GET request being sent to a specific TiKV region with the following key components:
  - `req.Type=Get`: The request type is a GET operation
  - `region_id:10011`: Target region ID where the key is located
  - `region_epoch:<conf_ver:1 version:71>`: Region metadata version for consistency checks
  - `peer:<id:10012 store_id:1>`: The specific TiKV store and peer handling this region
  - `max_execution_duration_ms:30000`: 30-second timeout for the request
  - `task_id:51713`: Unique identifier for tracking this specific request
  - `resource_group_tag`: Binary-encoded resource group information for resource control
  - `request_source:"leader_external_Select"`: Indicates this is an external SELECT query processed by the leader
  - `busy_threshold_ms:1000`: Threshold for considering the store "busy" (1 second)
  - `resource_control_context:<resource_group_name:"default">`: Resource group configuration
  - `cluster_id:7508966257383359189`: Unique cluster identifier

2. `SendRequest`

- File: `client_collapase.go` --> `client_interceptor.go` --> `client.go`

3. `sendBatchRequest`

- File: `client-go/internal/client/client_batch.go`
- Description: the request is queued to be sent via the batchConn.sendLoop, which handles batching requests over the gRPC stream.

  ```go
   select {
  case batchConn.batchCommandsCh <- entry:

   select {
  case res, ok := <-entry.res:
  ```

4. `batchConn.sendLoop`

- File: `client-go/internal/client/client_batch.go`
- Description: This function is part of the batching mechanism in the TiKV client. It continuously listens for requests to be sent and processes them in batches.

5. `FromBatchCommandsResponse`

- File: `client-go/internal/client/client_batch.go`
- Logs:

  ```
  ["FROM BATCH COMMANDS RESPONSE"] [res="Get:<value:\"\\200\\000\\001\\000\\000\\000\\002\\005\\000Alice\" exec_details_v2:<time_detail:<total_rpc_wall_time_ns:484335 > scan_detail_v2:<processed_versions:1 processed_versions_size:41 total_versions:1 rocksdb_block_cache_hit_count:3 get_snapshot_nanos:202485 read_index_propose_wait_nanos:52875 read_index_confirm_wait_nanos:12406 read_pool_schedule_wait_nanos:75443 > time_detail_v2:<wait_wall_time_ns:313449 process_wall_time_ns:95937 kv_read_wall_time_ns:416801 total_rpc_wall_time_ns:484335 > > > "]
  ```

- Description: This log shows the response received from TiKV after processing the GET request. The response contains both the actual data and detailed execution metrics:

  **Retrieved Value:**

  - `Get:<value:\"\\200\\000\\001\\000\\000\\000\\002\\005\\000Alice\">`: The actual value retrieved from TiKV, where "Alice" is the stored data with TiDB's internal encoding prefix

  **Execution Details (`exec_details_v2`):**

  - `total_rpc_wall_time_ns:484335`: Total RPC wall time (484.335 microseconds)

  **Scan Details (`scan_detail_v2`):**

  - `processed_versions:1`: Number of MVCC versions processed (1 version)
  - `processed_versions_size:41`: Total size of processed versions (41 bytes)
  - `total_versions:1`: Total MVCC versions encountered
  - `rocksdb_block_cache_hit_count:3`: Number of RocksDB block cache hits (3)
  - `get_snapshot_nanos:202485`: Time to get snapshot (202.485 microseconds)
  - `read_index_propose_wait_nanos:52875`: Raft read index propose wait time (52.875 microseconds)
  - `read_index_confirm_wait_nanos:12406`: Raft read index confirm wait time (12.406 microseconds)
  - `read_pool_schedule_wait_nanos:75443`: Read pool scheduling wait time (75.443 microseconds)

  **Time Details (`time_detail_v2`):**

  - `wait_wall_time_ns:313449`: Total wait time (313.449 microseconds)
  - `process_wall_time_ns:95937`: Processing time (95.937 microseconds)
  - `kv_read_wall_time_ns:416801`: KV read operation time (416.801 microseconds)
  - `total_rpc_wall_time_ns:484335`: Total RPC time (484.335 microseconds, matching the outer metric)

6. Final Resp in `snapshot.go`
   ```
   resp="{\"Resp\":{\"value\":\"gAABAAAAAgUAQWxpY2U=\",\"exec_details_v2\":{\"time_detail\":{\"total_rpc_wall_time_ns\":484335},\"scan_detail_v2\":{\"processed_versions\":1,\"processed_versions_size\":41,\"total_versions\":1,\"rocksdb_block_cache_hit_count\":3,\"get_snapshot_nanos\":202485,\"read_index_propose_wait_nanos\":52875,\"read_index_confirm_wait_nanos\":12406,\"read_pool_schedule_wait_nanos\":75443},\"time_detail_v2\":{\"wait_wall_time_ns\":313449,\"process_wall_time_ns\":95937,\"kv_read_wall_time_ns\":416801,\"total_rpc_wall_time_ns\":484335}}}}"] [val="gAABAAAAAgUAQWxpY2U="]
   ```

- Description: This log shows the final response being prepared for the client in `snapshot.go`. The raw TiKV response has been transformed into a structured JSON format that will be returned to the application:

  **Response Transformation:**

  - **Format**: The protobuf response from TiKV has been converted to JSON format
  - **Value Encoding**: The raw binary value `"\\200\\000\\001\\000\\000\\000\\002\\005\\000Alice"` has been base64 encoded as `"gAABAAAAAgUAQWxpY2U="`
  - **Structure**: Wrapped in a `"Resp"` object containing both the value and execution details

  **Key Components:**

  - `val="gAABAAAAAgUAQWxpY2U="`: The base64-encoded value that will be decoded by the client to retrieve "Alice"
  - **Execution Details**: All timing and performance metrics from TiKV are preserved in JSON format:
    - Same timing metrics as the batch response (484.335μs total RPC time)
    - Same scan details (1 MVCC version, 3 cache hits, etc.)
    - Same detailed timing breakdown for wait/process/read operations

  **Purpose**: This represents the final step before the response is sent back to the client application, providing both the requested data and comprehensive performance metrics for observability and debugging.

```
userReq -> batchConn.batchCommandsCh <- entry
                      ↓
        batchSendLoop fetches many entries
                      ↓
        Builds BatchCommandsRequest
                      ↓
        getClientAndSend() sends to TiKV
                      ↓
        recvLoop demuxes response to entry.res
```
