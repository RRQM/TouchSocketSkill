# TouchSocket Skill

这是一个面向 Codex 的 TouchSocket 专用 Skill 仓库，用于让 Codex 在实现、审查、重构或解释 TouchSocket / TouchSocketPro 相关代码时，优先遵循 TouchSocket 当前源码、程序集边界、配置方式、协议适配器、源码生成器和若汝棋茗风格约定。

仓库根目录本身就是一个完整 Skill。安装后 Skill 名称为 `touchsocket`。

## 适用场景

当任务涉及以下内容时使用本 Skill：

- TouchSocket TCP、UDP、SSL、NamedPipe、SerialPort、KCP、Socket.IO、CoAP 等网络组件。
- DMTP、RPC、WebApi、JsonRpc、XmlRpc、MQTT、MCP、Swagger、限流等消息与 RPC 组件。
- TouchSocketPro 工业协议、Modbus、PLC bridge、专业版程序集和授权相关代码。
- `TouchSocketConfig`、`DependencyProperty<T>`、`[GeneratorProperty]`、源生成配置扩展。
- 自定义 TCP 客户端、服务端、Session、插件、数据包适配器、业务优先 API 设计。
- `IBytesBuilder`、`IBytesWriter`、BytesPool、大小端读写、协议报文构建和解析。
- 在可用的 TouchSocket/TouchSocketPro 源码、项目引用、包版本或公开资料中验证 API。

## 目录结构

```text
TouchSocketSkill/
├── SKILL.md
├── README.md
├── LICENSE
├── agents/
│   └── openai.yaml
└── references/
    ├── assembly-catalog.md
    ├── core-networking.md
    ├── industrial-pro.md
    ├── repository-workflow.md
    ├── rpc-http-messaging.md
    └── source-generators.md
```

## Skill 规范

本仓库按 Codex Skill 规范组织：

- `SKILL.md` 是必须文件，包含 YAML frontmatter 和主路由说明。
- `references/` 放按需加载的详细参考资料，避免 `SKILL.md` 过大。
- `agents/openai.yaml` 提供 Skill 列表展示所需的 UI 元数据。
- 根目录保持可直接安装的 Skill 形态，不需要再包一层 `touchsocket/` 目录。

`SKILL.md` 只保留入口规则、引用路由和通用工作原则；具体程序集、网络、RPC、工业协议、源生成器等细节放在 `references/` 中，符合渐进披露原则。

## 安装

### 安装到本机 Codex Skills

可以将仓库克隆或复制到 Codex skills 目录：

```powershell
git clone https://github.com/RRQM/TouchSocketSkill.git $env:USERPROFILE\.codex\skills\touchsocket
```

如果已经克隆到其他目录，也可以复制：

```powershell
Copy-Item -Recurse -Force <repo-path> $env:USERPROFILE\.codex\skills\touchsocket
```

安装后，新会话中可通过 `$touchsocket` 显式触发，或在 TouchSocket 相关任务中自动触发。

## 使用方式

示例提示：

```text
使用 $touchsocket，基于 TouchSocket 实现一个业务优先的 TCP 客户端和服务端组件。
```

```text
使用 $touchsocket，检查这个自定义 DataHandlingAdapter 是否符合 TouchSocket 当前源码约定。
```

```text
使用 $touchsocket，按 TouchSocketConfig + GeneratorProperty 的方式重构这组协议配置。
```

Codex 使用本 Skill 时应优先读取 `SKILL.md`，再根据任务路由读取对应参考文件：

- 通用仓库流程：`references/repository-workflow.md`
- 程序集选择：`references/assembly-catalog.md`
- TCP/UDP/适配器/Hosting：`references/core-networking.md`
- RPC/HTTP/MQTT/MCP：`references/rpc-http-messaging.md`
- 工业协议/专业版：`references/industrial-pro.md`
- 源生成器：`references/source-generators.md`

## 可选源码检出

本 Skill 不要求使用者拥有 TouchSocketPro 本地源码。若使用者本机有 TouchSocket/TouchSocketPro 源码检出，可在提示或环境说明中提供路径；文档中的 `$TouchSocketRepo` 仅表示这个可选源码根目录。

没有源码检出时，Codex 应根据当前项目引用、NuGet 包版本、已安装源码包、公开文档和用户提供的代码上下文判断 API，不应假设任何固定本机路径存在。

## 核心约定

- 不猜 API 名称；有源码时先用 `rg` 查源码和示例，没有源码时优先查项目引用、包版本和附近代码。
- 自定义 TCP 客户端优先继承 `TcpClientBase`，业务 API 优先，不暴露底层发送/接收接口。
- 自定义 TCP 服务端优先继承 `TcpService<TSessionClient>` 或 `TcpServiceBase<TSessionClient>`，使用自定义 Session 承载每连接业务能力。
- 配置优先走 `TouchSocketConfig`、`DependencyProperty<T>` 和 `[GeneratorProperty]` 生成方法，不重复手写 `GetXxxOption` / `SetXxxOption`。
- 协议 option 只放协议行为；host、port、日志、容器、插件等基础设施配置放在 `TouchSocketConfig` 对应扩展中。
- 出站协议报文优先实现 `IBytesBuilder`，通过 `IBytesWriter` 写入，不用 `Build(Encoding encoding)` 这类返回缓冲区的自定义方法。
- 接收生命周期内优先使用 `ReadOnlyMemory<byte>` / `ReadOnlySpan<byte>`，避免 `ToArray()` 和不必要的 `byte[]`。
- 字符串写入 `IBytesWriter` 优先用 `WriteNormalString`、`WriteString`、`WriteVarString`。
- 大小端读写优先用 TouchSocket.Core BytesPool、`WriterExtension`、`ReaderExtension`、`Span.WriteValue`、`ReadOnlySpan.ReadValue`、`TouchSocketBitConverter`。

## 维护流程

修改 Skill 后建议执行以下校验：

```powershell
$target = Join-Path $env:TEMP 'codex-pyyaml-validate'
if (-not (Test-Path $target)) {
    python -m pip install PyYAML -t $target --quiet
}
$env:PYTHONPATH = $target
python <skill-creator-path>\scripts\quick_validate.py <repo-path>
```

如果修改了参考文件中的具体 TouchSocket API 名称，且本机有源码检出，建议在 `$TouchSocketRepo\Src` 中用 `rg` 验证真实源码。

## 发布前检查

发布到 GitHub 前建议检查：

```powershell
git status --short
git diff -- SKILL.md references agents README.md
```

并至少运行一次 Skill 校验脚本，确保 frontmatter、名称和基础结构仍符合 Skill 规范。
