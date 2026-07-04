# Industrial And Professional Edition

## `TouchSocket.Modbus`

Use for Modbus masters.

Capabilities:

- `ModbusTcpMaster`
- `ModbusUdpMaster`
- `ModbusRtuMaster`
- `ModbusRtuOverTcpMaster`
- `ModbusRtuOverUdpMaster`

It references `TouchSocket.SerialPorts` and `TouchSocket`, so serial and TCP/UDP adapter behavior is inherited from core networking.

When a TouchSocket/TouchSocketPro source checkout is available, use these search terms:

```powershell
rg -n "ModbusTcpMaster|ModbusUdpMaster|ModbusRtuMaster|Read|Write|FunctionCode" $TouchSocketRepo\Src\TouchSocket.Modbus
```

Use `TouchSocketPro.Modbus` when the task needs slave/server support.

## `TouchSocket.Semi`

Use for semiconductor equipment protocols.

Capabilities:

- HSMS (SEMI E37) TCP transport.
- SECS-II (SEMI E5) item parsing and serialization.

Search for HSMS client/service names, SECS item model types, and sample packet parsing before implementing equipment/host behavior.

## `TouchSocketPro`

Professional base assembly. Use when a professional feature references enterprise metadata, licensing, or shared pro extensions.

Key areas:

- Enterprise/license helpers: `Enterprise`, `EnterpriseEditionAttribute`, `AuthorizedHelper`, `LicenceKeyInvalidException`.
- Professional socket plugin extension methods in `SocketProPluginsManagerExtension`.

Do not assume professional features are available in open-source packages.

## `TouchSocketPro.Dmtp`

Professional DMTP extensions. Use for features beyond open-source DMTP.

Key feature groups:

- Named-pipe DMTP: `NamedPipeDmtpClient`, `NamedPipeDmtpService`, `NamedPipeDmtpSessionClient`, named-pipe DMTP interfaces.
- Factories: `DmtpClientFactory<TClient>`, `TcpDmtpClientFactory`, `HttpDmtpClientFactory`.
- Relay: `DmtpRelayFeature`, `IDmtpRelayActor`, `IRelayService`, relay listeners, registering/unregistering plugins, `RelayMode`, `RelayType`, `RelayStatus`.
- Remote stream: `DmtpRemoteStreamFeature`, `IDmtpRemoteStreamActor`, `DmtpRemoteStream`, `DmtpResponseStream`, `DmtpRemoteStreamOption`, loading-stream plugin/event args.
- Remote access: `RemoteAccessFeature`, `IDmtpRemoteAccessActor`, `RemoteAccessOption`, `RemoteAccessMode`, `RemoteAccessType`, remote file/directory result models.
- Event bus: `EventBus`, `EventSender`, `EventRaiser`, `EventUnit`, `AccessType`, `RaiseSourceType`.
- Message queue: `MessageQueuePlugin`, delivery event args.
- Route packages: `DmtpRouterPackageFeature`, `DmtpRouterPackageActor`, `DmtpRouterPackage`, route-package options/plugins.
- File transfer: `MultithreadingFileOperator` and client-factory file transfer extensions.

Typical relay setup shape from samples:

```csharp
var service = new TcpDmtpService();
await service.SetupAsync(new TouchSocketConfig()
    .SetListenIPHosts(7789)
    .SetDmtpOption(option => option.VerifyToken = "Dmtp")
    .ConfigureContainer(container =>
    {
        container.AddConsoleLogger();
        container.AddDmtpRelayService();
    })
    .ConfigurePlugins(pluginManager =>
    {
        pluginManager.UseDmtpRelay();
        pluginManager.AddDmtpRelayRegisteringPlugin(async (actor, e) => { });
    }));
await service.StartAsync();
```

## `TouchSocketPro.Dmtp.PixStream`

Use for image frame transfer over DMTP.

Capabilities:

- Full-frame transfer on first request.
- Delta-frame transfer after the first request.
- BGR24 pixel format, 3 bytes per pixel, row-major.
- Direct and routed frame retrieval.
- Provider model through `IPixStreamProvider`.

Typical shape:

```csharp
pluginManager.UseDmtpPixStream().SetProvider(new MyCameraProvider());

var frame = await client.GetDmtpPixStreamActor().GetPixFrameAsync(cancellationToken);
var routed = await client.GetDmtpPixStreamActor().GetPixFrameAsync(targetId, cancellationToken);
```

## `TouchSocketPro.PlcBridges`

Use for PLC data bridge abstractions and request coalescing.

Key concepts:

- Service: `PlcBridgeService`, `IPlcBridgeService`.
- Drives: `PlcDrive`, `PlcDrive<TValue>`, `MemoryPlcDrive<TValue>`.
- Operators: `IPlcOperator<TValue>` and operator extensions.
- PLC objects: `PlcObject`, `PlcPropertyInfo`, `PlcFieldAttribute`, `PlcFieldAttribute<T>`.
- Values: `PlcValue`, `PlcValue<TValue>`, `ReadableValue<TValue>`, `WritableValue<TValue>`, pooled readable values, read/write value collections.
- Options: `PlcDriveOption`.
- Utility: `PlcUtility`.

Important behavior:

- Drives own address ranges with `Start` and `Count`.
- Adjacent read/write requests can be merged.
- Same-group drives execute serially.
- Value types are generally constrained to `unmanaged`.
- Results use `Result`-style objects rather than throwing for normal operation outcomes.

Use `SimpleTest2\Program.cs` for an observed PLC bridge sample. It shows adding drives, `SetupAsync`, `StartAsync`, and generated/object-style property operations.

## `TouchSocketPro.Modbus`

Use for Modbus slaves and Modbus-backed PLC bridge drives.

Slave components:

- `ModbusTcpSlave`
- `ModbusUdpSlave`
- `ModbusRtuSlave`
- `ModbusRtuOverTcpSlave`
- `ModbusRtuOverUdpSlave`

PLC bridge drives:

- `ModbusCoilsDrive`
- `ModbusDiscreteInputsDrive`
- `ModbusHoldingRegistersDrive`
- `ModbusInputRegistersDrive`
- `ModbusDrive<TValue>`
- `ModbusDriveOption`

Data/point model:

- `Partition`
- `DataPartition<T>`, `BooleanDataPartition`, `ShortDataPartition`
- `ModbusObject`, `ModbusPropertyAttribute`
- `ModbusDataLocater`, `IModbusDataLocater`, `IDataLocaterMaster`
- `ModbusSlavePointOption`, `IModbusSlavePoint`
- `ModbusResult`

Plugins:

- `IModbusSlaveExecutingPlugin`
- `IModbusSlaveExecutedPlugin`

Use this assembly when the task mentions Modbus server/slave simulation, PLC bridge over Modbus, partition mapping, or slave request interception.

## `TouchSocketPro.SocketIo`

Use for Socket.IO server. It includes base client functionality via `TouchSocket.SocketIo` but adds server-side features.

Key capabilities:

- `ISocketIoService` session collection.
- WebSocket plus HTTP long-polling.
- HTTP polling to WebSocket upgrade.
- Ping interval, ping timeout, max payload, session timeout, upgrade control.
- Connection verification via server option callback.
- Server plugins such as Socket.IO event and handshaked plugins.
- DI registration through `AddSocketIoService()`.
- Plugin setup through `UseSocketIoServer(...)`.

Typical setup:

```csharp
container.AddSocketIoService();

pluginManager.UseSocketIoServer(options =>
{
    options.SetUrl("/socket.io/");
    options.PingInterval = 25000;
    options.PingTimeout = 20000;
    options.AllowUpgrade = true;
});
```

## `TouchSocketPro.Hosting` And `TouchSocketPro.AspNetCore`

Use professional hosting integrations when the application is built around Generic Host or ASP.NET Core and requires professional features such as NamedPipe DMTP, professional DMTP relay/remote features, or professional socket extensions.

Prefer open-source `TouchSocket.Hosting`/`TouchSocket.AspNetCore` unless the requested feature explicitly requires `TouchSocketPro.*`.
