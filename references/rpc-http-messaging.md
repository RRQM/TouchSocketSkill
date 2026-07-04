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
