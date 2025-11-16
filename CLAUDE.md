# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Visual Studio Code (VS Code) is a TypeScript-based code editor built on Electron and web technologies. The codebase supports multiple deployment targets: desktop (Electron), web browser, and remote server environments, all from a single source tree.

## Build and Development Commands

### Initial Setup
```bash
npm install                    # Install dependencies
```

### Development
```bash
npm run watch                  # Watch mode for client + extensions
npm run watch-client           # Watch core client code only
npm run watch-extensions       # Watch built-in extensions only
npm run compile                # Full compile (slow, prefer watch)
npm run compile-web            # Compile web version
```

### Running VS Code
```bash
./scripts/code.sh              # Launch VS Code (macOS/Linux)
./scripts/code.bat             # Launch VS Code (Windows)
./scripts/code-web.sh          # Launch web version
./scripts/code-server.sh       # Launch remote server
```

### Testing
```bash
# Unit tests
./scripts/test.sh              # Run all unit tests (macOS/Linux)
./scripts/test.sh --grep <pattern>  # Run specific unit tests
./scripts/test.bat             # Run all unit tests (Windows)

# Integration tests
./scripts/test-integration.sh  # Run integration tests (macOS/Linux)
./scripts/test-integration.bat # Run integration tests (Windows)

# Browser tests
npm run test-browser           # Browser unit tests with Playwright

# Node tests
npm run test-node              # Node.js unit tests with Mocha
```

### Code Quality
```bash
npm run eslint                 # Run ESLint
npm run stylelint              # Run Stylelint
npm run valid-layers-check     # Validate architectural layering
npm run hygiene                # Pre-commit hygiene checks
```

### Other Useful Scripts
```bash
npm run monaco-compile-check   # Check Monaco editor types
npm run vscode-dts-compile-check  # Check VS Code API types
npm run download-builtin-extensions  # Download bundled extensions
```

## Architecture Overview

### Strict Layering System

VS Code enforces a strict architectural layering model to ensure code can run on different platforms (browser, Node.js, Electron, web workers). Layers are validated at compile time via `npm run valid-layers-check`.

**The 6 architectural layers:**

1. **Common** (`common/**/*.ts`)
   - Pure TypeScript, no platform APIs (no DOM, no Node.js)
   - Shared interfaces, utilities, data structures
   - Can only depend on other common code
   - Example: `vs/base/common`, `vs/platform/*/common`

2. **Browser** (`browser/**/*.ts`)
   - Uses DOM APIs, runs in browser/Electron renderer
   - Can depend on: Common, Browser
   - Cannot use Node.js or Electron main process APIs
   - Example: `vs/workbench/browser`, `vs/editor/browser`

3. **Node** (`node/**/*.ts`)
   - Uses Node.js APIs, runs in Node.js processes
   - Can depend on: Common, Node
   - Cannot use DOM or Electron-specific APIs
   - Example: `vs/platform/*/node`

4. **Worker** (`worker/**/*.ts`)
   - Runs in Web Workers (no DOM, no Node.js)
   - Can depend on: Common, Worker
   - Used for: tokenization, language detection, search
   - Example: `vs/editor/common/services/editorWebWorkerMain`

5. **Electron Browser** (`electron-browser/**/*.ts`)
   - Electron renderer process with DOM access
   - Can depend on: Common, Browser, Electron-Browser
   - Electron-specific UI integration

6. **Electron Main/Utility** (`electron-main/**/*.ts`, `electron-utility/**/*.ts`)
   - Electron main/utility processes
   - Can depend on: Common, Node, Electron-Main/Utility
   - Process management, native APIs, file system

**Layer violations will cause build failures.** If you see layer errors, move code to the appropriate layer or justify an exception.

### Source Code Organization: `/src/vs`

```
/src/vs
├── /base           - Foundation utilities (arrays, events, lifecycle, URI)
│   ├── /common     - Platform-agnostic utilities
│   ├── /browser    - DOM utilities, styling
│   ├── /node       - Node.js utilities
│   └── /parts      - Reusable components (tree, quickinput, storage)
│
├── /platform       - ~90 injectable services (dependency injection layer)
│   ├── /instantiation  - IoC container
│   ├── /files          - File system abstraction
│   ├── /configuration  - Settings
│   ├── /workspace      - Workspace/folder management
│   └── ... 85+ more services (keybindings, terminal, debug, etc.)
│
├── /editor         - Monaco editor implementation
│   ├── /common     - Editor core (models, tokenization, selections)
│   ├── /browser    - Editor rendering
│   ├── /contrib    - Editor features (find, format, folding, etc.)
│   └── /standalone - Standalone Monaco bundle
│
├── /workbench      - Application shell and UI
│   ├── /browser    - Workbench UI framework (parts, layout)
│   ├── /services   - Workbench-specific services
│   ├── /contrib    - Major features (git, debug, search, terminal, etc.)
│   ├── /api        - Extension host and VS Code extension API
│   └── /electron-browser - Electron-specific workbench code
│
├── /code           - Application entry points
│   ├── /browser         - Web entry point
│   ├── /electron-main   - Electron main process
│   ├── /electron-browser - Electron renderer
│   └── /node            - Node.js/CLI entry point
│
└── /server         - Remote server implementation
```

### Key Architectural Patterns

**1. Dependency Injection**

All services use constructor-based dependency injection via `@IServiceName` decorators:

```typescript
export class MyService implements IMyService {
  constructor(
    @IConfigurationService private readonly configService: IConfigurationService,
    @IFileService private readonly fileService: IFileService
  ) {}
}
```

Services are registered in the `IInstantiationService` container. Never instantiate services with `new` - always use the instantiation service.

**2. Service Definition Pattern**

Services follow a consistent pattern:
- Interface in `common/` (e.g., `platform/files/common/files.ts`)
- Common implementation in `common/`
- Platform-specific implementations in `browser/`, `node/`, `electron-browser/`, etc.

**3. Contribution Model**

Features register themselves via contribution registries:
```typescript
Registry.as<IConfigurationRegistry>(Extensions.Configuration)
  .registerConfiguration({ ... });
```

This decouples features from the core workbench.

**4. Observable State Management**

Modern VS Code code uses observables for reactive state:
```typescript
import { observableValue, autorun } from 'vs/base/common/observableInternal';

const state$ = observableValue('state', initialValue);
autorun(reader => {
  console.log(state$.read(reader)); // Reactive updates
});
```

## Built-in Extensions

The `/extensions` directory contains ~100 bundled extensions:
- Language support: `typescript-language-features/`, `html-language-features/`, etc.
- Core features: `git/`, `markdown-language-features/`, `emmet/`
- Themes: `theme-*` folders

Each extension follows standard VS Code extension structure with `package.json` and TypeScript sources.

## TypeScript Compilation & Validation

### CRITICAL: Always Check Compilation Status

**Before running tests or declaring work complete:**

1. Monitor the `VS Code - Build` task output for compilation errors
2. This task runs `Core - Build` and `Ext - Build` for incremental compilation
3. Fix ALL compilation errors before proceeding

**NEVER:**
- Run tests with compilation errors
- Use `npm run compile` for incremental builds (use watch tasks instead)
- Skip compilation validation

### Layer Validation

Run `npm run valid-layers-check` to verify architectural layer compliance. This checks that:
- Browser code doesn't import Node.js modules
- Common code doesn't use platform-specific APIs
- Worker code doesn't access DOM
- Layer boundaries are respected

## Coding Guidelines

### Code Style

- **Indentation:** Use tabs, not spaces
- **Naming:**
  - PascalCase for types, enums, enum values, classes
  - camelCase for functions, methods, properties, variables
- **Functions:** Prefer `export function foo() {}` over `export const foo = () => {}` (better stack traces)
- **Async:** Prefer `async`/`await` over `.then()` chains
- **Arrow functions:** Only use parens when necessary:
  ```typescript
  x => x + 1           // ✓
  (x) => x + 1         // ✗
  (x, y) => x + y      // ✓
  <T>(x: T) => x       // ✓
  ```
- **Braces:** Always use braces for loops and conditionals

### Strings and Localization

- Use `'single quotes'` for internal strings
- Use `"double quotes"` for user-facing strings (that need localization)
- User-facing strings must use `nls.localize()` for i18n
- No string concatenation in localized strings - use placeholders: `nls.localize('key', "Hello {0}", name)`

### UI Labels

Use title-case capitalization for commands, buttons, menus (e.g., "Open File in New Window")

### Comments

Use JSDoc-style comments for functions, interfaces, enums, and classes.

### TypeScript

- Avoid `any` or `unknown` types - use proper types or interfaces
- Don't export types/functions unless needed across multiple components
- Don't add to global namespace
- Never duplicate imports - reuse existing imports
- Prefer named regex capture groups over numbered groups

### Copyright Header

All files must include the Microsoft copyright header:
```typescript
/*---------------------------------------------------------------------------------------------
 *  Copyright (c) Microsoft Corporation. All rights reserved.
 *  Licensed under the MIT License. See License.txt in the project root for license information.
 *--------------------------------------------------------------------------------------------*/
```

## Testing Guidelines

- Tests are co-located with source: `src/vs/*/test/`
- Integration tests end with `.integrationTest.ts` or are in `/extensions/`
- Don't add tests to wrong suite (e.g., appending to file instead of relevant suite)
- Look for existing patterns before creating new test structures
- Use `describe` and `test` consistently with existing code

## Finding Code

### Search Strategy

1. **Semantic search:** Use file/symbol search for general concepts
2. **Grep for exact strings:** Error messages, specific function names
3. **Follow imports:** Check what imports the module you're investigating
4. **Check tests:** Often reveal usage patterns and expected behavior

### Common Locations

- **Settings/config:** `vs/platform/configuration/`
- **File operations:** `vs/platform/files/`
- **Editor features:** `vs/editor/contrib/`
- **Git integration:** `extensions/git/`
- **Terminal:** `vs/workbench/contrib/terminal/`
- **Debug:** `vs/workbench/contrib/debug/`
- **Extension API:** `src/vscode.d.ts`, `vs/workbench/api/`

## Common Development Workflows

### Adding a New Service

1. Define interface in `platform/myservice/common/myservice.ts`
2. Implement in platform-specific folder (e.g., `browser/myservice.ts`)
3. Register in instantiation container (typically in main entry points)
4. Use via `@IMyService` constructor injection

### Adding a New Workbench Feature

1. Create in `workbench/contrib/myfeature/`
2. Use platform-specific subdirectories: `common/`, `browser/`, etc.
3. Register contributions via registry (actions, views, configuration, etc.)
4. Import in `workbench/workbench.desktop.main.ts` or appropriate entry point

### Fixing a Bug

1. Search for error message or symptom
2. Add breakpoint or logging
3. Run via `./scripts/code.sh` or launch configuration
4. Fix issue
5. Add/update tests
6. Verify compilation with watch task

## Performance Considerations

- Most modules use dynamic imports for lazy loading
- Heavy computation runs in Web Workers (tokenization, search, language detection)
- Lists and editors use virtual scrolling
- Async operations use Promises/async-await

## Development Environment

### System Requirements

- **RAM:** 8 GB minimum, 9 GB recommended for full builds
- **CPU:** 4 cores minimum
- **Node.js:** Version specified in `.nvmrc`

### Dev Container

A dev container configuration is available in `.devcontainer/`:
- Uses Docker with VNC for GUI testing
- Requires 4 cores, 8 GB RAM minimum
- Access via VNC on port 5901 (password: `vscode`) or web client on port 6080

### VS Code Setup

When developing in VS Code:
1. Use the `VS Code - Build` task to watch compilation
2. Use launch configurations in `.vscode/launch.json` for debugging
3. Monitor task output for TypeScript errors

---

# FlowLeap Patent IDE — Project Architecture Plan

## Vision

This VS Code fork will become **FlowLeap Patent IDE**, a desktop application designed specifically for patent examination workflows, powered by:

* **Local AI agents** (Reasoning Agent + Coding Agent)
* **File-based project memory**
* **Terminal access & script execution**
* **OPS, USPTO, WIPO patent APIs**
* **MCP server integration**
* **Vercel AI SDK v5** as the backend

This environment becomes an **AI-first Patent IDE** — faster, more flexible, and more powerful than any web-based patent analysis tool.

## Why Desktop (VS Code fork) > Web Apps

Local desktop environment provides:

* Full filesystem access
* Terminal execution (curl, scripts)
* Fast cache & memory
* Unlimited project file size
* Instant creation of scripts/tools
* Persistent multi-day workflows
* Local embeddings
* Offline mode
* True autonomy

The desktop approach has been validated: Cursor AI executing curl + writing XML → MD was **faster and more capable** than all web apps for patent analysis.

## Core Architecture

### Top-Level Layers

1. **UI Layer** — VS Code fork (FlowLeap branded)
2. **Extension Layer** — Built-in FlowLeap agent extension
3. **Agent Core** — Node.js backend (AI SDK v5)
4. **Tools Layer** — OPS, vector DB, scripts, CQL, etc.
5. **MCP Layer** — External tools (web search, Python, DB)
6. **Storage Layer** — Project folder + local vector DB

### Multi-Agent System

#### Reasoning Agent

* Handles prior art search
* Writes search strategy
* Runs novelty analysis
* Uses tools
* Reads/writes memory
* Delegates to coding agent when needed

#### Coding Agent (CodeAct)

* Generates scripts (Node, Python, Bash)
* Executes scripts locally
* Fetches patent data
* Cleans XML
* Builds vectors
* Computes similarity
* Produces artifacts

This architecture mirrors Cursor, Devin, and other AI coding assistants.

## Patent Project Folder Structure

Local folder = long-term memory for each patent analysis project.

```
PatentProject_EP1234567/
│
├── claims.md
├── description.md
├── search-strategy.md
├── cql/
│   ├── initial.txt
│   ├── refined.txt
│
├── prior-art/
│   ├── EPxxxxxx.xml
│   ├── USxxxxxx.md
│
├── analysis/
│   ├── novelty-matrix.json
│   ├── novelty-report.md
│   ├── feature-extraction.json
│
├── scripts/
│   ├── fetch_ops.js
│   ├── clean_xml.py
│
├── outputs/
│   ├── ranked-prior-art.md
│   ├── heatmap.png
│
└── .flowleap/
     ├── memory.json
     ├── embeddings.index
     ├── task-history.json
```

The agent can rehydrate context instantly from this structure. This is the ultimate advantage over web SaaS tools.

## First Milestone: MVP Small Win

The initial implementation steps:

1. **Fork VS Code (Code-OSS)** ✓
2. **Build it using npm**
   ```bash
   npm install
   npm run compile
   ./scripts/code.sh
   ```
3. **Rebrand it** by editing [product.json](product.json)
4. **Replace icons** in `/resources/darwin`, `/resources/win32`, `/resources/linux`
5. **Build macOS app**
   ```bash
   npm run gulp vscode-darwin-arm64
   ```
6. **Add a simple built-in extension**
   * Command: "FlowLeap Chat"
   * Simple notification or chat panel

This delivers:

✓ FlowLeap Patent IDE.app
✓ Custom branding
✓ Pre-installed extension
✓ Working build pipeline
✓ Foundation ready for agent system

## Technology Stack

* **VS Code fork** → UI shell
* **Node.js in Extension Host** → Agent runtime
* **Vercel AI SDK v5** → LLM + tool calling
* **Terminal Access** → curl / scripts
* **MCP protocol** → External tools
* **Local vector DB** → Embeddings (e.g., LanceDB, ChromaDB)
* **FlowLeap Project Format** → Persistence

## Tools to Build

### Patent Data Tools

* `opsFetchClaims` - Fetch claims from OPS API
* `opsFetchBiblio` - Fetch bibliographic data
* `opsFetchFulltext` - Fetch full text
* `fetchCitations` - Fetch citation network
* `fetchFamily` - Fetch patent family
* `fetchClassification` - Fetch IPC/CPC classifications

### Analysis Tools

* `buildNoveltyMatrix` - Compare claims vs prior art
* `rankPriorArt` - Rank documents by relevance
* `extractFeatures` - Extract technical features from claims
* `mapClaimsToPriorArt` - Map claim elements to prior art passages

### Code Execution Tools

* `createScriptFile()` - Generate script in project folder
* `runScriptInSandbox()` - Execute with proper isolation
* `readScriptOutput()` - Return results to agent

### Memory Tools

* `projectMemoryRead()` - Read from `.flowleap/memory.json`
* `projectMemoryWrite()` - Persist agent state

## Why This Architecture Wins

* **Local-first** - No cloud dependency for core workflows
* **Agent-native** - Built for AI from the ground up
* **Script automation** - Generate and execute analysis scripts
* **Multi-step reasoning** - Complex patent analysis workflows
* **Durable project memory** - Resume work across sessions
* **Instant data retrieval** - No API rate limits or timeouts
* **High autonomy** - Agents can explore and iterate
* **Attorney-grade workflows** - Professional patent examination tools
* **Enterprise privacy** - All data stays local

This architecture **outperforms every patent tool available today**, including EPO examiner tools.

## Implementation Locations in VS Code Codebase

Based on VS Code architecture, FlowLeap components will be located:

### FlowLeap Agent Extension
* **Location:** `/extensions/flowleap/`
* **Structure:**
  ```
  /extensions/flowleap/
  ├── package.json           - Extension manifest
  ├── src/
  │   ├── extension.ts      - Extension entry point
  │   ├── agents/
  │   │   ├── reasoningAgent.ts
  │   │   └── codingAgent.ts
  │   ├── tools/
  │   │   ├── opsTools.ts
  │   │   ├── analysisTools.ts
  │   │   └── scriptTools.ts
  │   ├── services/
  │   │   ├── aiService.ts  - AI SDK v5 integration
  │   │   ├── mcpService.ts - MCP protocol client
  │   │   └── projectService.ts - Project memory
  │   └── ui/
  │       ├── chatPanel.ts
  │       └── analysisView.ts
  └── resources/
      └── icons/
  ```

### Branding Changes
* **Product configuration:** [product.json](product.json)
  * Change `nameShort`, `nameLong`, `applicationName`
  * Update `dataFolderName` to `flowleap`
  * Customize `extensionsGallery` if using custom marketplace
* **Icons:** `/resources/darwin/`, `/resources/win32/`, `/resources/linux/`
* **Welcome page:** `src/vs/workbench/contrib/welcome/`

### Custom Workbench Contributions
* **Patent project view:** `src/vs/workbench/contrib/flowleap/`
  * Custom sidebar for patent projects
  * Prior art comparison view
  * Novelty analysis dashboard

### Service Integration
* **AI Agent Service:** `src/vs/platform/ai/` (new)
  * Platform service for agent orchestration
  * Follows standard service pattern with common/browser/node layers
* **Patent Project Service:** `src/vs/platform/patentProject/` (new)
  * Manage patent project folder structure
  * Memory persistence
  * Project metadata

## Current Status

* ✓ VS Code forked
* ✓ Validated that Cursor AI works excellently for OPS curl, XML parsing, CQL queries
* Next: Branding + minimal chat extension

## Next Actions

1. ✓ Finish building branded VS Code fork
2. Build macOS application package
3. Add minimal "FlowLeap Chat" built-in extension
4. Integrate Vercel AI SDK v5 in extension
5. Implement first tool: `opsFetchClaims`
6. Add project folder scaffolding
7. Build reasoning agent loop

## Build Commands for FlowLeap

```bash
# Development
npm run watch                  # Watch mode during development

# Building the branded app
npm run gulp vscode-darwin-arm64      # macOS Apple Silicon
npm run gulp vscode-darwin-x64        # macOS Intel
npm run gulp vscode-linux-x64         # Linux
npm run gulp vscode-win32-x64         # Windows

# The built app will be in:
# ../VSCode-darwin-arm64/FlowLeap.app (or similar)
```
