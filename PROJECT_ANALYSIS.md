# ACP Go SDK 项目深度分析

## 项目概览

**ACP Go SDK** 是一个用于实现 Agent Client Protocol (ACP) 的 Go 语言库。ACP 是一个标准化的通信协议，用于在代码编辑器和 AI 驱动的编码代理（agents）之间进行通信。

### 核心信息
- **项目名称**: acp-go-sdk
- **语言**: Go 1.21+
- **当前版本**: v0.6.3
- **协议版本**: 遵循 agentclientprotocol/agent-client-protocol 官方规范
- **许可证**: Apache 2.0
- **仓库**: github.com/coder/acp-go-sdk

## 架构设计

### 1. 核心架构组件

项目采用清晰的分层架构，主要包含以下核心组件：

#### 1.1 连接层 (Connection Layer)
**文件**: `connection.go` (340 行)

- **Connection 结构**: 实现了基于 JSON-RPC 2.0 的双向通信
- **关键特性**:
  - 基于换行符分隔的 JSON 消息传输
  - 异步消息处理，支持并发请求
  - 通知（notification）和请求（request）的统一处理
  - 完善的错误处理机制
  - 上下文（context）感知，支持取消操作

**核心方法**:
```go
- SendRequest[T any](...)     // 发送请求并等待类型化响应
- SendRequestNoResult(...)    // 发送无返回值的请求
- SendNotification(...)       // 发送通知（单向消息）
- receive()                   // 异步接收消息循环
```

#### 1.2 Agent 端实现
**文件**: `agent.go`, `agent_gen.go`

- **AgentSideConnection**: Agent 视角的连接封装
- **Agent 接口**: 定义了 Agent 必须实现的方法
  - `Initialize`: 初始化协议连接
  - `NewSession`: 创建新的会话
  - `Prompt`: 处理用户提示并生成响应
  - `Authenticate`: 处理身份验证
  - `Cancel`: 取消正在进行的操作

- **AgentLoader 接口**: 可选接口，支持会话加载
- **AgentExperimental 接口**: 实验性功能（如设置模型）

#### 1.3 Client 端实现
**文件**: `client.go`, `client_gen.go`

- **ClientSideConnection**: Client 视角的连接封装
- **Client 接口**: 定义了 Client 必须实现的方法
  - `RequestPermission`: 请求用户权限
  - `SessionUpdate`: 接收会话更新流
  - `ReadTextFile`: 读取文本文件
  - `WriteTextFile`: 写入文本文件

- **ClientTerminal 接口**: 可选的终端支持功能

#### 1.4 错误处理
**文件**: `errors.go` (74 行)

实现了标准 JSON-RPC 错误码：
- `-32700`: Parse error (解析错误)
- `-32600`: Invalid request (无效请求)
- `-32601`: Method not found (方法未找到)
- `-32602`: Invalid params (无效参数)
- `-32603`: Internal error (内部错误)
- `-32000`: Authentication required (需要认证)

### 2. 代码生成系统

#### 2.1 生成文件标识
所有生成的文件都以 `_gen.go` 后缀命名：
- `types_gen.go` (126,881 行) - 协议类型定义
- `agent_gen.go` (6,478 行) - Agent 端代码
- `client_gen.go` (6,065 行) - Client 端代码
- `constants_gen.go` (1,154 行) - 常量定义
- `helpers_gen.go` (685 行) - 辅助函数

#### 2.2 生成流程
```bash
# 更新 schema 版本
echo "0.6.3" > schema/version

# 下载官方 schema
make version
# 1. 从 agentclientprotocol/agent-client-protocol 下载 meta.json
# 2. 下载 schema.json
# 3. 运行 cmd/generate 生成 Go 代码
# 4. 使用 gofumpt 格式化代码
```

#### 2.3 Schema 来源
- **schema/version**: 版本号文件
- **schema/meta.json**: 协议元数据
- **schema/schema.json**: 完整的协议 schema 定义

### 3. 辅助工具和帮助函数

**文件**: `helpers.go` (260 行)

提供了丰富的构建器函数，简化 API 使用：

#### 3.1 内容块（Content Blocks）构建器
```go
TextBlock(text string) ContentBlock
ImageBlock(data, mimeType string) ContentBlock
AudioBlock(data, mimeType string) ContentBlock
ResourceLinkBlock(name, uri string) ContentBlock
ResourceBlock(res EmbeddedResourceResource) ContentBlock
```

#### 3.2 工具调用（Tool Call）构建器
```go
ToolContent(block ContentBlock) ToolCallContent
ToolDiffContent(path, newText string, oldText ...string) ToolCallContent
ToolTerminalRef(terminalID string) ToolCallContent
```

#### 3.3 会话更新（Session Update）构建器
```go
UpdateUserMessage(content ContentBlock) SessionUpdate
UpdateAgentMessage(content ContentBlock) SessionUpdate
UpdateAgentThought(content ContentBlock) SessionUpdate
UpdatePlan(entries ...PlanEntry) SessionUpdate
StartToolCall(id ToolCallId, title string, opts ...ToolCallStartOpt) SessionUpdate
UpdateToolCall(id ToolCallId, opts ...ToolCallUpdateOpt) SessionUpdate
```

#### 3.4 专用工具调用构建器
```go
StartReadToolCall(id, title, path string, opts ...) SessionUpdate
StartEditToolCall(id, title, path string, content any, opts ...) SessionUpdate
```

这些辅助函数采用了**建造者模式**和**函数式选项模式**，提供了优雅的 API 设计。

## 示例实现

### 1. Example Agent (`example/agent/main.go`)
**326 行完整示例**

展示了如何实现一个功能完整的 Agent：

**关键特性**:
- 实现所有必需的 Agent 接口
- 会话管理（创建、取消）
- 流式更新（streaming updates）
- 权限请求处理
- 工具调用演示（读取、编辑文件）
- 优雅的取消处理

**模拟的交互流程**:
1. 发送 agent_message_chunk（Agent 消息块）
2. 启动 tool_call（工具调用）- 无需权限的读操作
3. 更新 tool_call 状态为 completed
4. 启动需要权限的 tool_call - 编辑操作
5. 请求用户权限（RequestPermission）
6. 根据用户选择执行或跳过操作

### 2. Example Client (`example/client/main.go`)
**270 行完整示例**

展示了如何实现一个功能完整的 Client：

**关键特性**:
- 实现所有必需的 Client 接口
- 文件系统操作（读/写文本文件）
- 权限请求的交互式处理
- 会话更新的实时显示
- 终端功能的基础实现（可选）
- 自动连接到 example/agent 或自定义 agent

**工作流程**:
1. 启动 Agent 进程（通过 stdio）
2. 建立 ClientSideConnection
3. 初始化协议（Initialize）
4. 创建新会话（NewSession）
5. 发送提示（Prompt）
6. 处理流式更新和权限请求

### 3. 其他示例
- **example/claude-code**: 桥接 Claude Code 的示例
- **example/gemini**: 桥接 Gemini CLI 的示例（支持 -model, -sandbox, -debug 等标志）

## 测试策略

### 1. 测试文件结构
```
acp_test.go           - 核心协议测试（31,937 行）
defaults_test.go      - 默认值测试
json_parity_test.go   - JSON 序列化一致性测试
example_agent_test.go - Agent 示例测试
example_client_test.go - Client 示例测试
example_gemini_test.go - Gemini 集成测试
```

### 2. 测试覆盖范围

**连接层测试**:
- ✅ 双向错误处理
- ✅ 并发请求处理
- ✅ 消息顺序保证
- ✅ 通知处理
- ✅ 初始化流程
- ✅ 取消机制
- ✅ 会话更新等待机制
- ✅ 嵌套请求处理

**序列化测试**:
- ✅ 内容块的 JSON 编码/解码
- ✅ 工具调用内容的序列化
- ✅ 权限结果的序列化
- ✅ 会话更新的序列化
- ✅ 各种方法载荷的序列化

**默认值测试**:
- ✅ InitializeResponse.AuthMethods 默认为空数组
- ✅ AgentCapabilities 的默认值
- ✅ ClientCapabilities 的默认值

### 3. 测试数据
`testdata/` 目录包含测试用的 JSON fixtures

## 构建和开发流程

### 1. 开发命令

```bash
# 运行所有测试
go test ./...

# 运行测试并构建所有示例
make test

# 格式化代码（需要 Nix）
make fmt

# 完整检查（需要 Nix）
make check

# 运行示例
go run ./example/agent
go run ./example/client
```

### 2. 发布流程

```bash
# 方式 A: 使用 make 辅助命令
make release VERSION=0.6.3

# 方式 B: 手动执行
echo "0.6.3" > schema/version
make version
make fmt
make test
make check

# 提交和标记
git add .
git commit -m "release: v0.6.3"
git tag v0.6.3
git push origin v0.6.3
```

### 3. Nix 集成
项目使用 Nix flakes 进行可重现的构建环境：
- `flake.nix`: Nix flake 定义
- `flake.lock`: 锁定的依赖版本
- 支持 `nix develop` 进入开发环境
- `nix flake check` 验证 flake 定义

## 设计模式和最佳实践

### 1. 接口隔离原则
项目将功能分解为多个小接口：
- `Agent` - 核心 Agent 功能
- `AgentLoader` - 可选的会话加载
- `AgentExperimental` - 实验性功能
- `Client` - 核心 Client 功能
- `ClientTerminal` - 可选的终端功能

这使得实现者可以只实现所需的功能。

### 2. 类型安全的泛型
```go
func SendRequest[T any](c *Connection, ctx context.Context, 
                        method string, params any) (T, error)
```
使用 Go 1.18+ 的泛型提供类型安全的请求/响应处理。

### 3. 函数式选项模式
```go
StartToolCall(id, title, 
    WithStartKind(ToolKindRead),
    WithStartStatus(ToolCallStatusPending),
    WithStartLocations([]ToolCallLocation{{Path: path}}))
```
提供灵活且可读的 API。

### 4. 上下文感知
所有 IO 操作都接受 `context.Context`，支持：
- 超时控制
- 取消操作
- 请求追踪

### 5. 并发安全
- 使用 `sync.Mutex` 保护共享状态
- 使用 `sync.WaitGroup` 等待通知处理完成
- 使用 `atomic.Uint64` 生成请求 ID

### 6. 错误处理策略
- 返回结构化的 `*RequestError` 而非简单字符串
- 错误包含代码、消息和可选数据
- JSON 友好的错误表示

## 关键技术实现

### 1. 消息接收循环
```go
func (c *Connection) receive() {
    scanner := bufio.NewScanner(c.r)
    // 支持最大 10MB 的消息
    scanner.Buffer(make([]byte, 0, 1MB), 10MB)
    
    for scanner.Scan() {
        // 跳过空行
        // 解析 JSON
        // 路由到响应或请求处理器
        // 并发处理通知
    }
    // 连接关闭时取消上下文
}
```

### 2. 通知同步机制
```go
// 确保在请求返回前，所有通知都已处理完成
c.notificationWg.Wait()
```
这保证了客户端在 `Prompt` 返回时，所有 `SessionUpdate` 通知都已被处理。

### 3. 取消处理
```go
// Agent 端维护会话取消函数
sessionCancels map[string]context.CancelFunc

// Client 可以发送 Cancel 通知
// Agent 调用对应的 cancel 函数
```

## 协议特性支持

### 1. 核心功能
- ✅ 初始化握手
- ✅ 会话管理
- ✅ 提示处理
- ✅ 流式更新
- ✅ 工具调用
- ✅ 权限请求
- ✅ 文件系统操作
- ✅ 取消操作

### 2. 可选功能
- ✅ 会话加载（AgentLoader）
- ✅ 终端支持（ClientTerminal）
- ✅ 身份验证
- ✅ MCP 服务器集成

### 3. 实验性功能
- ✅ 设置会话模式
- ✅ 设置会话模型

## 代码统计

```
总代码行数: ~7,106 行（不包括生成代码）
生成代码: ~141,163 行
测试代码: ~50,000+ 行（包含在总计中）

核心文件分布:
- connection.go:    340 行
- helpers.go:       260 行
- errors.go:         74 行
- agent.go:          34 行
- client.go:         28 行
- doc.go:             6 行

示例代码:
- example/agent:    326 行
- example/client:   270 行
```

## 依赖关系

项目采用**零外部依赖**策略：
```go
// go.mod
module github.com/coder/acp-go-sdk
go 1.21
```

仅依赖 Go 标准库，体现了：
- 轻量级设计
- 高度可移植性
- 易于审计和维护
- 减少供应链风险

## 优势和特色

### 1. 技术优势
- **类型安全**: 完全类型化的 API，编译时检查
- **零依赖**: 无外部依赖，易于集成
- **高性能**: 基于流式 JSON-RPC，支持并发
- **可扩展**: 清晰的接口设计，易于扩展
- **完善测试**: 高测试覆盖率，包含集成测试

### 2. 开发体验
- **丰富示例**: 包含完整的 Agent 和 Client 实现
- **辅助函数**: 大量构建器函数简化使用
- **清晰文档**: README、AGENTS.md、RELEASING.md
- **自动生成**: Schema 更新自动生成代码

### 3. 生产就绪
- **错误处理**: 完善的错误类型和处理机制
- **日志支持**: 集成 slog 用于诊断
- **上下文支持**: 全面的 context 支持
- **并发安全**: 线程安全的设计

## 应用场景

### 1. AI 编码助手
- 实现 AI-powered 代码编辑器插件
- 构建智能代码补全系统
- 开发自动化重构工具

### 2. 编辑器集成
- VSCode/Vim/Emacs 等编辑器的 AI 扩展
- IDE 的智能代码生成功能
- 代码审查助手

### 3. 开发工具
- CLI 工具集成 AI 能力
- CI/CD 流程中的智能建议
- 代码质量分析工具

## 项目演进

### 版本历史
当前版本基于 ACP v0.6.3 规范，项目随官方协议同步更新。

### 贡献模式
- 遵循 Apache 2.0 许可证
- 代码风格遵循 Go 1.21 惯例
- 使用 gofumpt 格式化
- 要求通过所有测试
- 需要 Nix 环境进行格式化和检查

## 总结

**ACP Go SDK** 是一个设计精良、工程化程度高的协议实现库。它的主要特点是：

1. **架构清晰**: 分层明确，职责分明
2. **代码质量高**: 零依赖、类型安全、高测试覆盖
3. **易于使用**: 丰富的辅助函数和完整示例
4. **生产就绪**: 完善的错误处理和并发控制
5. **可维护性强**: 代码生成、自动化测试、清晰文档

该项目展示了如何用 Go 语言优雅地实现复杂的通信协议，适合作为学习 Go 项目架构和 RPC 实现的优秀案例。对于需要在编辑器和 AI Agent 之间建立标准化通信的开发者来说，这是一个理想的选择。

## 学习资源

### 官方资源
- [ACP 官方网站](https://agentclientprotocol.com)
- [协议规范仓库](https://github.com/agentclientprotocol/agent-client-protocol)
- [Go SDK 文档](https://pkg.go.dev/github.com/coder/acp-go-sdk)

### 生产实现参考
- [Gemini CLI Agent](https://github.com/google-gemini/gemini-cli) - Google 官方实现

### 快速开始
```bash
# 安装
go get github.com/coder/acp-go-sdk@v0.6.3

# 运行示例
git clone https://github.com/coder/acp-go-sdk
cd acp-go-sdk
go run ./example/client
```
