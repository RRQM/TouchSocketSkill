# Core Networking Assemblies

## `TouchSocket.Core`

Use for shared infrastructure. Check this assembly before creating a new utility, lifecycle abstraction, result type, wait handle, logger abstraction, serializer, or DI helper.

Important areas:

- `WaitPool`: `WaitHandlePool<T>`, `WaitDataAsync<T>`, `AsyncWaitData<T>`, `WaitResult`, `IWaitResult`, `WaitDataStatus`.
- `BytesPool` and `TouchSocketBitConverter`: byte readers/writers, span read/write extensions, endian conversion, byte blocks, pooled writers, and package serialization helpers.
- `Threading`: locks, timeout tokens, easy task helpers.
- `DependencyObject`/resolver/container abstractions used by most higher-level clients and services.
- `DisposableObject` and `IDisposableObject` lifecycle patterns.
- Serializer/converter and binary helpers used by protocol packages.
- Logging abstractions and console logger registration used in samples.

When implementing shared behavior, avoid protocol assumptions here. Higher-level assemblies should own protocol-specific options, plugins, and event args.

## `TouchSocket`

Use for TCP, UDP, SSL, socket clients/services, base plugin/event contracts, senders, receivers, and data adapters.

### Custom Client Rules

- Prefer deriving reusable business TCP clients from `TcpClientBase` instead of wrapping an inner `TcpClient`. This keeps the type inside TouchSocket's lifecycle, resolver, logging, adapter, and plugin pipeline.
- Keep the public API business-first. Expose domain operations such as `SendTextAsync`, `SendCommandAsync`, `ReadStatusAsync`, or protocol-specific request methods. Do not expose `SendAsync(IRequestInfo)`, receiver creation, raw byte sending, or other generic transport APIs unless they are part of the intended business contract.
- Implement only the TouchSocket convention interfaces that match the public contract. For example, implement `ITcpConnectableClient` when the type should be connectable. Implement `IClientSender`, `IRequestInfoSender`, or `IReceiverClient<IReceiverResult>` only when callers should use those generic transport capabilities.
- Do not implement `ITcpClient` just because the type is TCP-based. `ITcpClient` means the full generic TCP client surface is public; if that is desired, inherit `TcpClient` and extend it. If the component is domain-specific, inherit `TcpClientBase` and expose narrower business APIs.
- Route business send methods to protected base APIs such as `ProtectedSendAsync`, route `ConnectAsync` to `TcpConnectAsync` when implementing connectability, and override `OnTcpConnected`/`OnTcpClosed`/`OnTcpReceived` to raise business events before calling `base`.

### Custom Server Rules

- Apply the same business-first rules to TCP servers as to clients. Prefer deriving reusable business servers from `TcpService<TSessionClient>` instead of wrapping an inner `TcpService` when the server itself is the component being delivered.
- Use a custom `TcpSessionClient` or `TcpSessionClientBase` descendant for per-connection business behavior, for example `RobokitTcpSessionClient`, and put session-level send helpers such as `SendPacketAsync` or `SendTextAsync` there. Session classes can route those helpers to protected send APIs, including `ProtectedSendAsync(IBytesBuilder, cancellationToken)` through TouchSocket's injected TCP send region.
- Keep service public APIs business-first. Expose operations such as `SendPacketAsync(clientId, request)`, `BroadcastTextAsync`, `KickClientAsync`, or domain-specific query methods. Do not expose generic `SendAsync(string id, IRequestInfo)`, `CreateReceiver`, raw sender interfaces, or the inner `TcpService` unless that generic transport surface is the intended contract.
- Implement only the TouchSocket convention interfaces the service contract needs. Do not implement `ITcpService` merely because the type listens on TCP; if callers need the full generic service surface, inherit `TcpService`/`TcpService<TSessionClient>` and extend it deliberately.
- Route `StartAsync`/`StopAsync` through the base service lifecycle, configure through `TouchSocketConfig`, and override `OnTcpConnected`/`OnTcpClosed`/`OnTcpReceived` on `TcpService<TSessionClient>` to raise business events before calling `base`.
- When the server sends protocol packages, prefer building requests with `IBytesBuilder` and sending from the custom session's protected send path. If a service-level method targets a client id, resolve the typed session and call its business send method instead of constructing raw buffers in the service wrapper.

### Receive Memory Rules

- In callbacks, plugins, overrides, or events triggered by receive data, keep payloads as `ReadOnlyMemory<byte>` or protocol-specific request objects for the whole receive lifecycle.
- Do not call `e.Memory.ToArray()` by default, and do not expose `byte[]` payloads from business event args unless the business contract explicitly requires ownership outside the receive lifecycle.
- When decoding text from receive memory or spans, prefer TouchSocket.Core's extension method, for example `e.Memory.Span.ToString(option.Encoding)`. This works on older targets such as `net472` through TouchSocket's extension API.
- If data must survive beyond the receive callback, make the copy explicit at the boundary and document that the caller owns the copied payload.

### Generated Configuration Rules

- Put custom feature options on `TouchSocketConfig` via `DependencyProperty<T>` and `[GeneratorProperty(TargetType = typeof(TouchSocketConfig))]`.
- Use `ActionMode = true` for option objects so consumers can write `.SetXxxOption(option => { ... })`.
- Prefer generated `.SetXxxOption(...)` and `.GetXxxOption(...)` methods in setup code and runtime code. Do not hand-write duplicate getters/setters such as `GetXxxOptionOrDefault` when `GeneratorProperty` already generates the accessors. Use manual helper extensions only to compose generated config calls with adapter/plugin installation.
- Keep protocol option objects focused on protocol behavior. Do not put transport endpoint settings such as host/port or infrastructure concerns such as console logging into a protocol option; configure those through `TouchSocketConfig` with `SetRemoteIPHost`, `SetListenIPHosts`, `ConfigureContainer`, and related TouchSocket extensions.
- If a custom protocol needs packet framing, configure the adapter from the generated option values, commonly by setting `SetSingleStreamDataHandlingAdapter`.
- For components that inherit TouchSocket setup/config base types, prefer the inherited `Config` flow. Configure them with `SetupAsync(new TouchSocketConfig()...)` and read options from `this.Config.GetXxxOption...()` inside clients, services, and sessions instead of storing a separate options field passed through the constructor.

Key public concepts to search:

- Services and clients: `TcpService`, `TcpService<TClient>`, `TcpClient`, `UdpSessionBase`, `TcpSessionClient`, `ITcpService`, `ITcpClient`, `ITcpSessionClient`, `IUdpSession`.
- Lifecycle/event args: `ConnectingEventArgs`, `ConnectedEventArgs`, `ClosingEventArgs`, `ClosedEventArgs`, `ReceivedDataEventArgs`, `UdpReceivedDataEventArgs`.
- Plugins: `ITcpConnectingPlugin`, `ITcpConnectedPlugin`, `ITcpReceivingPlugin`, `ITcpReceivedPlugin`, `ITcpSendingPlugin`, `ITcpClosingPlugin`, `ITcpClosedPlugin`, `IUdpReceivingPlugin`, `IUdpReceivedPlugin`, `IUdpSendingPlugin`.
- Sender/receiver abstractions: `IClientSender`, `IIdSender`, `IRequestInfo`, `IReceiverResult`, `IReceiverClient<TResult>`.
- Waiting client helpers: `IWaitingClient<TClient,TResult>`, `WaitingOptions`, `ResponsedData`, waiting extensions.
- Transport abstractions: `ITransport`, `ITransportReader`, `ITransportWriter`.

Common setup:

```csharp
var service = new TcpService();
await service.SetupAsync(new TouchSocketConfig()
    .SetListenIPHosts(7789)
    .SetDataHandlingAdapter(() => new NormalDataHandlingAdapter())
    .ConfigureContainer(container => container.AddConsoleLogger())
    .ConfigurePlugins(pluginManager =>
    {
        pluginManager.UseTcpSessionCheckClear();
    }));
await service.StartAsync();
```

When receiving data, prefer plugin interfaces for reusable behavior and tests. Direct event assignment is acceptable for small samples but plugin contracts scale better inside library code.

## Data Handling Adapters

TouchSocket solves packet sticking/splitting through adapters. Before custom parsing, search built-in adapters in the available source checkout; when no checkout is available, inspect package docs, metadata, and nearby project usage instead.

```powershell
rg -n "DataHandlingAdapter|FixedHeader|Terminator|FixedSize|PackageAdapter|IRequestInfo" $TouchSocketRepo\Src\TouchSocket
```

Use built-in templates for fixed header, fixed length, delimiter/terminator, or custom package formats. Write a custom adapter only when the payload format is genuinely unique.

When reading or writing numeric fields in `Memory`/`Span`/protocol headers, prefer TouchSocket.Core built-ins before writing manual bit shifts or endian helpers:

- `Span<byte>.WriteValue(value, EndianType.Big)` or `WriterExtension.WriteValue(ref writer, value, EndianType.Big)` for big-endian writes.
- `ReadOnlySpan<byte>.ReadValue<T>(EndianType.Big)` or `ReaderExtension.ReadValue<TReader,T>(ref reader, EndianType.Big)` for big-endian reads.
- `TouchSocketBitConverter.GetBitConverter(endianType)`, `TouchSocketBitConverter.BigEndian`, or BytesPool readers/writers when a lower-level converter is needed.

When a protocol request/package object needs to build outbound bytes, prefer implementing `IBytesBuilder` with `MaxLength` and `Build<TWriter>(ref TWriter writer)` instead of adding ad hoc methods such as `Build(Encoding encoding)` that return buffers. Pass build context such as encoding, endian choice, protocol version, or options through properties/configuration before sending. On custom `TcpClientBase` descendants, route business send methods to `ProtectedSendAsync(builder, cancellationToken)` when the request implements `IBytesBuilder`.

When writing strings to an `IBytesWriter`, prefer BytesPool writer extensions instead of first allocating encoded byte arrays. Use `WriterExtension.WriteNormalString(ref writer, value, encoding)` when the protocol already carries length elsewhere, or `WriteString`/`WriteVarString` when the TouchSocket length-prefixed formats are intended.

## SSL And Connection Options

TCP clients and services expose SSL-related configuration through `TouchSocketConfig` extension methods such as `SetClientSslOption` and service-side SSL options. Search exact names before use because overloads vary by target framework.

## `TouchSocket.NamedPipe`

Use for local IPC with TCP-like shape.

Search terms:

- `NamedPipeClientBase`
- `NamedPipeServiceBase`
- `NamedPipeSessionClientBase`
- `INamedPipeStreamClient`
- named-pipe plugin and option extension methods.

Prefer this assembly when the user wants same-machine communication without opening TCP ports.

## `TouchSocket.SerialPorts`

Use for serial port communication with TouchSocket lifecycle and adapter support.

Search terms:

- `SerialPortClientBase`
- `SetSerialPortOption`
- serial port adapter examples in `SimpleTest\SerialPortTester.cs`.

Use the same adapter strategy as TCP when serial payloads need frame parsing.

## `TouchSocket.Hosting`

Use when integrating TouchSocket services into `Microsoft.Extensions.Hosting`.

Look for registration extensions under `TouchSocket.Hosting`, especially `Add*Service`, hosted-service registration, and options classes. Prefer this assembly for Worker Services, console daemons, and DI-first applications.

## `TouchSocket.AspNetCore`

Use for ASP.NET Core integration around TouchSocket WebApi, DMTP, WebSocket, and HTTP services. It targets modern .NET only.

Read this assembly when the user asks to host TouchSocket inside an ASP.NET Core app, bridge ASP.NET Core DI/logging, or expose TouchSocket WebApi/DMTP through the web host.

## `TouchSocket.CoAP`

Use for constrained-device CoAP over UDP.

Capabilities observed from package metadata:

- CoAP RFC 7252 client/server.
- CON, NON, ACK, RST message types.
- GET, POST, PUT, DELETE.
- Standard CoAP options.

Search this assembly for option/message/request/response classes before implementing a CoAP sample.

## `TouchSocket.SocketIo`

Use for Socket.IO client work.

Key capabilities:

- Socket.IO 3.x/4.x and Engine.IO v3/v4.
- WebSocket and HTTP long-polling, with automatic upgrade.
- Namespace support.
- Emit/Ack event model.
- Binary attachments with placeholder support.
- System.Text.Json serializer by default, replaceable via serializer interfaces.

Typical client shape from the Readme:

```csharp
var client = new SocketIoClient();
client.Setup(new TouchSocketConfig()
    .SetRemoteIPHost("http://localhost:3000")
    .Configure<SocketIoOption>(option =>
    {
        option.EIO = EngineIoVersion.V4;
        option.Transport = EngineIoTransportType.WebSocket;
        option.Namespace = "/chat";
    }));

await client.ConnectAsync();
await client.EmitAsync("message", "hello");
var response = await client.EmitWithAckAsync("query", new object[] { "id" }, 5000);
```

For Socket.IO server features, switch to `TouchSocketPro.SocketIo`.
