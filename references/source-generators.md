# Source Generators

All TouchSocket source generators target `netstandard2.0` and reference `Microsoft.CodeAnalysis.CSharp`. Most non-generator assemblies include `TouchSocket.Core.SourceGenerator` as an analyzer through `Src/Directory.Build.props`.

## General Rules

- Do not hand-write generated code unless the task is specifically to inspect generated output.
- Search generator attributes and incremental providers before changing public contracts.
- When changing generator input attributes, build at least one consumer assembly.
- Keep generator APIs compatible with the current repository's target frameworks and analyzer packaging style.
- Check `AnalyzerReleases.Shipped.md` and `AnalyzerReleases.Unshipped.md` where present.

## Assemblies

| Generator | Use Area |
|---|---|
| `TouchSocket.Core.SourceGenerator` | Common generated helpers and analyzer support used by most assemblies. |
| `TouchSocket.Rpc.SourceGenerator` | Shared RPC generated proxies, interface clients, server glue, or invocation helpers. |
| `TouchSocket.Dmtp.SourceGenerator` | DMTP-specific RPC or actor generated contracts. |
| `TouchSocket.WebApi.SourceGenerator` | WebApi generated clients/proxies or route binding helpers. |
| `TouchSocket.JsonRpc.SourceGenerator` | JSON-RPC generated client/proxy contracts. |
| `TouchSocket.XmlRpc.SourceGenerator` | XML-RPC generated client/proxy contracts. |
| `TouchSocket.Mqtt.Rpc.SourceGenerator` | MQTT RPC generated client/proxy contracts. |
| `TouchSocketPro.PlcBridges.SourceGenerator` | PLC object, field, operator, or bridge generated helpers. |

## Search Patterns

Use these patterns only when a TouchSocket/TouchSocketPro source checkout is available. Without a source checkout, inspect the consumer project's analyzer/package references, generated-output configuration, and nearby attribute usage instead.

```powershell
rg -n "IIncrementalGenerator|ISourceGenerator|GeneratorAttributeSyntaxContext|RegisterSourceOutput|SyntaxProvider" $TouchSocketRepo\Src\*.SourceGenerator
rg -n "Attribute|RpcAttribute|WebApiAttribute|JsonRpcAttribute|XmlRpcAttribute|DmtpRpcAttribute|PlcFieldAttribute" $TouchSocketRepo\Src
rg -n "Generated|partial|SourceGenerator|Generator" $TouchSocketRepo\XUnitTest $TouchSocketRepo\SimpleTest
```

## RPC Generator Work

For RPC-related generator tasks:

1. Identify the transport attribute: `DmtpRpcAttribute`, `WebApiAttribute`, `JsonRpcAttribute`, `XmlRpcAttribute`, or MQTT RPC-specific attributes.
2. Inspect the concrete protocol generator and `TouchSocket.Rpc.SourceGenerator`.
3. Inspect test interfaces under `XUnitTest`, especially RPC class libraries.
4. Build the generator and a consumer project.

## PLC Generator Work

For `TouchSocketPro.PlcBridges.SourceGenerator`:

1. Inspect `PlcFieldAttribute` and `PlcFieldAttribute<T>`.
2. Inspect `PlcObject`, `PlcDrive<TValue>`, `IPlcOperator<TValue>`, and generated extension expectations.
3. Use `SimpleTest2\Program.cs` as an example of PLC object methods and operator-backed property set/get flows.
4. Build `TouchSocketPro.PlcBridges.SourceGenerator` and `TouchSocketPro.PlcBridges`, then a sample or test project that consumes generated members.
