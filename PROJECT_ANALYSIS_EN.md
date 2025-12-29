# ACP Go SDK - Comprehensive Project Analysis

## Project Overview

**ACP Go SDK** is a Go library implementing the Agent Client Protocol (ACP) - a standardized communication protocol for interaction between code editors and AI-powered coding agents.

### Key Information
- **Project Name**: acp-go-sdk
- **Language**: Go 1.21+
- **Current Version**: v0.6.3
- **Protocol Version**: Follows agentclientprotocol/agent-client-protocol official specification
- **License**: Apache 2.0
- **Repository**: github.com/coder/acp-go-sdk

## Architecture Design

### 1. Core Architecture Components

The project adopts a clear layered architecture with the following main components:

#### 1.1 Connection Layer
**File**: `connection.go` (340 lines)

- **Connection Structure**: Implements bidirectional JSON-RPC 2.0 communication
- **Key Features**:
  - Line-delimited JSON message transport
  - Asynchronous message processing with concurrent request support
  - Unified handling of notifications and requests
  - Comprehensive error handling mechanism
  - Context-aware with cancellation support

**Core Methods**:
```go
- SendRequest[T any](...)     // Send request and wait for typed response
- SendRequestNoResult(...)    // Send request with no return value
- SendNotification(...)       // Send notification (one-way message)
- receive()                   // Async message receive loop
```

#### 1.2 Agent-Side Implementation
**Files**: `agent.go`, `agent_gen.go`

- **AgentSideConnection**: Connection wrapper from agent's perspective
- **Agent Interface**: Defines methods that agents must implement
  - `Initialize`: Initialize protocol connection
  - `NewSession`: Create new session
  - `Prompt`: Process user prompt and generate response
  - `Authenticate`: Handle authentication
  - `Cancel`: Cancel ongoing operations

- **AgentLoader Interface**: Optional interface for session loading support
- **AgentExperimental Interface**: Experimental features (e.g., set model)

#### 1.3 Client-Side Implementation
**Files**: `client.go`, `client_gen.go`

- **ClientSideConnection**: Connection wrapper from client's perspective
- **Client Interface**: Defines methods that clients must implement
  - `RequestPermission`: Request user permission
  - `SessionUpdate`: Receive session update stream
  - `ReadTextFile`: Read text file
  - `WriteTextFile`: Write text file

- **ClientTerminal Interface**: Optional terminal support features

#### 1.4 Error Handling
**File**: `errors.go` (74 lines)

Implements standard JSON-RPC error codes:
- `-32700`: Parse error
- `-32600`: Invalid request
- `-32601`: Method not found
- `-32602`: Invalid params
- `-32603`: Internal error
- `-32000`: Authentication required

### 2. Code Generation System

#### 2.1 Generated File Identification
All generated files are named with `_gen.go` suffix:
- `types_gen.go` (126,881 lines) - Protocol type definitions
- `agent_gen.go` (6,478 lines) - Agent-side code
- `client_gen.go` (6,065 lines) - Client-side code
- `constants_gen.go` (1,154 lines) - Constants
- `helpers_gen.go` (685 lines) - Helper functions

#### 2.2 Generation Workflow
```bash
# Update schema version
echo "0.6.3" > schema/version

# Download official schema and regenerate
make version
# 1. Download meta.json from agentclientprotocol/agent-client-protocol
# 2. Download schema.json
# 3. Run cmd/generate to create Go code
# 4. Format code with gofumpt
```

#### 2.3 Schema Source
- **schema/version**: Version number file
- **schema/meta.json**: Protocol metadata
- **schema/schema.json**: Complete protocol schema definition

### 3. Helper Utilities

**File**: `helpers.go` (260 lines)

Provides rich builder functions to simplify API usage:

#### 3.1 Content Block Builders
```go
TextBlock(text string) ContentBlock
ImageBlock(data, mimeType string) ContentBlock
AudioBlock(data, mimeType string) ContentBlock
ResourceLinkBlock(name, uri string) ContentBlock
ResourceBlock(res EmbeddedResourceResource) ContentBlock
```

#### 3.2 Tool Call Builders
```go
ToolContent(block ContentBlock) ToolCallContent
ToolDiffContent(path, newText string, oldText ...string) ToolCallContent
ToolTerminalRef(terminalID string) ToolCallContent
```

#### 3.3 Session Update Builders
```go
UpdateUserMessage(content ContentBlock) SessionUpdate
UpdateAgentMessage(content ContentBlock) SessionUpdate
UpdateAgentThought(content ContentBlock) SessionUpdate
UpdatePlan(entries ...PlanEntry) SessionUpdate
StartToolCall(id ToolCallId, title string, opts ...ToolCallStartOpt) SessionUpdate
UpdateToolCall(id ToolCallId, opts ...ToolCallUpdateOpt) SessionUpdate
```

#### 3.4 Specialized Tool Call Builders
```go
StartReadToolCall(id, title, path string, opts ...) SessionUpdate
StartEditToolCall(id, title, path string, content any, opts ...) SessionUpdate
```

These helper functions use the **Builder Pattern** and **Functional Options Pattern** for elegant API design.

## Example Implementations

### 1. Example Agent (`example/agent/main.go`)
**326 lines complete example**

Demonstrates how to implement a fully functional agent:

**Key Features**:
- Implements all required Agent interfaces
- Session management (create, cancel)
- Streaming updates
- Permission request handling
- Tool call demonstration (read, edit files)
- Graceful cancellation handling

**Simulated Interaction Flow**:
1. Send agent_message_chunk
2. Start tool_call - read operation without permission
3. Update tool_call status to completed
4. Start tool_call requiring permission - edit operation
5. Request user permission (RequestPermission)
6. Execute or skip based on user choice

### 2. Example Client (`example/client/main.go`)
**270 lines complete example**

Demonstrates how to implement a fully functional client:

**Key Features**:
- Implements all required Client interfaces
- File system operations (read/write text files)
- Interactive permission request handling
- Real-time session update display
- Basic terminal functionality (optional)
- Auto-connects to example/agent or custom agent

**Workflow**:
1. Start agent process (via stdio)
2. Establish ClientSideConnection
3. Initialize protocol (Initialize)
4. Create new session (NewSession)
5. Send prompt (Prompt)
6. Handle streaming updates and permission requests

### 3. Other Examples
- **example/claude-code**: Bridge to Claude Code
- **example/gemini**: Bridge to Gemini CLI (supports -model, -sandbox, -debug flags)

## Testing Strategy

### 1. Test File Structure
```
acp_test.go           - Core protocol tests (31,937 lines)
defaults_test.go      - Default value tests
json_parity_test.go   - JSON serialization consistency tests
example_agent_test.go - Agent example tests
example_client_test.go - Client example tests
example_gemini_test.go - Gemini integration tests
```

### 2. Test Coverage

**Connection Layer Tests**:
- ✅ Bidirectional error handling
- ✅ Concurrent request processing
- ✅ Message ordering guarantees
- ✅ Notification handling
- ✅ Initialize flow
- ✅ Cancellation mechanism
- ✅ Session update waiting
- ✅ Nested request handling

**Serialization Tests**:
- ✅ Content block JSON encoding/decoding
- ✅ Tool call content serialization
- ✅ Permission outcome serialization
- ✅ Session update serialization
- ✅ Various method payload serialization

**Default Value Tests**:
- ✅ InitializeResponse.AuthMethods defaults to empty array
- ✅ AgentCapabilities defaults
- ✅ ClientCapabilities defaults

### 3. Test Data
`testdata/` directory contains JSON fixtures for testing

## Build and Development Workflow

### 1. Development Commands

```bash
# Run all tests
go test ./...

# Run tests and build all examples
make test

# Format code (requires Nix)
make fmt

# Full checks (requires Nix)
make check

# Run examples
go run ./example/agent
go run ./example/client
```

### 2. Release Process

```bash
# Method A: Use make helper
make release VERSION=0.6.3

# Method B: Manual steps
echo "0.6.3" > schema/version
make version
make fmt
make test
make check

# Commit and tag
git add .
git commit -m "release: v0.6.3"
git tag v0.6.3
git push origin v0.6.3
```

### 3. Nix Integration
Project uses Nix flakes for reproducible build environment:
- `flake.nix`: Nix flake definition
- `flake.lock`: Locked dependency versions
- Supports `nix develop` to enter dev environment
- `nix flake check` validates flake definition

## Design Patterns and Best Practices

### 1. Interface Segregation Principle
Project decomposes functionality into small interfaces:
- `Agent` - Core agent functionality
- `AgentLoader` - Optional session loading
- `AgentExperimental` - Experimental features
- `Client` - Core client functionality
- `ClientTerminal` - Optional terminal features

This allows implementers to implement only required features.

### 2. Type-Safe Generics
```go
func SendRequest[T any](c *Connection, ctx context.Context, 
                        method string, params any) (T, error)
```
Uses Go 1.18+ generics for type-safe request/response handling.

### 3. Functional Options Pattern
```go
StartToolCall(id, title, 
    WithStartKind(ToolKindRead),
    WithStartStatus(ToolCallStatusPending),
    WithStartLocations([]ToolCallLocation{{Path: path}}))
```
Provides flexible and readable API.

### 4. Context-Aware
All IO operations accept `context.Context` supporting:
- Timeout control
- Cancellation
- Request tracing

### 5. Concurrency Safety
- Uses `sync.Mutex` to protect shared state
- Uses `sync.WaitGroup` to wait for notification processing
- Uses `atomic.Uint64` for request ID generation

### 6. Error Handling Strategy
- Returns structured `*RequestError` instead of simple strings
- Errors contain code, message, and optional data
- JSON-friendly error representation

## Key Technical Implementations

### 1. Message Receive Loop
```go
func (c *Connection) receive() {
    scanner := bufio.NewScanner(c.r)
    // Support messages up to 10MB
    scanner.Buffer(make([]byte, 0, 1MB), 10MB)
    
    for scanner.Scan() {
        // Skip empty lines
        // Parse JSON
        // Route to response or request handler
        // Process notifications concurrently
    }
    // Cancel context when connection closes
}
```

### 2. Notification Synchronization
```go
// Ensure all notifications are processed before request returns
c.notificationWg.Wait()
```
This guarantees that when `Prompt` returns, all `SessionUpdate` notifications have been processed.

### 3. Cancellation Handling
```go
// Agent-side maintains session cancel functions
sessionCancels map[string]context.CancelFunc

// Client can send Cancel notification
// Agent calls corresponding cancel function
```

## Protocol Feature Support

### 1. Core Features
- ✅ Initialize handshake
- ✅ Session management
- ✅ Prompt handling
- ✅ Streaming updates
- ✅ Tool calls
- ✅ Permission requests
- ✅ File system operations
- ✅ Cancellation

### 2. Optional Features
- ✅ Session loading (AgentLoader)
- ✅ Terminal support (ClientTerminal)
- ✅ Authentication
- ✅ MCP server integration

### 3. Experimental Features
- ✅ Set session mode
- ✅ Set session model

## Code Statistics

```
Total Code Lines: ~7,106 lines (excluding generated code)
Generated Code: ~141,163 lines
Test Code: ~50,000+ lines (included in total)

Core File Distribution:
- connection.go:    340 lines
- helpers.go:       260 lines
- errors.go:         74 lines
- agent.go:          34 lines
- client.go:         28 lines
- doc.go:             6 lines

Example Code:
- example/agent:    326 lines
- example/client:   270 lines
```

## Dependencies

Project follows **zero external dependencies** strategy:
```go
// go.mod
module github.com/coder/acp-go-sdk
go 1.21
```

Only depends on Go standard library, demonstrating:
- Lightweight design
- High portability
- Easy to audit and maintain
- Reduced supply chain risk

## Strengths and Features

### 1. Technical Advantages
- **Type Safety**: Fully typed API with compile-time checking
- **Zero Dependencies**: No external dependencies, easy integration
- **High Performance**: Streaming JSON-RPC with concurrency support
- **Extensible**: Clear interface design, easy to extend
- **Well Tested**: High test coverage including integration tests

### 2. Developer Experience
- **Rich Examples**: Complete Agent and Client implementations
- **Helper Functions**: Many builder functions simplify usage
- **Clear Documentation**: README, AGENTS.md, RELEASING.md
- **Auto-Generation**: Schema updates automatically generate code

### 3. Production Ready
- **Error Handling**: Comprehensive error types and handling
- **Logging Support**: Integrated slog for diagnostics
- **Context Support**: Full context support throughout
- **Concurrency Safe**: Thread-safe design

## Use Cases

### 1. AI Coding Assistants
- Implement AI-powered code editor plugins
- Build intelligent code completion systems
- Develop automated refactoring tools

### 2. Editor Integration
- AI extensions for VSCode/Vim/Emacs
- Intelligent code generation in IDEs
- Code review assistants

### 3. Development Tools
- CLI tools with AI capabilities
- Intelligent suggestions in CI/CD pipelines
- Code quality analysis tools

## Project Evolution

### Version History
Current version based on ACP v0.6.3 specification. Project updates synchronously with official protocol.

### Contribution Model
- Follows Apache 2.0 license
- Code style follows Go 1.21 conventions
- Uses gofumpt for formatting
- Requires all tests to pass
- Needs Nix environment for formatting and checks

## Summary

**ACP Go SDK** is a well-designed, highly engineered protocol implementation library. Its main characteristics:

1. **Clear Architecture**: Well-layered with clear responsibilities
2. **High Code Quality**: Zero dependencies, type-safe, high test coverage
3. **Easy to Use**: Rich helper functions and complete examples
4. **Production Ready**: Comprehensive error handling and concurrency control
5. **Maintainable**: Code generation, automated testing, clear documentation

This project demonstrates how to elegantly implement complex communication protocols in Go. It serves as an excellent case study for learning Go project architecture and RPC implementation. For developers needing standardized communication between editors and AI agents, this is an ideal choice.

## Learning Resources

### Official Resources
- [ACP Official Website](https://agentclientprotocol.com)
- [Protocol Spec Repository](https://github.com/agentclientprotocol/agent-client-protocol)
- [Go SDK Documentation](https://pkg.go.dev/github.com/coder/acp-go-sdk)

### Production Implementation Reference
- [Gemini CLI Agent](https://github.com/google-gemini/gemini-cli) - Google's official implementation

### Quick Start
```bash
# Install
go get github.com/coder/acp-go-sdk@v0.6.3

# Run example
git clone https://github.com/coder/acp-go-sdk
cd acp-go-sdk
go run ./example/client
```

## Technical Deep Dive

### Message Flow Architecture

```
┌─────────────────┐                    ┌─────────────────┐
│                 │    JSON-RPC 2.0    │                 │
│     Client      │◄──────────────────►│     Agent       │
│                 │     over stdio     │                 │
└─────────────────┘                    └─────────────────┘
        │                                      │
        │ 1. Initialize                        │
        ├─────────────────────────────────────►│
        │◄─────────────────────────────────────┤
        │ 2. NewSession                        │
        ├─────────────────────────────────────►│
        │◄─────────────────────────────────────┤
        │ 3. Prompt                            │
        ├─────────────────────────────────────►│
        │                                      │
        │◄──── SessionUpdate (stream) ─────────┤
        │◄──── SessionUpdate ──────────────────┤
        │◄──── RequestPermission ──────────────┤
        ├───── Response ─────────────────────►│
        │◄──── SessionUpdate ──────────────────┤
        │◄─────────────────────────────────────┤
        │ 4. Cancel (optional)                 │
        ├─────────────────────────────────────►│
```

### Concurrency Model

The SDK handles concurrency at multiple levels:

1. **Connection Level**: Each connection has a dedicated receive goroutine
2. **Request Level**: Requests block waiting for response, but don't block other requests
3. **Notification Level**: Notifications spawn individual goroutines with WaitGroup tracking
4. **Session Level**: Each session can be independently cancelled

This design ensures:
- Non-blocking notification delivery
- Ordered processing guarantees where needed
- Clean cancellation semantics
- No deadlocks between client and agent

### Memory Management

- Uses `bufio.Scanner` with configurable buffer (1MB initial, 10MB max)
- Channels are buffered (size 1) to avoid goroutine leaks
- Proper cleanup of pending requests on context cancellation
- No persistent memory growth during long-running sessions

## Comparison with Other Implementations

While this is the official Go SDK, the ACP protocol has implementations in other languages:
- **TypeScript/JavaScript**: Reference implementation
- **Python**: Community implementation
- **Rust**: Community implementation

The Go SDK stands out for:
- Zero dependencies (smallest footprint)
- Strong type safety
- Excellent concurrency model
- Production-ready error handling
- Complete examples

## Future Considerations

Based on the codebase structure, potential future enhancements could include:

1. **Metrics and Observability**: Built-in OpenTelemetry support
2. **Transport Abstraction**: Support for WebSocket, HTTP/2
3. **Middleware Support**: Request/response interceptors
4. **Connection Pooling**: For multi-session scenarios
5. **Protocol Versioning**: Backward compatibility helpers

The current design already supports these extensions through its clean architecture.
