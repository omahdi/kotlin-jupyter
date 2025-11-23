# Jupyter Kernel API Implementation in Kotlin Jupyter

This document provides a comprehensive overview of how the Kotlin Jupyter kernel implements the official Jupyter protocol and what Kotlin-specific extensions it provides.

## Table of Contents

- [Overview](#overview)
- [Jupyter Protocol Reference](#jupyter-protocol-reference)
- [Jupyter API Implementation](#jupyter-api-implementation)
  - [Managing Cells](#a-managing-cells)
  - [Managing Code Execution](#b-managing-code-execution)
  - [Managing State and Context](#c-managing-state-and-context)
  - [Rendering Context](#d-rendering-context)
  - [Communication and Messaging](#e-communication-and-messaging)
  - [Debugging and Control](#f-debugging-and-control)
- [Kotlin-Specific Extensions](#kotlin-specific-extensions)
  - [Library Integration System](#1-library-integration-system)
  - [Magic Commands](#2-magic-commands)
  - [Advanced Rendering](#3-advanced-rendering)
  - [REPL Commands](#4-repl-commands)
  - [Notebook API](#5-notebook-api)
  - [Dependency Management](#6-dependency-management)
  - [Custom Processors](#7-custom-processors)
- [Architecture Overview](#architecture-overview)
- [Key Implementation Files](#key-implementation-files)

## Overview

The Kotlin Jupyter kernel is a fully-featured implementation of the [Jupyter Messaging Protocol](https://jupyter-client.readthedocs.io/en/latest/messaging.html), enabling Kotlin code execution in Jupyter notebooks. It bridges the Jupyter notebook environment with the Kotlin compiler and runtime.

**Protocol Version**: The kernel implements Jupyter Messaging Protocol version 5.3+

## Jupyter Protocol Reference

The official Jupyter protocol documentation is available at:
- **Main Documentation**: [Jupyter Client Messaging](https://jupyter-client.readthedocs.io/en/latest/messaging.html)
- **Kernel Specs**: [Jupyter Kernels](https://docs.jupyter.org/en/latest/projects/kernels.html)

The Jupyter protocol defines a messaging system where:
- Messages are sent over ZeroMQ (ZMQ) sockets
- Messages follow a specific structure with headers, parent headers, metadata, and content
- Communication happens across multiple channels: shell, iopub, stdin, control, and heartbeat

## Jupyter API Implementation

### A. Managing Cells

The kernel implements the following Jupyter APIs for cell management:

#### 1. **Execute Request/Reply** (`execute_request`, `execute_reply`)
- **Location**: `MessageTypes.kt` (lines 46-47), `IdeCompatibleMessageRequestProcessor.kt` (line 145)
- **Purpose**: Execute code in a cell and return results
- **Implementation**: 
  - `processExecuteRequest()` method handles execution
  - Manages execution counters
  - Captures output and errors
  - Supports silent execution, storing history, and user expressions
- **Message Flow**:
  1. Receives `ExecuteRequest` with code
  2. Sends `ExecuteInput` to iopub channel
  3. Executes code via REPL
  4. Sends `ExecuteResult` or `ExecuteErrorReply` based on outcome

#### 2. **Is Complete Request/Reply** (`is_complete_request`, `is_complete_reply`)
- **Location**: `MessageTypes.kt` (lines 62-63), `MessageRequestProcessorImpl.kt` (line 32)
- **Purpose**: Check if code is complete, incomplete, or invalid
- **Implementation**:
  - Checks if input looks like REPL command
  - Uses `repl.checkComplete()` to validate Kotlin syntax
  - Returns status: "complete", "incomplete", or "invalid"

#### 3. **History Request/Reply** (`history_request`, `history_reply`)
- **Location**: `MessageTypes.kt` (lines 93-94), `IdeCompatibleMessageRequestProcessor.kt` (line 191)
- **Purpose**: Retrieve code execution history
- **Implementation**: Currently returns empty list (not fully implemented)

### B. Managing Code Execution

#### 1. **Execute Input** (`execute_input`)
- **Location**: `MessageTypes.kt` (lines 48, 401-407)
- **Purpose**: Broadcast code about to be executed
- **Implementation**: Sent to iopub channel before execution with code and execution count

#### 2. **Execute Result** (`execute_result`)
- **Location**: `MessageTypes.kt` (lines 49, 409-417)
- **Purpose**: Return execution results
- **Implementation**: Contains data, metadata, and execution count

#### 3. **Stream Messages** (`stream`)
- **Location**: `MessageTypes.kt` (lines 79, 385-390)
- **Purpose**: Send stdout/stderr output during execution
- **Implementation**:
  - Captures stdout and stderr using `CapturingOutputStream`
  - Streams are substituted per-thread for proper output capture
  - Sends incremental output to client via iopub

#### 4. **Interrupt Request/Reply** (`interrupt_request`, `interrupt_reply`)
- **Location**: `MessageTypes.kt` (lines 71-72), `IdeCompatibleMessageRequestProcessor.kt` (line 256)
- **Purpose**: Interrupt running code execution
- **Implementation**: `executor.interruptExecution()` stops current execution

#### 5. **Shutdown Request/Reply** (`shutdown_request`, `shutdown_reply`)
- **Location**: `MessageTypes.kt` (lines 68-69), `IdeCompatibleMessageRequestProcessor.kt` (line 239)
- **Purpose**: Shutdown the kernel
- **Implementation**:
  - Executes shutdown callbacks
  - Closes executor
  - Can exit process or interrupt threads based on run mode

### C. Managing State and Context

#### 1. **Kernel Info Request/Reply** (`kernel_info_request`, `kernel_info_reply`)
- **Location**: `MessageTypes.kt` (lines 65-66, 283-310), `IdeCompatibleMessageRequestProcessor.kt` (line 200)
- **Purpose**: Provide kernel metadata and capabilities
- **Implementation**: Returns:
  - Protocol version (5.3+)
  - Implementation: "Kotlin"
  - Kotlin and kernel version
  - Language info (file extension, name, version, MIME type)
  - Current session metadata

#### 2. **Connect Request/Reply** (`connect_request`, `connect_reply`)
- **Location**: `MessageTypes.kt` (lines 96-97, 471-478)
- **Purpose**: Get connection information for kernel sockets
- **Implementation**: Returns port information for shell, iopub, stdin, control, and heartbeat
- **Note**: Deprecated since messaging protocol v5.1

#### 3. **Update Client Metadata** (`update_client_metadata_request`, `update_client_metadata_reply`)
- **Location**: `MessageTypes.kt` (lines 116-117, 313-353)
- **Purpose**: **Custom extension** - Update notebook file path information
- **Implementation**: Updates the notebook's absolute file path (IntelliJ plugin specific)

### D. Rendering Context

#### 1. **Display Data** (`display_data`, `update_display_data`)
- **Location**: `MessageTypes.kt` (lines 81-82, 392-399)
- **Purpose**: Display rich output (HTML, images, plots, etc.)
- **Implementation**:
  - Supports multiple MIME types
  - Can update existing displays using transient IDs
  - Used by rendering system to show rich outputs

#### 2. **Clear Output** (`clear_output`)
- **Location**: `MessageTypes.kt` (lines 86, 425-429)
- **Purpose**: Clear cell output
- **Implementation**: Can wait or clear immediately

#### 3. **Status Messages** (`status`)
- **Location**: `MessageTypes.kt` (lines 84, 418-423)
- **Purpose**: Report kernel execution state
- **Implementation**: Reports states: busy, idle, starting

### E. Communication and Messaging

#### 1. **Comm Messages** (`comm_open`, `comm_msg`, `comm_close`, `comm_info_request`, `comm_info_reply`)
- **Location**: `MessageTypes.kt` (lines 99-104, 480-524)
- **Purpose**: Custom widget communication channel
- **Implementation**:
  - `CommManagerImpl` manages communication channels
  - Used for interactive widgets and custom frontends
  - Supports opening, messaging, and closing comm channels
  - `processCommOpen()`, `processCommMsg()`, `processCommClose()` handle lifecycle

#### 2. **Input Request/Reply** (`input_request`, `input_reply`)
- **Location**: `MessageTypes.kt` (lines 90-91, 435-446)
- **Purpose**: Request user input during execution
- **Implementation**:
  - `StdinInputStream` handles input requests
  - Can request password-protected input
  - Notebook API provides `prompt()` method

#### 3. **Complete Request/Reply** (`complete_request`, `complete_reply`)
- **Location**: `MessageTypes.kt` (lines 59-60, 252-268), `IdeCompatibleMessageRequestProcessor.kt` (line 112)
- **Purpose**: Provide code completion suggestions
- **Implementation**:
  - Uses Kotlin compiler for intelligent completion
  - Returns matches, cursor position, and metadata

#### 4. **Inspect Request/Reply** (`inspect_request`, `inspect_reply`)
- **Location**: `MessageTypes.kt` (lines 56-57, 236-250)
- **Purpose**: Provide introspection/documentation for code elements
- **Implementation**:
  - Returns documentation and type information
  - Supports different detail levels (standard, detailed)

### F. Debugging and Control

#### 1. **Debug Request/Reply/Event** (`debug_request`, `debug_reply`, `debug_event`)
- **Location**: `MessageTypes.kt` (lines 76-78, 88, 379-383, 431-433)
- **Purpose**: Support Jupyter debugging protocol
- **Implementation**: Enables debugging of notebook code

#### 2. **Thread Dump Request** (`thread_dump_request`)
- **Location**: `MessageTypes.kt` (lines 74-376)
- **Purpose**: **Custom extension** - Dump thread information for debugging
- **Implementation**: Writes thread dump to specified file path

#### 3. **List Errors Request/Reply** (`list_errors_request`, `list_errors_reply`)
- **Location**: `MessageTypes.kt` (lines 108-109, 527-537)
- **Purpose**: **Custom extension** - Get compilation errors without execution (Jupyter Web)
- **Implementation**: Returns list of script diagnostics for given code

## Kotlin-Specific Extensions

Beyond the standard Jupyter protocol, the Kotlin kernel provides extensive Kotlin-specific functionality:

### 1. Library Integration System

**Purpose**: Seamlessly integrate JVM libraries into notebooks

**Implementation Files**:
- `LibraryDefinition.kt` - Define library integrations
- `JupyterIntegration.kt` - Integration API
- `LibrariesMagicsHandler.kt` - Handle `%use` magic

**Features**:
- **Dependency Resolution**: Automatically resolve and load Maven dependencies
- **Default Imports**: Inject imports when library is loaded
- **Initialization Code**: Execute setup code on library load
- **Type Renderers**: Register custom renderers for library types
- **Callbacks**: Before/after cell execution, on shutdown, on interrupt

**Usage**: `%use lets-plot, dataframe`

**Key Classes**:
- `LibraryDefinition` - Interface for library descriptors
- `JupyterIntegration` - API for programmatic integration
- `LibraryLoader` - Loads and manages libraries
- `LibraryResolutionRequest` - Tracks library requests

### 2. Magic Commands

**Purpose**: Line-based commands for notebook configuration

**Implementation Files**:
- `MagicsProcessor.kt` - Process magic commands
- `LibrariesMagicsHandler.kt` - Library magics
- `ReplOptionsMagicsHandler.kt` - Output configuration

**Supported Magics**:
- `%use <lib1>, <lib2>` - Load libraries
- `%trackClasspath [on/off]` - Debug classpath changes
- `%trackExecution [all/generated/off]` - Debug code execution
- `%useLatestDescriptors [on/off]` - Use latest library versions
- `%output` - Configure output capturing
- `%logLevel [off/error/warn/info/debug]` - Set logging level

### 3. Advanced Rendering

**Purpose**: Rich output rendering beyond basic display

**Implementation Files**:
- `RenderersProcessor.kt` - Manage value renderers
- `TextRenderersProcessor.kt` - Text-based rendering
- `ThrowableRenderersProcessor.kt` - Exception rendering
- `Display.kt`, `DisplayHandler.kt` - Display system

**Features**:
- **Custom Renderers**: Register renderers for any type
- **MIME Type Support**: HTML, SVG, LaTeX, images, JSON, etc.
- **Display Updates**: Update displays dynamically
- **Animations**: Frame-based animations
- **Graphs**: Graph visualization DSL

**Key APIs**:
- `Renderable` interface - Objects that can render themselves
- `DisplayResult` - Rich display outputs
- `MIME()` function - Create MIME-typed results
- `HTML()` function - Create HTML outputs
- `notebook.renderersProcessor` - Register custom renderers

### 4. REPL Commands

**Purpose**: Interactive commands for notebook exploration

**Implementation Files**:
- `Commands.kt` - Command execution
- `IdeCompatibleMessageRequestProcessor.kt` - Command routing

**Supported Commands**:
- `:help` - Show help information
- `:classpath` - Display current classpath
- `:vars` - Show declared variables

### 5. Notebook API

**Purpose**: Programmatic access to notebook state and capabilities

**Implementation Files**:
- `Notebook.kt` - Main notebook interface
- `KotlinKernelHost.kt` - Kernel host interface
- `CodeCell.kt` - Cell representation

**Key Features**:

#### Notebook Interface
Available via global `notebook` variable:

- **Cell Access**:
  - `cellsList` - All executed cells
  - `getCell(id)` - Get cell by execution number
  - `currentCell` - Currently executing cell
  - `history(n)` - Get cell n steps back

- **State Management**:
  - `variablesState` - Current variable state
  - `cellVariables` - Variables per cell
  - `resultsAccessor` - Access execution results
  - `getResult(id)` - Get result by execution number

- **Display Management**:
  - `displays` - Display container
  - `getAllDisplays()` - All display objects
  - `getDisplaysById(id)` - Displays by ID

- **Library Management**:
  - `libraryLoader` - Load libraries
  - `libraryRequests` - Track library requests
  - `getLibraryFromDescriptor()` - Parse library descriptors

- **Configuration**:
  - `sessionOptions` - Session configuration
  - `kernelVersion` - Current kernel version
  - `kernelRunMode` - Execution environment mode
  - `workingDir` - Notebook directory

#### KotlinKernelHost Interface
Available via `notebook.executionHost`:

- **Display Operations**:
  - `display(value, id)` - Display a value
  - `updateDisplay(value, id)` - Update existing display

- **Execution Control**:
  - `execute(code)` - Execute code immediately
  - `scheduleExecution(code)` - Execute after current cell

- **Library Management**:
  - `addLibrary(library)` - Add library programmatically
  - `loadKotlinArtifacts()` - Load Kotlin standard libraries

- **Variable Management**:
  - `declare(variables)` - Declare global variables

### 6. Dependency Management

**Purpose**: Dynamic dependency resolution and classpath management

**Implementation Files**:
- `DependencyManager.kt` - Manage dependencies
- Annotations: `@file:DependsOn()`, `@file:Repository()`

**Features**:
- Maven dependency resolution
- Local JAR loading
- Custom repository configuration
- Gradle-like syntax via `USE {}` blocks

**Usage**:
```kotlin
// Annotation style
@file:DependsOn("io.ktor:ktor-client-core:2.0.0")
@file:Repository("https://my.repo.com/maven")

// Programmatic style
USE {
    repositories {
        maven("https://my.repo.com/maven")
    }
    dependencies {
        implementation("io.ktor:ktor-client-core:2.0.0")
    }
}
```

### 7. Custom Processors

**Purpose**: Extensible processing pipelines

**Implementation Files**:
- `RenderersProcessor.kt` - Value rendering
- `TextRenderersProcessor.kt` - Text rendering
- `FieldsProcessor.kt` - Field handling
- `CodePreprocessor.kt` - Code transformation
- `ExtensionsProcessor.kt` - Generic extension processing

**Processor Types**:

- **Renderers Processor**: Convert values to displayable formats
- **Text Renderers Processor**: Convert values to text strings
- **Throwable Renderers Processor**: Handle exception rendering
- **Fields Processor**: Process field declarations and updates
- **Code Preprocessors**: Transform code before compilation
- **Execution Callbacks**: Before/after cell execution hooks
- **Interruption Callbacks**: Handle execution interruption
- **Color Scheme Callbacks**: Respond to theme changes

**Access**: Via `notebook` object properties:
- `notebook.renderersProcessor`
- `notebook.textRenderersProcessor`
- `notebook.throwableRenderersProcessor`
- `notebook.fieldsHandlersProcessor`
- `notebook.beforeCellExecutionsProcessor`
- `notebook.afterCellExecutionsProcessor`
- `notebook.shutdownExecutionsProcessor`
- `notebook.codePreprocessorsProcessor`
- `notebook.interruptionCallbacksProcessor`
- `notebook.colorSchemeChangeCallbacksProcessor`

## Architecture Overview

### Module Structure

The kernel is organized into several modules:

1. **`src/main/kotlin`** - Main kernel implementation
   - Message handling (`MessageHandlerImpl.kt`)
   - Message processing (`MessageRequestProcessorImpl.kt`)
   - ZMQ server (`JupyterZmqServerRunner.kt`)

2. **`jupyter-lib/shared-compiler`** - Shared compilation and messaging
   - Message types (`MessageTypes.kt`)
   - Message processing (`AbstractMessageRequestProcessor.kt`)
   - Communication facility (`JupyterCommunicationFacility.kt`)
   - Execution counter (`ExecutionCounter.kt`)

3. **`jupyter-lib/protocol`** - Protocol implementation
   - Socket management (`JupyterSocket.kt`, `JupyterServerSockets.kt`)
   - Message routing
   - Connection handling

4. **`jupyter-lib/zmq-protocol`** - ZeroMQ transport layer
   - ZMQ socket wrappers
   - Message serialization/deserialization

5. **`jupyter-lib/api`** - Public API for extensions
   - `Notebook.kt` - Notebook interface
   - `KotlinKernelHost.kt` - Kernel host interface
   - Rendering APIs
   - Library integration APIs

6. **`jupyter-lib/lib`** - Notebook runtime
   - Standard library extensions
   - Display helpers
   - REPL context

### Message Flow

1. **Client → Kernel**:
   - Client sends message via ZMQ
   - `JupyterZmqSocket` receives raw message
   - `MessageHandler` converts to `Message` object
   - `MessageRequestProcessor` routes to appropriate handler
   - Handler processes and generates response

2. **Kernel → Client**:
   - Handler creates reply message
   - `MessageFactory` constructs message with proper headers
   - `JupyterZmqSocket` sends via appropriate channel (shell/iopub/stdin)

### Execution Flow

1. **Execute Request**:
   - Receive `ExecuteRequest` on shell channel
   - Send `StatusMessage(BUSY)` on iopub
   - Send `ExecuteInput` on iopub
   - Substitute stdout/stderr streams
   - Execute code via REPL
   - Capture and send output streams
   - Render and send result
   - Send `StatusMessage(IDLE)` on iopub
   - Send `ExecuteReply` on shell channel

## Key Implementation Files

### Protocol Implementation
- **`MessageTypes.kt`** - All Jupyter message type definitions
- **`Message.kt`** - Message structure and helpers
- **`AbstractMessageRequestProcessor.kt`** - Message routing logic
- **`IdeCompatibleMessageRequestProcessor.kt`** - Core message handlers
- **`MessageHandlerImpl.kt`** - Top-level message handler

### Execution Engine
- **`ReplForJupyter.kt`** - Main REPL interface
- **`InternalEvaluatorImpl.kt`** - Code evaluation
- **`JupyterExecutor.kt`** - Execution management
- **`ExecutionCounter.kt`** - Execution numbering

### Communication
- **`JupyterSocket.kt`** - Socket abstraction
- **`JupyterServerSockets.kt`** - Socket manager
- **`SocketDisplayHandler.kt`** - Display message sending
- **`CommManagerImpl.kt`** - Comm channel management

### Library System
- **`LibraryDefinition.kt`** - Library definition interface
- **`JupyterIntegration.kt`** - Integration builder
- **`LibraryLoader.kt`** - Library loading
- **`LibrariesMagicsHandler.kt`** - `%use` magic handler

### Rendering
- **`RenderersProcessorImpl.kt`** - Renderer management
- **`Display.kt`** - Display functions
- **`DisplayHandler.kt`** - Display handling

### Extensions
- **`Notebook.kt`** - Notebook API
- **`KotlinKernelHost.kt`** - Kernel host API
- **`CodePreprocessor.kt`** - Code preprocessing
- **`FieldsProcessor.kt`** - Field handling

---

## Summary

The Kotlin Jupyter kernel provides:

1. **Complete Jupyter Protocol Implementation**: All standard message types are supported for managing cells, execution, state, and rendering.

2. **Kotlin-Optimized Extensions**: 
   - Rich library integration system
   - Magic commands for configuration
   - Advanced rendering capabilities
   - Interactive REPL commands
   - Comprehensive notebook API

3. **Extensibility**: Through processors, callbacks, and integration APIs, the kernel can be extended to support new types, libraries, and workflows.

The implementation is modular, well-organized, and provides both protocol compliance and Kotlin-specific enhancements that make notebook development with Kotlin powerful and productive.
