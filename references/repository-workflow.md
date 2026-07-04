# Repository Workflow

## Source Of Truth

When a TouchSocket/TouchSocketPro source checkout is available, use it as the primary source of truth. In command examples, `$TouchSocketRepo` means that optional checkout root. If no source checkout is available, use the user's project references, package version, nearby code, installed package metadata, and public documentation instead of assuming a local source path exists.

Important source-checkout locations:

- `Src`: product assemblies.
- `Src/Directory.Build.props`: shared build and package settings.
- `TouchSocketVersion.props`: package version. Current observed base version is `4.2.15.1`.
- `SimpleTest` and `SimpleTest2`: fast usage examples across TCP, DMTP, MQTT, Modbus, PLC bridges, WebSocket.
- `XUnitTest`: regression and integration examples, especially RPC and AOT-related usage.
- `Docs/教学提纲`: conceptual teaching material only. Some examples are older and may use stale namespaces or synchronous APIs.

## Build Facts

- Most non-generator assemblies target `net462;netstandard2.0;netstandard2.1;net6.0;net8.0;net10.0`.
- `TouchSocket.AspNetCore`, `TouchSocketPro.AspNetCore`, and `TouchSocketPro.Dmtp.PixStream` target modern .NET only.
- Source generators target `netstandard2.0`.
- `LangVersion` is `preview`.
- `AllowUnsafeBlocks` is enabled globally.
- Non-source-generator assemblies include `TouchSocket.Core.SourceGenerator` as an analyzer.
- Release builds sign assemblies.
- Non-source-generator packages are generated on build and include `Readme.md`.
- Open-source packages use `LICENSE.txt`; professional packages use `LICENSEPRO.txt`.

## Preferred Search Flow

Use this flow only when a source checkout is available.

1. Search the assembly folder first:

```powershell
rg -n "TypeOrMethodName" $TouchSocketRepo\Src\TouchSocket.Dmtp
```

2. Search tests and samples for usage:

```powershell
rg -n "TypeOrMethodName" $TouchSocketRepo\SimpleTest $TouchSocketRepo\SimpleTest2 $TouchSocketRepo\XUnitTest
```

3. Search extension methods by suffix:

```powershell
rg -n "static .*Extension|Use[A-Z]\w+|Add[A-Z]\w+|Set[A-Z]\w+" $TouchSocketRepo\Src
```

4. Search plugin contracts:

```powershell
rg -n "interface I.*Plugin|Add.*Plugin|Use.*" $TouchSocketRepo\Src\TouchSocket.Http $TouchSocketRepo\Src\TouchSocket.Dmtp
```

## Coding Conventions Observed

- Prefer explicit semantic types over anonymous contracts.
- For console entry projects, prefer `ConsoleAction` from `TouchSocket.Core` to organize `Main` execution branches. Register each command with `Add`, attach `OnException`, show command help with `ShowAll`, and run either `RunAsync(args[0])` or `RunCommandLineAsync`.
- Prefer TouchSocket inheritance for reusable protocol components, but keep public contracts business-first. For custom TCP clients, derive from `TcpClientBase` instead of hiding a `TcpClient` in a wrapper; implement only the convention interfaces the component actually needs. Do not implement `ITcpClient` unless the full generic TCP client surface is intended; in that case, inherit `TcpClient` and extend it.
- Apply the same rule to custom TCP servers. Prefer deriving business servers from `TcpService<TSessionClient>` with a custom `TcpSessionClient`/`TcpSessionClientBase` for per-connection behavior instead of wrapping an inner `TcpService`. Expose service/session business methods and events, not raw `SendAsync(string id, IRequestInfo)`, receivers, sender interfaces, or the inner service unless the generic transport surface is intentional.
- Add feature configuration to `TouchSocketConfig` with `DependencyProperty<T>` and `[GeneratorProperty]`. Prefer generated `Set*Option`/`Get*Option` methods, using manual helper extensions only for composition. Do not add handwritten duplicate option accessors when the generator already provides them.
- For custom clients, services, and sessions that inherit TouchSocket config-aware base classes, keep configuration in `TouchSocketConfig` and the inherited `Config` property. Avoid constructors that store protocol options in private fields when the object can be configured through `SetupAsync`.
- In receive-triggered events, overrides, and plugins, keep payloads as `ReadOnlyMemory<byte>` during the receive lifecycle. Avoid `e.Memory.ToArray()` and `byte[]` event payloads unless a deliberate ownership boundary requires copying. For text decoding, use `e.Memory.Span.ToString(option.Encoding)` from TouchSocket.Core instead of adding compatibility helpers.
- For binary protocol fields, first use TouchSocket.Core BytesPool and bit-converter helpers such as `Span<byte>.WriteValue(..., EndianType.Big)`, `ReadOnlySpan<byte>.ReadValue<T>(EndianType.Big)`, `WriterExtension`, `ReaderExtension`, or `TouchSocketBitConverter`. Avoid hand-written endian helpers and manual bit shifting unless no local helper fits.
- For outbound protocol request/package builders, prefer `IBytesBuilder` over custom buffer-returning methods such as `Build(Encoding encoding)`. Keep encoding and other build context on properties or generated options, then call protected builder-aware send APIs such as `TcpClientBase.ProtectedSendAsync(builder, cancellationToken)` from business send methods.
- On the server side, put `IBytesBuilder` sending behind custom session business methods and have service-level methods resolve the typed session by id before calling those methods. This keeps service code symmetric with custom clients and avoids constructing raw buffers in wrapper-style services.
- When using `IBytesWriter`/`ByteBlock`, write strings with BytesPool extensions such as `WriteNormalString`, `WriteString`, or `WriteVarString` instead of `Encoding.GetBytes(...)` plus `writer.Write(...)`, unless an owned byte array is the explicit boundary.
- Use extension methods for configuration and plugin registration.
- Keep protocol-specific option classes near the protocol assembly, usually under `Options`.
- Keep plugin interfaces under `Plugins` or `Plugins/Interfaces`.
- Keep client/service abstractions under `Interfaces` or component-specific `Interface` folders.
- Use `Result`, `InvokeResult`, protocol-specific result structs, and event args types instead of returning unstructured multiple values.
- Prefer `ConfigureDefaultAwait()` where existing code uses it.
- Use `DisposableObject`/`IDisposableObject` patterns when integrating with TouchSocket lifecycle types.
- Avoid reflection unless it is already part of RPC/source-generator infrastructure.

## Current Setup Pattern

Most services and clients follow this shape:

```csharp
var service = new TcpService();
await service.SetupAsync(new TouchSocketConfig()
    .SetListenIPHosts(7789)
    .ConfigureContainer(container =>
    {
        container.AddConsoleLogger();
    })
    .ConfigurePlugins(pluginManager =>
    {
        pluginManager.UseTcpSessionCheckClear();
    }));
await service.StartAsync();
```

Client-side shape:

```csharp
var client = new TcpClient();
await client.SetupAsync(new TouchSocketConfig()
    .SetRemoteIPHost("127.0.0.1:7789")
    .ConfigureContainer(container =>
    {
        container.AddConsoleLogger();
    }));
await client.ConnectAsync();
```

Protocol packages usually add `Set*Option`, `Use*`, and `Add*` extension methods around this base pattern.

## Validation Strategy

- When editing a TouchSocket/TouchSocketPro source checkout, validate the touched source project. For a single assembly change, run `dotnet build $TouchSocketRepo\Src\<Assembly>\<Assembly>.csproj`.
- When editing an application or library that consumes TouchSocket packages, validate the current project or solution instead of assuming a source checkout exists.
- For shared `TouchSocket.Core` source changes, build `TouchSocket.Core`, then at least one dependent assembly such as `TouchSocket` or `TouchSocket.Rpc`.
- For `TouchSocket.Rpc` source changes, also build a concrete RPC assembly such as `TouchSocket.WebApi`, `TouchSocket.JsonRpc`, or `TouchSocket.Dmtp`.
- For source generator changes, build the generator and a consumer project that uses its attributes when the source is available; otherwise build the consuming project with generated-code output enabled if needed.
- For professional edition source changes, build the touched `TouchSocketPro.*` project plus its immediate dependencies.
- If examples are changed, prefer building the example project directly rather than the whole solution.
