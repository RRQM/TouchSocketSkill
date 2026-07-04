---
name: touchsocket
description: Build, review, refactor, test, or explain code that uses or changes TouchSocket and TouchSocketPro. Use for .NET networking with TouchSocket TCP/UDP/SSL/NamedPipe/SerialPort, DMTP, RPC, WebApi, JsonRpc, XmlRpc, MQTT, MCP, Modbus, CoAP, Socket.IO, PLC bridges, hosting/AspNetCore integrations, source generators, package selection, or work inside a TouchSocket/TouchSocketPro source checkout.
---

# TouchSocket

## Start Here

Use this skill whenever a task touches TouchSocket usage, package choice, source changes, examples, or API validation. If a TouchSocket/TouchSocketPro source checkout is available, treat it as the primary source of truth for implementation details. In command examples, `$TouchSocketRepo` means the optional local checkout root.

Before answering or editing:

1. Identify the target assembly or protocol from the user request.
2. Read the matching reference file below.
3. When a source checkout is available, verify current API names in `$TouchSocketRepo\Src` with `rg` before writing code.
4. When source examples are available, prefer `SimpleTest`, `SimpleTest2`, and `XUnitTest` over older prose docs.
5. If public docs and source disagree, trust the source and tests for the version being edited.

## Reference Routing

- General repo workflow, build facts, validation commands, and search patterns: read `references/repository-workflow.md`.
- Package and assembly selection across all projects: read `references/assembly-catalog.md`.
- Core services, TCP/UDP, adapters, NamedPipe, SerialPort, hosting, AspNetCore, CoAP, Socket.IO client: read `references/core-networking.md`.
- RPC, DMTP, HTTP, WebApi, JsonRpc, XmlRpc, MQTT, MQTT RPC, MCP, Swagger, rate limiting: read `references/rpc-http-messaging.md`.
- Industrial protocols and professional edition assemblies: read `references/industrial-pro.md`.
- Source generators and generated-code contracts: read `references/source-generators.md`.

## Working Rules

- Do not guess namespaces, extension method names, or option names. Search the local source checkout first when available; otherwise infer from the user's project references, package version, and nearby code.
- If a source checkout is available, start searches from `Src/<AssemblyName>` and then broaden to tests.
- For TouchSocketPro professional features, check licensing/enterprise helpers in `Src/TouchSocketPro` and the `TouchSocketPro.*` assembly before assuming open-source behavior.
- Prefer async APIs and current setup flow: `SetupAsync(new TouchSocketConfig()...)`, then `StartAsync()` for services or `ConnectAsync()` for clients.
- Configure TouchSocket objects through `TouchSocketConfig`, commonly with `SetListenIPHosts`, `SetRemoteIPHost`, `ConfigureContainer`, `ConfigurePlugins`, protocol-specific `Set*Option`, and `SetDataHandlingAdapter`.
- Add features through plugin-manager/container extension methods when available instead of manually wiring internal classes.
- For plugin work, implement the smallest matching plugin interface, for example `ITcpReceivedPlugin`, `IHttpPlugin`, `IDmtpReceivedPlugin`, `IMqttReceivedPlugin`, `IWebApiRequestPlugin`, or Socket.IO plugin interfaces.
- For data framing, use the built-in adapter types and templates when possible. Only create a custom adapter after checking existing adapters in `TouchSocket` and protocol assemblies.
- For RPC endpoints, prefer typed attributes and generated/proxy patterns already present in the corresponding RPC assembly.
- When changing code in TouchSocketPro, follow the repository's current style and do not preserve obsolete compatibility layers unless the user explicitly asks.

## Common Source Searches

Use these patterns when a TouchSocket/TouchSocketPro source checkout is available:

```powershell
rg -n "class TcpService|class TcpClient|SetListenIPHosts|SetRemoteIPHost" $TouchSocketRepo\Src\TouchSocket
rg -n "ConfigurePlugins|ConfigureContainer|AddConsoleLogger|SetDataHandlingAdapter" $TouchSocketRepo\Src
rg -n "UseDmtp|SetDmtpOption|DmtpRpc|IDmtp.*Plugin" $TouchSocketRepo\Src $TouchSocketRepo\SimpleTest
rg -n "WebApiAttribute|JsonRpcAttribute|XmlRpcAttribute|RpcAttribute|DispatchProxy" $TouchSocketRepo\Src
rg -n "MqttTcpClient|MqttTcpService|SetMqttConnectOptions|UseMqttWebSocket" $TouchSocketRepo\Src $TouchSocketRepo\SimpleTest
rg -n "ModbusTcpMaster|ModbusTcpSlave|PlcBridgeService|PlcFieldAttribute" $TouchSocketRepo\Src $TouchSocketRepo\SimpleTest2
```

## Validation

For repository changes, validate with the narrowest build or test that covers the touched assembly first. If the change crosses shared abstractions such as `TouchSocket.Core`, `TouchSocket`, `TouchSocket.Rpc`, or `TouchSocket.Dmtp`, broaden validation to affected dependent projects.

Use `dotnet build` against the touched `.csproj` when possible. Use `XUnitTest` or the relevant sample only when the test surface actually covers the feature.
