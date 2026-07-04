# Assembly Catalog

Use this file to choose the package/assembly before editing or writing example code. Each entry lists the observed role and immediate project references from `Src/*.csproj`.

## Foundation

| Assembly | Role | References |
|---|---|---|
| `TouchSocket.Core` | Foundation utilities: memory/object/file pools, stream unpacking, wait pools, AppMessenger, encryption, XML storage, binary serialization, logging, DI abstractions, shared lifecycle/result types. | none |
| `TouchSocket` | Core socket networking for TCP, UDP, SSL, data adapters, connection/session abstractions, waiting client helpers, plugin contracts. | `TouchSocket.Core` |
| `TouchSocket.Core.DependencyInjection` | Adapter from TouchSocket IOC abstractions to `IServiceCollection`. | `TouchSocket.Core` |
| `TouchSocket.Core.Autofac` | Adapter from TouchSocket IOC abstractions to Autofac. | `TouchSocket.Core` |
| `TouchSocket.Hosting` | Generic Host integration for TCP, UDP, NamedPipe, DMTP, SerialPort and related services. | `TouchSocket.Core.DependencyInjection`, `TouchSocket` |
| `TouchSocket.AspNetCore` | ASP.NET Core integration for TouchSocket WebApi, DMTP, WebSocket, HTTP services. | `TouchSocket.Dmtp`, `TouchSocket.Hosting` |

## Transports And Protocols

| Assembly | Role | References |
|---|---|---|
| `TouchSocket.NamedPipe` | Named pipe server/client with TCP-like lifecycle, multi-pipe listen names, stream parsing. | `TouchSocket` |
| `TouchSocket.SerialPorts` | Serial port client/framework with connection lifecycle and data adapter support. | `TouchSocket` |
| `TouchSocket.Http` | HTTP/1.1 server/client, connection-based HTTP client, WebSocket, SSE, static pages, upload/download helpers. | `TouchSocket` |
| `TouchSocket.CoAP` | CoAP RFC 7252 client/server over UDP; CON/NON/ACK/RST and GET/POST/PUT/DELETE support. | `TouchSocket` |
| `TouchSocket.SocketIo` | Socket.IO client for Engine.IO v3/v4, Socket.IO 3.x/4.x, WebSocket/long-polling, namespaces, emit/ack. | `TouchSocket` |

## RPC And Messaging

| Assembly | Role | References |
|---|---|---|
| `TouchSocket.Rpc` | Shared RPC management platform: `RpcStore`, server lifetimes, call context, invoke options/results, dispatch proxy base, code generation helpers. | `TouchSocket.Core` |
| `TouchSocket.Rpc.RateLimiting` | Rate limiting extension for RPC families including DMTP RPC, WebApi, JsonRpc, XmlRpc. | `TouchSocket.Rpc` |
| `TouchSocket.Dmtp` | Duplex Message Transport Protocol: verification token, actor, channels, routing, DMTP RPC, file transfer, Redis feature, TCP/UDP/HTTP/WebSocket DMTP components. | `TouchSocket.Http`, `TouchSocket.Rpc` |
| `TouchSocket.WebApi` | HTTP WebApi RPC server/client, route attributes, query/form/body/header binding, OpenAPI metadata model. | `TouchSocket.Http`, `TouchSocket.Rpc` |
| `TouchSocket.WebApi.Swagger` | Embedded Swagger page for debugging TouchSocket.WebApi. | `TouchSocket.WebApi` |
| `TouchSocket.JsonRpc` | JSON-RPC server/client over TCP, HTTP, WebSocket. | `TouchSocket.WebApi` |
| `TouchSocket.XmlRpc` | XML-RPC server/client with XML-RPC type support and CookComputing.XmlRpcV2 interop intent. | `TouchSocket.WebApi` |
| `TouchSocket.Mqtt` | MQTT server/client over TCP and WebSocket; MQTT v3.1.1/v5 message model, broker, subscriptions. | `TouchSocket.Http` |
| `TouchSocket.Mqtt.Rpc` | RPC over MQTT clients using JSON encoded request/response. | `TouchSocket.Mqtt`, `TouchSocket.Rpc` |
| `TouchSocket.Mcp` | MCP server/client over stdio and Streamable HTTP, with tools/resources/prompts attributes. | `TouchSocket.Http`, `TouchSocket.Rpc` |

## Industrial And Device Protocols

| Assembly | Role | References |
|---|---|---|
| `TouchSocket.Modbus` | Modbus master library: TCP, UDP, RTU, RTU-over-TCP, RTU-over-UDP masters. | `TouchSocket.SerialPorts`, `TouchSocket` |
| `TouchSocket.Semi` | Semiconductor protocols: HSMS (SEMI E37) transport and SECS-II (SEMI E5) item parsing/serialization. | `TouchSocket` |

## Professional Edition

| Assembly | Role | References |
|---|---|---|
| `TouchSocketPro` | Professional base package, enterprise attributes/license helpers, professional socket plugin extensions. | `TouchSocket` |
| `TouchSocketPro.Dmtp` | Professional DMTP extensions: named-pipe DMTP, client factories, remote access, remote stream, relay, event bus, message queue, route packages, multithread file transfer. | `TouchSocket.Dmtp`, `TouchSocket.NamedPipe`, `TouchSocketPro` |
| `TouchSocketPro.Dmtp.PixStream` | Professional DMTP pixel stream extension, full-frame and delta-frame BGR24 transfer, route support. | `TouchSocketPro.Dmtp` |
| `TouchSocketPro.Hosting` | Professional Generic Host integration for TCP, UDP, NamedPipe, DMTP, SerialPort, NamedPipeDmtp. | `TouchSocket.Hosting`, `TouchSocketPro.Dmtp` |
| `TouchSocketPro.AspNetCore` | Professional ASP.NET Core integration with pro DMTP and base AspNetCore features. | `TouchSocket.AspNetCore`, `TouchSocketPro.Dmtp` |
| `TouchSocketPro.PlcBridges` | PLC data bridge framework: drives, operators, address ranges, request merging, PLC object/field attributes. | `TouchSocketPro` |
| `TouchSocketPro.Modbus` | Professional Modbus slave library plus PLC bridge drives for Modbus partitions. | `TouchSocket.Modbus`, `TouchSocketPro.PlcBridges`, `TouchSocketPro` |
| `TouchSocketPro.SocketIo` | Professional Socket.IO server, session service, WebSocket/long-polling server feature, DI integration. | `TouchSocket.Http`, `TouchSocket.SocketIo` |

## Source Generators

| Assembly | Consumer Area |
|---|---|
| `TouchSocket.Core.SourceGenerator` | Common source generation analyzer included by most non-generator assemblies. |
| `TouchSocket.Rpc.SourceGenerator` | RPC interface/proxy/server related generation. |
| `TouchSocket.Dmtp.SourceGenerator` | DMTP-specific generated contracts. |
| `TouchSocket.WebApi.SourceGenerator` | WebApi generated client/proxy contracts. |
| `TouchSocket.JsonRpc.SourceGenerator` | JsonRpc generated client/proxy contracts. |
| `TouchSocket.XmlRpc.SourceGenerator` | XmlRpc generated client/proxy contracts. |
| `TouchSocket.Mqtt.Rpc.SourceGenerator` | MQTT RPC generated client/proxy contracts. |
| `TouchSocketPro.PlcBridges.SourceGenerator` | PLC object/operator generated helpers. |

## Selection Heuristics

- Need raw TCP/UDP/SSL or packet framing: start with `TouchSocket`.
- Need HTTP, WebSocket, static files, SSE, upload/download: start with `TouchSocket.Http`.
- Need RPC-independent shared abstractions: start with `TouchSocket.Rpc`.
- Need DMTP actor/routing/channel/RPC/file/Redis: start with `TouchSocket.Dmtp`.
- Need REST-like RPC over HTTP: start with `TouchSocket.WebApi`; add `TouchSocket.WebApi.Swagger` only for browser debugging.
- Need JSON-RPC or XML-RPC: start with the matching RPC assembly, then inspect `TouchSocket.WebApi` for shared HTTP call context behavior.
- Need MQTT broker/client/subscription or MQTT v5: start with `TouchSocket.Mqtt`.
- Need MQTT-based RPC: use both `TouchSocket.Mqtt.Rpc` and `TouchSocket.Mqtt`.
- Need Modbus master: `TouchSocket.Modbus`. Need slave: `TouchSocketPro.Modbus`.
- Need PLC request coalescing/object mapping: `TouchSocketPro.PlcBridges`.
- Need Socket.IO client only: `TouchSocket.SocketIo`. Need Socket.IO server: `TouchSocketPro.SocketIo`.
- Need Generic Host integration: `TouchSocket.Hosting` or `TouchSocketPro.Hosting`.
- Need ASP.NET Core integration: `TouchSocket.AspNetCore` or `TouchSocketPro.AspNetCore`.
