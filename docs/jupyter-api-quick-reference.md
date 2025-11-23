# Kotlin Jupyter Kernel: Quick Reference Guide

This is a quick reference guide for understanding the Kotlin Jupyter kernel implementation and its features.

## Related Documentation

- **[Jupyter API Implementation](jupyter-api-implementation.md)** - Comprehensive overview of protocol implementation and architecture
- **[Jupyter API Categorization](jupyter-api-categorization.md)** - Detailed categorization of all APIs by functionality
- **[Library Integration](libraries.md)** - Guide for integrating new libraries
- **[Line Magics](magics.md)** - Documentation for magic commands
- **[Main README](README.md)** - Getting started and installation guide

## At a Glance

### What is the Kotlin Jupyter Kernel?

A complete implementation of the Jupyter Messaging Protocol that enables Kotlin code execution in Jupyter notebooks. It acts as a bridge between:
- Jupyter notebook clients (JupyterLab, Jupyter Notebook, Kotlin Notebook plugin)
- The Kotlin compiler and runtime

### Protocol Coverage

- ✅ **Jupyter Protocol Version**: 5.3+
- ✅ **42 Message Types** implemented (3 custom extensions)
- ✅ **5 Communication Channels**: Shell, IOPub, Stdin, Control, Heartbeat
- ✅ **100% Coverage** of standard Jupyter protocol

### Key Files

| File | Purpose |
|------|---------|
| `MessageTypes.kt` | All message type definitions (42 types) |
| `Message.kt` | Message structure and serialization |
| `AbstractMessageRequestProcessor.kt` | Message routing |
| `IdeCompatibleMessageRequestProcessor.kt` | Core message handlers |
| `MessageHandlerImpl.kt` | Top-level message dispatcher |
| `ReplForJupyter.kt` | Main REPL interface |
| `Notebook.kt` | Notebook API interface |
| `KotlinKernelHost.kt` | Kernel host API |

## Jupyter APIs by Category

### Cell Management
- `execute_request/reply` - Execute code
- `is_complete_request/reply` - Check code completeness
- `history_request/reply` - Execution history

### Code Execution
- `execute_input` - Broadcast code
- `execute_result` - Return results
- `stream` - stdout/stderr output
- `interrupt_request/reply` - Interrupt execution
- `shutdown_request/reply` - Shutdown kernel

### State & Context
- `kernel_info_request/reply` - Kernel metadata
- `connect_request/reply` - Connection info (deprecated)

### Rendering
- `display_data` - Rich content display
- `update_display_data` - Update displays
- `clear_output` - Clear output
- `status` - Kernel state (busy/idle/starting)

### Communication
- `comm_open/msg/close` - Widget communication
- `comm_info_request/reply` - Comm information
- `input_request/reply` - User input
- `complete_request/reply` - Code completion
- `inspect_request/reply` - Introspection

### Debugging
- `debug_request/reply/event` - Debugging protocol

## Kotlin Extensions

### Custom Message Types (3)

1. **update_client_metadata** - Notebook file path (IntelliJ plugin)
2. **thread_dump_request** - Thread debugging
3. **list_errors** - Get errors without execution (Jupyter Web)

### Magic Commands (6)

| Magic | Purpose |
|-------|---------|
| `%use <libs>` | Load library integrations |
| `%output [options]` | Configure output capturing |
| `%trackClasspath` | Debug classpath |
| `%trackExecution` | Debug execution |
| `%useLatestDescriptors` | Use latest library versions |
| `%logLevel <level>` | Set logging level |

### REPL Commands (3)

| Command | Purpose |
|---------|---------|
| `:help` | Show help |
| `:classpath` | Display classpath |
| `:vars` | Show variables |

### Library Integration

- **80+ Supported Libraries** (lets-plot, dataframe, ktor, etc.)
- **JSON Descriptors** - Easy library integration
- **JupyterIntegration API** - Programmatic integration
- **`USE {}` DSL** - Gradle-like dependency syntax
- **Annotations** - `@file:DependsOn()`, `@file:Repository()`

### Notebook API Highlights

Available via global `notebook` variable:

```kotlin
// Cell access
notebook.currentCell
notebook.cellsList
notebook.getCell(id)

// Results
notebook.getResult(id)
notebook.resultsAccessor

// Variables
notebook.variablesState
notebook.cellVariables

// Display
notebook.displays
notebook.getAllDisplays()

// Libraries
notebook.libraryLoader
notebook.libraryRequests

// Execution
notebook.executionHost?.execute(code)
notebook.executionHost?.display(value)
```

### Rendering System

- **Multiple MIME types** - HTML, SVG, LaTeX, PNG, JPEG, JSON, Markdown
- **Custom renderers** - `notebook.renderersProcessor`
- **Text renderers** - `notebook.textRenderersProcessor`
- **Throwable renderers** - Exception formatting
- **Display API** - `MIME()`, `HTML()` helper functions
- **Graphs** - Graph visualization DSL
- **Animations** - Frame-based animations

### Processors (10 types)

1. **RenderersProcessor** - Value rendering
2. **TextRenderersProcessor** - Text conversion
3. **ThrowableRenderersProcessor** - Exception handling
4. **FieldsProcessor** - Field declarations
5. **CodePreprocessor** - Code transformation
6. **BeforeCellExecutions** - Pre-execution hooks
7. **AfterCellExecutions** - Post-execution hooks
8. **ShutdownExecutions** - Shutdown hooks
9. **InterruptionCallbacks** - Interrupt handlers
10. **ColorSchemeCallbacks** - Theme changes

## Architecture

### Message Flow

```
Client → ZMQ Socket → JupyterZmqSocket → RawMessage
    → MessageHandler → Message → MessageRequestProcessor
    → Specific Handler → REPL Execution → Response
    → MessageFactory → Message → ZMQ Socket → Client
```

### Execution Flow

```
ExecuteRequest → Send Status(BUSY) → Send ExecuteInput
    → Substitute Streams → Execute via REPL
    → Capture Output → Render Result
    → Send Status(IDLE) → Send ExecuteReply
```

### Module Organization

- **src/main** - Main kernel implementation
- **jupyter-lib/shared-compiler** - Messaging and execution
- **jupyter-lib/protocol** - Protocol implementation
- **jupyter-lib/zmq-protocol** - ZeroMQ transport
- **jupyter-lib/api** - Public extension API
- **jupyter-lib/lib** - Notebook runtime

## Use Cases

### 1. Execute Kotlin Code
Just type Kotlin code in a cell and run it - standard Jupyter behavior.

### 2. Add Dependencies
```kotlin
@file:DependsOn("io.ktor:ktor-client-core:2.0.0")
```

### 3. Load Libraries
```kotlin
%use lets-plot, dataframe
```

### 4. Display Rich Content
```kotlin
HTML("<h1>Hello from Kotlin!</h1>")
```

### 5. Access Previous Results
```kotlin
val previousResult = notebook.getResult(1)
```

### 6. Custom Rendering
```kotlin
notebook.renderersProcessor.register<MyType> { value ->
    HTML("<div>${value.customRender()}</div>")
}
```

### 7. Programmatic Execution
```kotlin
notebook.executionHost?.scheduleExecution("println(\"Delayed execution\")")
```

## Comparison with Other Kernels

| Feature | Kotlin Jupyter | IPython | IJava |
|---------|---------------|---------|-------|
| Protocol Version | 5.3+ | 5.3+ | 5.3+ |
| Language | Kotlin | Python | Java |
| Library Integration | ✅ Rich system | Native pip | Basic |
| Magic Commands | ✅ 6 types | ✅ Extensive | Limited |
| Custom Renderers | ✅ Yes | ✅ Yes | Limited |
| Dependency Management | ✅ Maven + Annotations | pip/conda | Maven |
| IDE Integration | ✅ IntelliJ plugin | Jupyter Lab | Limited |
| Graph Visualization | ✅ DSL | matplotlib | JavaPlot |

## Development

### Adding a New Message Type

1. Add to `MessageType` enum in `MessageTypes.kt`
2. Define content class (e.g., `MyRequest`, `MyReply`)
3. Add handler in `AbstractMessageRequestProcessor`
4. Implement in `IdeCompatibleMessageRequestProcessor`

### Adding a Library Integration

1. Create JSON descriptor in kotlin-jupyter-libraries repo
2. Or implement `JupyterIntegration` in your library
3. Add `META-INF/kotlin-jupyter-libraries/libraries.json`
4. Users load with `%use yourlib`

### Adding a Magic Command

1. Create handler implementing `MagicsHandler`
2. Register in `CompositeMagicsHandler`
3. Define parsing logic in `Parsing.kt`

## Performance Characteristics

- **Startup Time**: ~2-3 seconds (JVM + Kotlin compiler initialization)
- **Execution Time**: Near-native Kotlin (compiled code)
- **Memory**: ~100-200 MB base + user code memory
- **Concurrency**: Single-threaded execution per cell (can schedule async work)

## Limitations

- **History API**: Not fully implemented (returns empty)
- **Connect API**: Deprecated in Jupyter protocol v5.1
- **Thread Safety**: Execution is single-threaded per notebook
- **Breakpoints**: Debug support is basic (not full DAP implementation)

## Resources

- **Official Jupyter Protocol**: https://jupyter-client.readthedocs.io/en/latest/messaging.html
- **Kotlin Jupyter GitHub**: https://github.com/Kotlin/kotlin-jupyter
- **Library Repository**: https://github.com/Kotlin/kotlin-jupyter-libraries
- **IntelliJ Plugin**: https://plugins.jetbrains.com/plugin/16340-kotlin-notebook
- **API Documentation**: https://ileasile.github.io/kotlin-jupyter-docs

## Getting Help

- **Issues**: https://github.com/Kotlin/kotlin-jupyter/issues
- **Kotlin Slack**: #kotlin-jupyter channel
- **Stack Overflow**: Tag `kotlin-jupyter`

---

**Last Updated**: 2025-11-23  
**Kernel Version**: Supports Kotlin 2.2.20  
**Protocol Version**: Jupyter Messaging Protocol 5.3+
