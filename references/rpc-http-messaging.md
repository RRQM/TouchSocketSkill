# RPC, HTTP, DMTP, MQTT, MCP

## `TouchSocket.Rpc`

Use for shared RPC infrastructure. Concrete transports live in DMTP, WebApi, JsonRpc, XmlRpc, and MQTT RPC.

Key concepts:

- `IRpcClient`, `IRpcServer`, `IRemoteServer`, `ITargetRpcClient`.
- `ICallContext`, `CallContext`, call-context accessor types.
- Server lifetimes: `SingletonRpcServer`, `ScopedRpcServer`, `TransientRpcServer` and corresponding interfaces.
- Dispatchers: `ImmediateRpcDispatcher`, `QueueRpcDispatcher`, `ConcurrencyRpcDispatcher`.
- Invocation: `InvokeOption`, `InvokeResult`, `InvokeStatus`, `FeedbackType`, `RpcParameter`, `RpcMethod`.
- Registration/store: `RpcStore`, `ActionMap`, `RpcContainerExtension`, `RpcStoreExtension`.
- Proxy/code helpers: `RpcDispatchProxy<TClient,TAttribute>`, `CodeGenerator`, `ClassCodeGenerator`.

When adding a new RPC family, keep shared call semantics in `TouchSocket.Rpc`; keep protocol parsing and attributes in the protocol assembly.

## `TouchSocket.Rpc.RateLimiting`

Use for rate limiting across RPC families. It depends only on `TouchSocket.Rpc` plus `System.Threading.RateLimiting`.

Apply it when the request mentions throttling, overload protection, per-method limits, or limiting DmtpRpc/WebApi/JsonRpc/XmlRpc calls.

## `TouchSocket.Dmtp`

DMTP is the duplex binary protocol layer. Use it for bidirectional actor messaging, RPC, routing, channels, verification, file transfer, and Redis-like cache features.

Key concepts:

- Components: `TcpDmtpClient`, `TcpDmtpService`, `TcpDmtpSessionClient`, `UdpDmtp`, `HttpDmtpClient`, `HttpDmtpService`, `WebSocketDmtpClient`.
- Actor abstractions: `IDmtpActor`, `IDmtpActorObject`, `IDmtpClient`, `IDmtpService`, feature actors such as `IDmtpRpcActor` and Redis actor types.
- Options: `DmtpOption`, `DmtpFeatureOption`, `DmtpRpcOption`, Redis option types.
- Routing packages: `RouterPackage`, `MsgRouterPackage`, `WaitRouterPackage`, `RouteType`, `IDmtpRouteService`.
- Channels: `IDmtpChannel`, `ChannelStatus`, channel extension methods.
- Plugins: `IDmtpConnectingPlugin`, `IDmtpConnectedPlugin`, `IDmtpReceivedPlugin`, `IDmtpRoutingPlugin`, `IDmtpCreatedChannelPlugin`, `IDmtpClosingPlugin`, `IDmtpClosedPlugin`.
- RPC: `DmtpRpcAttribute`, `DmtpRpcActor`, `DmtpRpcFeature`, `DmtpInvokeOption`, serialization selector types.
- Protocol security and verification: `VerifyToken` on `DmtpOption`, wait types such as `WaitVerify`.

Common setup from samples:

```csharp
var service = new TcpDmtpService();
await service.SetupAsync(new TouchSocketConfig()
    .SetListenIPHosts(7789)
    .SetDmtpOption(option =>
    {
        option.VerifyToken = "Dmtp";
    })
    .ConfigureContainer(container => container.AddConsoleLogger())
    .ConfigurePlugins(pluginManager =>
    {
        pluginManager.UseDmtpRpc();
    }));
await service.StartAsync();
```

Always check whether a feature is open-source DMTP or professional DMTP. Relay, remote access, remote stream, event bus, message queue, named-pipe DMTP, and multithread file transfer are in `TouchSocketPro.Dmtp`.

## RPC reliability: scheduling, feedback, serialization, timeout

Applies to DmtpRpc and other RPC families. Confirm every behavior against the locked package version's source, samples, and tests; do not promote single-version observations to all versions.

### Dispatcher and `[Reenterable(false)]`

- `[Reenterable(false)]` controls reentrancy per method/service, not a performance switch. Whether it takes effect on interface vs. implementation depends on version and registration; verify on the target version.
- `ImmediateRpcDispatcher`: runs on the receive thread; keep handlers short. Blocking or long work delays later messages.
- `QueueRpcDispatcher`: queued scheduling; confirm queue scope, ordering, and capacity in the target version.
- `ConcurrencyRpcDispatcher`: concurrent scheduling; handlers and shared state must be thread-safe.
- Configure the dispatcher explicitly; do not rely on assumed defaults. Avoid sync blocking, long CPU work, or waiting for a reverse call that needs later messages on the same receive path. Guard shared state with bounded queues, locks, or explicit sync. Await the real result for calls with return values; do not return fake success early. Give queued work deadlines; on full queue return busy or enter fault handling, never drop silently.
- Different dispatcher plus `Reenterable` combinations change ordering, throughput, and tail latency. Confirm with same-connection vs. multi-connection, same-method vs. different-method, and sync vs. async handler comparisons.
- One observed single-connection case (loopback, `WaitInvoke`, async-wait handler): `Concurrency` plus reentrant allowed overlapped waits and raised throughput, while non-reentrant serialized; `Immediate` serialized either way. Treat this as a scoped example only. Dispatchers are normally created per Dmtp connection and reentrancy control is per Caller; cross-connection shared state still needs its own sync.
- Exception path must be verified separately: in one scoped combination (Immediate, non-reentrant, custom selector, `WaitInvoke`), a handler that threw synchronously or returned a faulted task was followed by caller timeout and later legal calls on the same path failing. The losing layer was not isolated. Return expected business rejections as a serializable result DTO with explicit status/code; log unknown exceptions with context and follow the service fault policy. Do not treat "faulted task instead of sync throw" as a fix. Regress at least `normal -> illegal -> explicit reject -> same-connection legal call still succeeds`, for sync and async faults separately.

### Feedback modes (`FeedbackType`)

- `OnlySend`: local send return proves neither receipt nor execution; local send can still fail. Build business ack, query, replay, and dedup when reliability matters.
- `WaitSend`: peer receipt only, not successful execution.
- `WaitInvoke`: waits for the remote result; always set timeout/cancel. Timeout or disconnect does not prove non-execution, so use request IDs, idempotency, and status queries instead of blind re-execution.
- `void` return does not imply `OnlySend`; configure the invoke option explicitly and confirm by test. For `WaitInvoke`, check the business result, not just absence of exceptions. Whether framework exceptions round-trip depends on version, dispatcher, and path; verify.

### Custom serializers and compatibility

- Check the serialization marker and the installed selector together. A custom selector installed via `DmtpRpcOption.SerializationSelector` can carry a different wire format under an existing `SerializationType` marker; such peers cannot interop with default-format peers.
- Freeze both ends: framework package version, selector and its config, DTO field layout, method names, enum values, nullable rules; test bidirectional calls.
- Validate frame length, parameter length, and remaining bytes before read/alloc; bound nested arrays/strings/collections before allocation, not only after deserialization. Test negative/oversize/truncated/malformed input, unknown enums, and DTO version skew. If the first request needs a shared selector, mismatched config may make it undecodable, so fix config up front or use a separately decodable handshake.

### Serialization performance: selection order

- Do not rank by serializer name. Measure the same DTO, content, repeat count, and caching: encode/decode time, bytes, alloc/GC, P50/P95/P99 plus cold start, then a full two-ended RPC A/B with identical dispatcher, feedback mode, and concurrency. Report per-call latency, not total time divided by concurrency; async stages can overlap, so do not sum stage times.
- Cost includes sender encode, receiver decode, alloc/copy, wire bytes, and the same again for the response. Small high-frequency messages make fixed per-call dispatch/queue/network cost significant; slow business work shrinks its share. Measure the real load.
- Optimize in order, A/B each step under identical contracts: trim fields first and measure bytes plus codec time; bounded batching for small messages while measuring added per-message delay, peak memory, and timeouts; remove duplicate serialization, temp arrays, extra copies, and per-call allocations, reusing buffers or writing to `IBufferWriter` only when the send lifetime is safe; then compare default FastBinary, MemoryPack, and manual `IPackage` on CPU, allocation, maintainability, and evolution, not codec speed alone; finally check alloc, GC pauses, and tail latency.
- Treat external benchmark tables as scoped to their workload, DTO, batching, runtime, and implementation; do not reuse them to rank the current project.

### Timeout, retry, and bidirectional calls

- Side-effect operations: request ID plus idempotent handling plus status query; never blindly resend as a new request after timeout.
- One-way notifications need app-level receipt, dedup, and loss recovery, with explicit full/closed/failed behavior and observable logs.
- In duplex RPC, reverse calls must use the actor bound to the authenticated connection and rebind after reconnect; do not use a global last-connection.
- Avoid circular waits among handlers, background workers, and reverse calls. Bound queues and pending counts; complete or cancel waiters on close. Control-channel order says nothing about independent transports or background tasks; correlate explicitly at the business layer.

### Validation checklist

After locking source and package versions, cover with repeatable two-ended tests: scheduler-by-reentrancy (concurrency, ordering, slow handlers, faults, reverse calls, full queues, reconnects, same and multi-connection); serialization (round-trips, version changes, nulls, boundary lengths, malformed input, size/alloc/GC); feedback modes (`OnlySend`/`WaitSend`/`WaitInvoke` behavior, disconnect/timeout, business ack and dedup); performance (fixed load and client concurrency, throughput/CPU/memory/error rate plus end-to-end P50/P95/P99, serialization and full RPC measured separately).

## `TouchSocket.Http`

Use for HTTP/1.1 server/client, WebSocket, static pages, SSE, form files, upload/download, and HTTP plugin pipelines.

Key concepts:

- HTTP plugin base: `IHttpPlugin`, `DefaultHttpServicePlugin`, authentication/static-page/CORS plugins.
- WebSocket: `IWebSocket`, `IWebSocketClient`, `WebSocketFeature`, `WebSocketFeatureOptions`, `WebSocketCommandLinePlugin`, WebSocket plugin interfaces.
- SSE: `SseReader`, `SseWriter`, `SseMessage`, HTTP SSE extension methods.
- Static files: `HttpStaticPagePlugin`, `StaticPageOptions`, content-type provider.
- HTTP values/forms/files: `IMultifileCollection`, form file abstractions, HTTP value extensions.

Use `TouchSocket.WebApi` for RPC-style HTTP endpoints; use `TouchSocket.Http` directly for low-level HTTP/WebSocket behavior.

## `TouchSocket.WebApi`

Use for REST-like RPC over HTTP.

Key concepts:

- Attributes: `WebApiAttribute`, `RouterAttribute`, `RegexRouterAttribute`, `FromBodyAttribute`, `FromFormAttribute`, `FromHeaderAttribute`, `FromQueryAttribute`, `EnableCorsAttribute`, response metadata attributes.
- Clients: `WebApiClient`, `WebApiClientSlim`, `IWebApiClient`, `IWebApiClientBase`.
- Context: `IHttpCallContext`, `IWebApiCallContext`.
- Plugins: `WebApiParserPlugin`, `OpenApiPlugin`, `IWebApiRequestPlugin`, `IWebApiResponsePlugin`.
- Options: `WebApiOption`, `OpenApiOption`.
- Mapping/OpenAPI: `IWebApiMapping`, `MappingMethod`, `RouteMatchResult`, OpenAPI model types.

Use `TouchSocket.WebApi.Swagger` only when the task needs a browser Swagger UI for a TouchSocket WebApi service.

## `TouchSocket.JsonRpc`

Use for JSON-RPC over TCP, HTTP, or WebSocket.

Key concepts:

- Attribute: `JsonRpcAttribute`.
- Clients: `TcpJsonRpcClient`, `HttpJsonRpcClient`, `WebSocketJsonRpcClient`, `ClientWebSocketJsonRpcClient`.
- Interfaces: `IJsonRpcClient`, `ITcpJsonRpcClient`, `IHttpJsonRpcClient`, `IWebSocketJsonRpcClient`, `IJsonRpcCallContext`.
- Parser plugins: `TcpJsonRpcParserPlugin`, `HttpJsonRpcParserPlugin`, `WebSocketJsonRpcParserPlugin`.
- Options: `JsonRpcOption`, `TcpJsonRpcOption`, `HttpJsonRpcOption`, `WebSocketJsonRpcOption`.
- Proxy: `JsonRpcDispatchProxy`.

Check the specific transport's client interface before writing a proxy or sample.

## `TouchSocket.XmlRpc`

Use for XML-RPC interop and XML-RPC clients/servers.

Key concepts:

- Attribute: `XmlRpcAttribute`.
- Client: `XmlRpcClient`.
- Interfaces: `IXmlRpcClient`, `IXmlRpcCallContext`.
- Plugin: `XmlRpcParserPlugin`.
- Option: `XmlRpcOption`.
- Proxy: `XmlRpcDispatchProxy`.

Prefer this when the user mentions XML-RPC, CookComputing.XmlRpcV2, XML-RPC arrays, or XML-RPC type compatibility.

## `TouchSocket.Mqtt`

Use for MQTT broker/client/server over TCP or WebSocket.

Key concepts:

- TCP: `MqttTcpService`, `MqttTcpServiceBase`, `MqttTcpClient`, `MqttTcpSessionClient`.
- WebSocket: `MqttWebSocketClient`, `MqttWebSocketFeatureOption`, WebSocket service/client interfaces.
- Actors: `MqttActor`, `MqttClientActor`, `MqttSessionActor`.
- Broker: `MqttBroker`, `MqttBrokerOption`.
- Messages: `MqttConnectMessage`, `MqttPublishMessage`, `MqttSubscribeMessage`, `MqttUnsubscribeMessage`, ACK messages, v3.1.1 and v5 partials.
- Options: `MqttConnectOptions`, subscription management options.
- Enums: `MqttProtocolVersion`, `MqttMessageType`, `QosLevel`, `MqttReasonCode`, retain/payload/property enums.
- Plugins: `IMqttConnectingPlugin`, `IMqttConnectedPlugin`, `IMqttReceivingPlugin`, `IMqttReceivedPlugin`, `IMqttClosingPlugin`, `IMqttClosedPlugin`, `IMqttMessageDiscardedPlugin`.

Sample TCP client pattern:

```csharp
var client = new MqttTcpClient();
await client.SetupAsync(new TouchSocketConfig()
    .SetRemoteIPHost("tcp://127.0.0.1:1883")
    .SetMqttConnectOptions(options =>
    {
        options.ClientId = "client1";
        options.ProtocolName = "Mqtt";
        options.Version = MqttProtocolVersion.V500;
    })
    .ConfigurePlugins(pluginManager =>
    {
        pluginManager.AddMqttConnectedPlugin(async (session, e) => { });
        pluginManager.AddMqttReceivedPlugin(async (session, e) => { });
    }));
await client.ConnectAsync();
```

For MQTT over WebSocket, check `UseMqttWebSocket` and `AddMqttWebSocketService` usage in `SimpleTest\MqttTester.cs`.

## `TouchSocket.Mqtt.Rpc`

Use when RPC calls should travel through MQTT clients. It builds on `TouchSocket.Mqtt` and `TouchSocket.Rpc`.

When a TouchSocket/TouchSocketPro source checkout is available, search for:

```powershell
rg -n "MqttRpc|Mqtt.*Rpc|MqttRpcAttribute|UseMqtt" $TouchSocketRepo\Src\TouchSocket.Mqtt.Rpc $TouchSocketRepo\SimpleTest
```

## `TouchSocket.Mcp`

Use for Model Context Protocol servers/clients.

Capabilities observed:

- stdio transport.
- Streamable HTTP transport.
- Tools, Resources, Prompts.
- Attribute style: `[McpTool]`, `[McpResource]`, `[McpPrompt]`.

Search the assembly for attribute definitions, transport options, and sample server/client builders before writing MCP code.
