# Copilot Tasks — Complete Architectural Audit

> **Audit Date:** February 12, 2026
> **Repository:** `copilot-tasks`  
> **Scope:** Full architectural, technical, and strategic audit  
> **Auditor:** Abraham Imani Bahati (and AI-assisted technical review )
> **License context:** Open-source friendly  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Phase 1 — Full Repository Analysis](#2-phase-1--full-repository-analysis)
3. [Phase 2 — Feature Gap Identification](#3-phase-2--feature-gap-identification)
4. [Phase 3 — Advanced System Improvements](#4-phase-3--advanced-system-improvements)
5. [Phase 4 — Documentation Review](#5-phase-4--documentation-review)
6. [Phase 5 — Pull Request Proposals](#6-phase-5--pull-request-proposals)
7. [Phase 6 — Code Improvements](#7-phase-6--code-improvements)
8. [Phase 7 — Summary & Recommendations](#8-phase-7--summary--recommendations)
9. [Open Questions for Maintainer](#open-questions-for-maintainer)

---

## 1. Executive Summary

**Copilot Tasks** is an Electron-based desktop tray application that bridges GitHub Copilot's SDK with real-time voice calling (via Eleven Labs Conversational AI) and an MCP Server for CLI integration. The application enables users to manage Copilot coding sessions through a dashboard, chat with Copilot agents via a rich text UI, and accept incoming voice calls that create an interactive voice-driven development assistant experience.

### Key Strengths
- **Innovative concept**: Merging voice interaction with AI-assisted development is forward-thinking.
- **Clean UI/UX**: GitHub-flavored dark theme is polished and consistent across all renderer views.
- **MCP integration**: Exposing voice calls as an MCP tool enables composability with other AI workflows.
- **SDK integration**: Demonstrates practical usage of `@github/copilot-sdk` for session management.
- **Simulated call UIs**: Platform-specific mock call windows (Teams, Slack, FaceTime, Agent) show creative demo capability.

### Key Concerns
- **Monolithic main process**: `main.js` (823 lines) handles routing, IPC, HTTP, windows, and business logic.
- **No test suite**: Zero automated tests across the entire codebase.
- **macOS-only process discovery**: `discoverRunningProcesses()` relies on `ps`, `lsof`, and `osascript`.
- **No authentication**: HTTP API on port 19741 is unauthenticated.
- **Single concurrent call**: Only one `activeWebSocket` is supported by `elevenLabs.js`.
- **Unused dependencies**: React and Fluent UI are in `package.json` but all renderers use vanilla HTML/JS.
- **In-memory state**: Most state lives in Maps and arrays; only call history persists via SQLite.
- **No logging framework**: Only raw `console.log` / `console.error` statements.

---

## 2. Phase 1 — Full Repository Analysis

### 2.1 High-Level Architecture

```
┌──────────────────────────────────────────────────┐
│                   Electron App                    │
│  ┌──────────────────────────────────────────────┐│
│  │              Main Process (main.js)          ││
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐ ││
│  │  │  Tray    │ │ Express  │ │  IPC Handlers │ ││
│  │  │  Manager │ │ HTTP API │ │  (accept/deny │ ││
│  │  │          │ │ :19741   │ │   end/chat)   │ ││
│  │  └──────────┘ └──────────┘ └──────────────┘ ││
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐ ││
│  │  │ Copilot  │ │ Eleven   │ │ Call History  │ ││
│  │  │ SDK      │ │ Labs WS  │ │ (SQLite)     │ ││
│  │  └──────────┘ └──────────┘ └──────────────┘ ││
│  └──────────────────────────────────────────────┘│
│  ┌──────────────────────────────────────────────┐│
│  │           Renderer Processes                  ││
│  │  ┌────────┐ ┌──────────┐ ┌────┐ ┌────────┐  ││
│  │  │ call   │ │dashboard │ │chat│ │history │  ││
│  │  │ .html  │ │ .html    │ │.html│ │ .html  │  ││
│  │  └────────┘ └──────────┘ └────┘ └────────┘  ││
│  └──────────────────────────────────────────────┘│
└──────────────────────────────────────────────────┘
        │                        │
        ▼                        ▼
  ┌──────────┐           ┌──────────────┐
  │ MCP      │           │ Copilot CLI  │
  │ Server   │           │ Hooks        │
  │ (stdio)  │           │ (preToolUse) │
  └──────────┘           └──────────────┘
```

### 2.2 File-by-File Analysis

#### `package.json` 
- **Type**: `"commonjs"` — while the SDK (`@github/copilot-sdk`) is ESM-only, requiring dynamic `import()`.
- **Key dependencies**: `@github/copilot-sdk@^0.1.9-preview.0`, `better-sqlite3`, `express`, `ws`, `dotenv`, `uuid`.
- **Unused dependencies**: `react`, `react-dom`, `@fluentui/react-components` — none of the renderers use React.
- **Build**: Vite + electron-builder targeting macOS (DMG) and Windows (NSIS).
- **Scripts**: `dev`, `build`, `test:call`, `build:mac`, `build:installer`.

#### `src/main/main.js`
The monolithic heart of the application. Responsibilities include:
- **Tray creation and menu management** with dynamic context menu.
- **Window management**: Call window (420×500, always-on-top), dashboard (420×520), chat (480×640), history windows.
- **State management**: `pendingCalls` Map, `callQueue` array, `activeCall` variable, `callHistory` array.
- **IPC handlers**: 15+ handlers for accept-call, deny-call, end-call, get-history, get-queue, chat messaging, session jumping, terminal opening, and more.
- **Express HTTP API**: 4 endpoints on `127.0.0.1:19741` — `/api/status`, `/api/call`, `/api/queue`, `/api/history`.
- **Call lifecycle**: Incoming call → queue if busy → show call window → accept/deny → voice session → complete → save to history.

**Technical debt**: This file violates single-responsibility heavily. It should be decomposed into separate modules for window management, IPC routing, HTTP API, call management, and tray logic.

#### `src/main/copilotSdk.js`
SDK integration layer providing:
- **`loadSDK()`**: Dynamic ESM import via `import()` from CommonJS context.
- **`getClient(cwd)`**: Per-workspace client caching in `clientsByWorkspace` Map.
- **`listSessions()`**: With 5-second cache TTL for performance.
- **Session operations**: `resumeSession()`, `createSession()`, `createTaskSession()`, `joinSession()`, `sendToSession()`, `getSessionMessages()`, `leaveSession()`.
- **`discoverRunningProcesses()`**: Uses `ps aux`, `lsof -i`, and `osascript` — **macOS-only**.
- **`focusConduitWindow()`**: AppleScript-based — **macOS-only**.
- **`listSessionsWithRunning()`**: Merges SDK sessions with discovered running processes.
- **Event streaming**: `resumeSession()` and `joinSession()` use async iterators on `session.events`.

**Notable patterns**: The `loadSDK()` function caches the SDK module. Session event handlers are stored in `sessionEventHandlers` Map with cleanup on leave. Error handling is generally try/catch with console.error logging.

#### `src/main/elevenLabs.js`
Eleven Labs Conversational AI integration:
- **WebSocket protocol**: Connects to `wss://api.elevenlabs.io/v1/convai/conversation`.
- **Single active session**: Only one `activeWebSocket` — no concurrent call support.
- **Audio format**: PCM 16-bit, 16kHz, mono — matching the renderer's microphone capture.
- **Protocol messages**: Handles `conversation_initiation_client_data`, `agent_response`, `user_transcript`, `audio`, `ping/pong`, `conversation_ended`.
- **`extractSummary()`**: Parses the conversation summary from the last agent response.
- **`registerVoiceHandlers()`**: Sets up IPC handlers for `voice-audio` and `voice-text` messages.

**Concerns**: No reconnection logic; no backpressure handling for audio chunks; hardcoded API URL.

#### `src/main/callHistory.js` (one class)
SQLite-based persistence via `better-sqlite3`:
- **Schema**: `calls` table with `id`, `call_id`, `topic`, `context`, `questions`, `status`, `result`, timestamps, and indexes.
- **Methods**: `initialize()`, `saveCall()`, `getHistory(limit=50)`, `getCall(callId)`, `clearHistory()`, `close()`.
- **Data model**: `questions` and `result` are JSON-serialized strings.

**Well-designed** — clean separation, proper indexing, and reasonable defaults.

#### `src/main/preload.js`
Main preload script for call windows. Exposes:
- `copilotTasks.getCallData()`, `acceptCall()`, `denyCall()`, `endCall()`, `sendVoiceAudio()`, `sendTextMessage()`.
- Event listeners: `onCallData()`, `onCallAccepted()`, `onVoiceAudio()`, `onVoiceTranscript()`, `onVoiceComplete()`.

#### `src/main/preload-dashboard.js`
Dashboard preload exposing:
- `dashboard.getData()`, `jumpIntoSession()`, `openInTerminal()`, `newChatRequest()`, `newVoiceRequest()`.

#### `src/main/preload-chat.js`
Chat preload exposing:
- `chat.getSessionInfo()`, `getMessages()`, `sendMessage()`, `goBack()`, `pickFolder()`, `onNewMessages()`, `onTyping()`, `onToolExecution()`.

#### `src/main/preload-sim.js`
Simulation preload for demo call windows:
- `electronAPI.acceptSimulatedCall()`, `declineSimulatedCall()`.

#### `src/renderer/call.html` 
The core call UI with two views:
- **Incoming call view**: Shows caller name, topic, context, questions with accept/deny buttons.
- **Active call view**: Duration timer, transcript display, microphone capture (ScriptProcessor, 16kHz, PCM16 → base64), audio playback queue, text input toggle.

**Audio pipeline**: `getUserMedia()` → `AudioContext(16kHz)` → `ScriptProcessor(4096)` → float32→int16→base64 → IPC. Playback reverses the process with a queue-based approach.

#### `src/renderer/dashboard.html` 
Session dashboard with:
- **Call queue section** with badge counter and queue card rendering.
- **Sessions section** showing SDK sessions merged with running terminal processes.
- **Split button**: New chat request (primary) + voice request (dropdown).
- **Auto-refresh**: `setInterval(loadData, 5000)`.
- Session rows with action buttons for chat, terminal, and focus.

#### `src/renderer/chat.html` 
Full chat interface with:
- **Message rendering**: User and assistant message bubbles with timestamps.
- **Tool execution display**: Shows running/completed tool executions inline.
- **Typing indicator**: Pulsing dots animation.
- **Workspace picker**: CWD chip with folder selection.
- **Auto-scroll**: Scrolls to bottom on new messages.

#### `src/renderer/history.html`
Call history viewer with:
- **Tab switching**: History and Queue tabs.
- **History cards**: Topic, status badge, date, duration, context.
- **Queue cards**: Topic with waiting indicator.
- **Auto-refresh**: Periodically reloads data.

#### `src/renderer/call-agent.html`, `call-facetime.html`, `call-slack.html`, `call-teams.html`
Simulated platform-specific incoming call UIs for demos. Each mimics the visual style of its respective platform (Teams purple gradient, Slack dark theme, FaceTime Apple style, Agent GitHub style). All support keyboard shortcuts (A/Enter to accept, D/Escape to decline).

#### `mcp-server.js`
MCP Server implementation:
- **Protocol**: JSON-RPC 2.0 over stdin/stdout, protocol version `2024-11-05`.
- **Capabilities**: Single tool `voice_call` with `topic` (required), `context`, `questions` parameters.
- **Connection**: POSTs to `http://127.0.0.1:19741/api/call` with 3-second timeout.
- **Error handling**: Fast-fail on connection errors with helpful message about desktop app status.

#### `hooks/voice-call.js` + `hooks/voice-call-hook.json`
Copilot CLI hook:
- **Trigger**: `preToolUse` on `ask_user` tool.
- **Behavior**: Intercepts user questions and routes them through the voice call API.
- **Known limitation**: Returns answer via `permissionDecision: 'deny'` with reason, which agents may misinterpret as denial rather than an answer.

#### `spawn-calls.js`
Demo script that spawns 4 simulated call windows (Teams, Slack, FaceTime, Agent) in sequence with 3-second intervals. Used for demonstration purposes.

#### `test-call.js`
Test CLI that simulates a voice call by:
1. Checking server status.
2. Submitting a call via HTTP API.
3. Polling for result every 2 seconds (30-second timeout).
4. Displaying the result.

#### `src/shared/types.ts`
TypeScript type definitions: `CallRequest`, `CallStatus`, `CallResult`, `TranscriptEntry`, `CallRecord`, `API_PORT`, `API_ENDPOINTS`. Provides the data model contract, though **not compiled or enforced at runtime** since the project is CommonJS JavaScript.

#### `build-resources/entitlements.mac.plist`
macOS entitlements for:
- `com.apple.security.device.audio-input` — microphone access.
- `com.apple.security.personal-information.microphone` — personal microphone data.

### 2.3 Orchestration & Task Lifecycle

```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│  MCP Server  │     │  HTTP API   │     │  CLI Hook    │
│  (stdio)     │────▶│  /api/call  │◀────│ (preToolUse) │
└─────────────┘     └──────┬──────┘     └──────────────┘
                           │
                    ┌──────▼──────┐
                    │ processCall │
                    │ (main.js)   │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
        ┌──────────┐           ┌──────────────┐
        │ Active?  │──No──────▶│ Show Call    │
        │ (busy)   │           │ Window       │
        └────┬─────┘           └──────┬───────┘
             │Yes                     │
             ▼                        ▼
        ┌──────────┐          ┌──────────────┐
        │ Add to   │          │ User accepts │
        │ Queue    │          │ or denies    │
        └──────────┘          └──────┬───────┘
                                     │Accept
                              ┌──────▼───────┐
                              │ Start Voice  │
                              │ Session (WS) │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │ Save to      │
                              │ Call History  │
                              └──────────────┘
```

### 2.4 State Management

| State | Storage | Scope | Persistence |
|-------|---------|-------|-------------|
| `pendingCalls` | `Map<callId, {resolve, reject, ...}>` | Main process | Memory only |
| `callQueue` | `Array` | Main process | Memory only |
| `activeCall` | Single variable | Main process | Memory only |
| `callHistory` | `Array` + SQLite DB | Main process | SQLite via `callHistory.js` |
| `clientsByWorkspace` | `Map` | `copilotSdk.js` | Memory only |
| `cachedSessions` | `Array` (5s TTL) | `copilotSdk.js` | Memory only |
| `activeSessions` | `Map` | `copilotSdk.js` | Memory only |
| `activeWebSocket` | Single variable | `elevenLabs.js` | Memory only |

### 2.5 Error Handling Assessment

| Component | Strategy | Rating |
|-----------|----------|--------|
| `main.js` | try/catch with `console.error` | ⚠️ Basic |
| `copilotSdk.js` | try/catch with fallback returns | ⚠️ Basic |
| `elevenLabs.js` | WebSocket error/close events | ⚠️ No retry |
| `mcp-server.js` | JSON-RPC error responses | ✅ Good |
| `callHistory.js` | try/catch with `console.error` | ⚠️ Basic |
| Renderers | `try/catch` + `console.error` | ⚠️ Basic |

**Pattern**: Errors are caught but only logged to console. No structured logging, no error reporting, no user-facing error recovery.

### 2.6 Security Assessment

| Issue | Severity | Location |
|-------|----------|----------|
| Unauthenticated HTTP API | **High** | `main.js` Express server |
| No CORS restrictions | Medium | Express endpoints |
| Hardcoded localhost binding | Low | `127.0.0.1:19741` mitigates remote access |
| API key in environment | Low | `ELEVENLABS_*` env vars — standard practice |
| No input validation on API | Medium | `/api/call` accepts any payload |
| No rate limiting | Medium | All HTTP endpoints |
| `nodeIntegration: false` | ✅ Good | All BrowserWindows |
| `contextIsolation: true` | ✅ Good | All BrowserWindows |

---

## 3. Phase 2 — Feature Gap Identification

### 3.1 Missing Features

| Feature | Impact | Difficulty |
|---------|--------|------------|
| **Concurrent voice calls** | High — current single-call limit is restrictive | Medium |
| **Call transfer/hold** | Medium — no ability to park or transfer calls | Medium |
| **Authentication for HTTP API** | High — security vulnerability | Low |
| **Windows process discovery** | High — `discoverRunningProcesses()` is macOS-only | Medium |
| **Linux support** | Medium — no build targets or platform logic | Medium |
| **Automated testing** | High — zero test coverage | High |
| **Graceful shutdown** | Medium — no cleanup on app exit for WebSocket/sessions | Low |
| **Settings/preferences UI** | Low — no user configuration interface | Medium |
| **Notification sounds** | Low — call notifications are silent | Low |
| **Call recording/export** | Low — transcripts exist but no export mechanism | Low |

### 3.2 SDK Limitations

- **Preview SDK** (`0.1.9-preview.0`): API surface may change without notice.
- **ESM-only**: Requires dynamic `import()` from CommonJS, adding complexity.
- **No TypeScript types at runtime**: `types.ts` exists but is not compiled or enforced.
- **Session event streaming**: Uses async iterators that block on `for await` — no timeout or cancellation.
- **No SDK-level error codes**: Error handling is string-based message parsing.

### 3.3 Missing Abstractions

1. **No event bus**: Components communicate through direct function calls and IPC. An event-driven architecture would decouple concerns.
2. **No state machine**: Call lifecycle transitions (`pending` → `active` → `completed`) are managed with ad-hoc conditionals.
3. **No service layer**: Business logic is interspersed with presentation and transport logic.
4. **No configuration management**: Settings are scattered across environment variables and hardcoded values.
5. **No middleware pattern**: HTTP and IPC handlers have no middleware for validation, logging, or auth.

### 3.4 Observability Gaps

- **No structured logging**: Only `console.log`/`console.error` with emoji prefixes.
- **No metrics collection**: No performance monitoring or usage analytics.
- **No health checks**: `/api/status` returns basic info but no deep health check.
- **No tracing**: No correlation IDs across call → voice session → SDK session.
- **No crash reporting**: Unhandled errors silently fail.

### 3.5 Concurrency & Coordination Risks

1. **Race condition in call acceptance**: If two calls arrive simultaneously, both could become "active" before the queue logic triggers.
2. **WebSocket lifecycle**: No mutex or lock around `activeWebSocket` — concurrent access could corrupt state.
3. **Session cache staleness**: 5-second TTL cache in `listSessions()` may serve stale data during rapid operations.
4. **IPC handler registration**: Multiple windows could send conflicting IPC messages (e.g., two call windows accepting simultaneously).

---

## 4. Phase 3 — Advanced System Improvements

### 4.1 Supervisor Agent Layer

**Proposal**: Introduce a `TaskSupervisor` class that orchestrates multiple concurrent tasks, manages priorities, and coordinates between voice and text sessions.

```
┌────────────────────────────────┐
│        TaskSupervisor          │
│  ┌──────────┐  ┌───────────┐  │
│  │ Priority  │  │ Lifecycle │  │
│  │ Queue     │  │ Manager   │  │
│  └──────────┘  └───────────┘  │
│  ┌──────────┐  ┌───────────┐  │
│  │ Retry    │  │ State     │  │
│  │ Policy   │  │ Machine   │  │
│  └──────────┘  └───────────┘  │
└────────────────────────────────┘
```

**Benefits**: Centralized control flow, configurable retry policies, priority-based scheduling, and clean state transitions.

### 4.2 Task Persistence & Recovery

**Current state**: All in-memory; app restart loses pending calls and active sessions.

**Proposed approach**:
- Extend SQLite schema with a `tasks` table tracking lifecycle state.
- On startup, recover pending tasks and resume where possible.
- Store session IDs for reconnection after crash/restart.

### 4.3 Scheduling & Retry Policies

**Proposal**: Implement configurable retry with exponential backoff for:
- WebSocket connection failures (Eleven Labs).
- SDK client initialization failures.
- HTTP API connection timeouts (MCP server → desktop app).

```javascript
// Example retry policy configuration
const retryPolicy = {
  maxAttempts: 3,
  baseDelay: 1000,
  maxDelay: 30000,
  backoffMultiplier: 2,
  retryableErrors: ['ECONNREFUSED', 'ETIMEDOUT', 'WebSocket closed']
};
```

### 4.4 Priority Queue System

**Current**: FIFO queue with no priority awareness.

**Proposed**: Weighted priority queue considering:
- Call source (MCP tool vs. CLI hook vs. manual).
- Topic keywords (e.g., "urgent", "security", "production").
- Time in queue (aging-based priority boost).
- User-defined priority labels.

### 4.5 State Machine for Call Lifecycle

**Proposal**: Replace ad-hoc state management with a formal state machine:

```
[idle] ──incoming──▶ [pending]
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
          [accepted]          [denied]
              │                   │
              ▼                   ▼
          [connecting]        [completed]
              │
              ▼
          [active]
              │
        ┌─────┴─────┐
        ▼           ▼
    [ending]    [errored]
        │           │
        ▼           ▼
    [completed] [completed]
```

**Benefits**: Prevents invalid transitions, makes debugging easier, enables state-based UI updates.

### 4.6 Observability & Telemetry

**Proposed stack**:
- **Structured logging**: Replace `console.log` with a logger (e.g., `pino` or `winston`) supporting JSON output, log levels, and correlation IDs.
- **Metrics**: Track call duration, acceptance rate, SDK latency, WebSocket health.
- **Error tracking**: Integrate with an error reporting service for crash analysis.
- **Audit log**: Record all state transitions, API calls, and user actions.

### 4.7 Plugin / Extension System

**Proposal**: Define a plugin interface for extending call handling:

```javascript
// Plugin interface
interface CopilotTasksPlugin {
  name: string;
  version: string;
  onCallReceived?(call: CallRequest): void;
  onCallAccepted?(call: CallRequest): void;
  onCallCompleted?(call: CallRecord): void;
  onSessionCreated?(session: Session): void;
}
```

**Use cases**: Custom notification sounds, Slack/Discord notifications, analytics hooks, custom voice providers.

### 4.8 Modular SDK Wrapper

**Proposal**: Wrap the Copilot SDK in an abstraction layer that:
- Normalizes error handling across SDK versions.
- Provides TypeScript-safe interfaces.
- Abstracts session management with lifecycle hooks.
- Enables mock implementations for testing.

### 4.9 Cloud Architecture (Future)

**Vision**: A cloud-hosted variant enabling:
- Multi-device session continuity.
- Shared team dashboards.
- Centralized call history and analytics.
- WebRTC-based voice (replacing Eleven Labs direct connect).

---

## 5. Phase 4 — Documentation Review

### 5.1 README Assessment

**Strengths**:
- Describes the MCP integration and voice call flow clearly.
- Includes development setup instructions.
- Documents the HTTP API endpoints.
- Lists environment variables.
- Describes the hooks system with known limitations.

**Weaknesses**:
- **No architecture diagram**: The relationship between components is not visualized.
- **No API reference**: HTTP endpoints lack request/response schema documentation.
- **No contributing guide**: No `CONTRIBUTING.md` for open-source contributors.
- **No changelog**: No `CHANGELOG.md` for version tracking.
- **No troubleshooting section**: Common issues (microphone permissions, SDK auth) not documented.
- **Missing platform notes**: No mention that process discovery is macOS-only.
- **No demo/screenshot**: Hard to understand the UX without visual examples.

### 5.2 Code Documentation

| File | Inline Comments | JSDoc | Rating |
|------|----------------|-------|--------|
| `main.js` | Sparse section headers | None | ⚠️ Needs work |
| `copilotSdk.js` | Moderate comments | None | ⚠️ Needs work |
| `elevenLabs.js` | Good inline comments | None | ✅ Acceptable |
| `callHistory.js` | Minimal | None | ⚠️ Needs work |
| `mcp-server.js` | Good comments | None | ✅ Acceptable |
| Renderers | Inline comments | N/A | ✅ Acceptable |
| `types.ts` | Type definitions serve as docs | N/A | ✅ Good |

### 5.3 Missing Documentation

1. **Architecture Decision Records (ADRs)**: No documentation of why certain technical decisions were made (e.g., why Eleven Labs over other voice providers, why SQLite over a JSON file).
2. **Sequence diagrams**: The call flow, session lifecycle, and MCP integration would benefit from sequence diagrams.
3. **API schema**: No OpenAPI/Swagger definition for the HTTP API.
4. **Environment setup guide**: The prerequisites (Eleven Labs API key, Copilot SDK access) need step-by-step onboarding docs.
5. **Security policy**: No `SECURITY.md` for responsible disclosure.

---

## 6. Phase 5 — Pull Request Proposals

### PR #1: Decompose `main.js` into Modular Services

**Title**: `refactor: decompose main.js into dedicated service modules`

**Description**: Extract the monolithic `main.js` (823 lines) into focused modules:
- `src/main/windowManager.js` — Window creation, positioning, and lifecycle.
- `src/main/ipcRouter.js` — IPC handler registration and routing.
- `src/main/httpApi.js` — Express server setup and endpoint handlers.
- `src/main/callManager.js` — Call queue, active call state, and call lifecycle.
- `src/main/trayManager.js` — Tray icon, menu, and update logic.

**Impact**: Improves maintainability, testability, and enables parallel development.

**Files changed**:
- `src/main/main.js` (major refactor — down to ~100 lines of composition)
- `src/main/windowManager.js` (new)
- `src/main/ipcRouter.js` (new)
- `src/main/httpApi.js` (new)
- `src/main/callManager.js` (new)
- `src/main/trayManager.js` (new)

**Estimated effort**: 2–3 days

---

### PR #2: Add HTTP API Authentication & Input Validation

**Title**: `security: add token-based auth and input validation for HTTP API`

**Description**: 
- Generate a random bearer token on startup, store in a known location (`~/.copilot-tasks/api-token`).
- Require `Authorization: Bearer <token>` on all API endpoints.
- Add request body validation for `/api/call` (validate `topic` is non-empty string, `questions` is array, etc.).
- Add rate limiting (e.g., 10 requests/second per IP).
- Update MCP server and hooks to read token from the known location.

**Impact**: Addresses the most significant security risk identified in the current implementation — unauthenticated localhost API.

**Files changed**:
- `src/main/main.js` (or new `httpApi.js`)
- `mcp-server.js`
- `hooks/voice-call.js`
- `README.md`

**Estimated effort**: 1 day

---

### PR #3: Cross-Platform Process Discovery

**Title**: `feat: add Windows and Linux support for process discovery`

**Description**:
- Replace `ps aux | grep copilot` with platform-specific implementations:
  - **macOS**: Keep existing `ps`/`lsof` approach.
  - **Windows**: Use `wmic process` or `tasklist` + `netstat`.
  - **Linux**: Use `/proc` filesystem or `ps`/`ss` commands.
- Replace `osascript` (AppleScript) window focusing with platform-appropriate alternatives.
- Add platform detection and strategy pattern for process discovery.

**Impact**: Enables the dashboard to show running Copilot sessions on all platforms.

**Files changed**:
- `src/main/copilotSdk.js` (refactor `discoverRunningProcesses()`)
- `src/main/platformUtils.js` (new — platform-specific utilities)

**Estimated effort**: 2 days

---

### PR #4: Implement Structured Logging

**Title**: `infra: replace console.log with structured logging`

**Description**:
- Add a lightweight logging library (e.g., `electron-log` or custom `pino`-based solution).
- Define log levels: `debug`, `info`, `warn`, `error`.
- Add correlation IDs to trace calls through the system.
- Log to both console and rotating file (`~/.copilot-tasks/logs/`).
- Replace all `console.log` / `console.error` calls with structured logger calls.
- Add startup banner with version, platform, and configuration summary.

**Impact**: Dramatically improves debuggability and operational visibility.

**Files changed**: All `.js` files in `src/main/`, `mcp-server.js`

**Estimated effort**: 1–2 days

---

### PR #5: Add Core Test Suite

**Title**: `test: add unit and integration test foundation`

**Description**:
- Set up test framework (Vitest, matching the existing Vite toolchain).
- Add unit tests for:
  - `callHistory.js` — CRUD operations, edge cases.
  - `elevenLabs.js` — Message parsing, summary extraction, duration calculation.
  - `mcp-server.js` — JSON-RPC protocol handling, tool listing, tool calling.
- Add integration tests for:
  - HTTP API endpoints (using `supertest`).
  - Call lifecycle (mock WebSocket, verify state transitions).
- Add CI configuration (GitHub Actions workflow).
- Target: 60%+ coverage on core modules.

**Impact**: Prevents regressions, enables confident refactoring, establishes quality baseline.

**Files changed**:
- `package.json` (add test dependencies and scripts)
- `vitest.config.js` (new)
- `tests/` directory (new — all test files)
- `.github/workflows/test.yml` (new)

**Estimated effort**: 3–4 days

---

## 7. Phase 6 — Code Improvements

### 7.1 Remove Unused Dependencies

The following dependencies are declared in `package.json` but never used in any source file:

- `react` — No JSX or React imports anywhere.
- `react-dom` — Same as above.
- `@fluentui/react-components` — No Fluent UI imports.

**Recommendation**: Remove from `dependencies` to reduce install size and avoid confusion. If they are planned for future use, move to a separate branch or document the intent.

### 7.2 Add Graceful Shutdown

Currently, the app can exit while a WebSocket is active, a call is pending, or an SDK session is streaming events. Add cleanup:

```javascript
// In main.js app.on('before-quit')
app.on('before-quit', async (event) => {
  event.preventDefault();
  try {
    // Close active voice session
    await closeActiveSession();
    // Leave all active SDK sessions
    for (const [sessionId] of activeSessions) {
      await leaveSession(sessionId);
    }
    // Close call history DB
    callHistoryDb.close();
    // Stop Express server
    httpServer.close();
  } finally {
    app.exit(0);
  }
});
```

### 7.3 Add Input Validation for HTTP API

The `/api/call` endpoint currently accepts any JSON body without validation:

```javascript
// Current (no validation)
app.post('/api/call', async (req, res) => {
  const { topic, context, questions } = req.body;
  // ... proceeds even if topic is undefined
});

// Proposed
app.post('/api/call', async (req, res) => {
  const { topic, context, questions } = req.body;
  
  if (!topic || typeof topic !== 'string' || topic.trim().length === 0) {
    return res.status(400).json({ 
      error: 'Validation error', 
      message: 'topic is required and must be a non-empty string' 
    });
  }
  
  if (questions && !Array.isArray(questions)) {
    return res.status(400).json({ 
      error: 'Validation error', 
      message: 'questions must be an array of strings' 
    });
  }
  
  // ... proceed with validated input
});
```

### 7.4 Fix Hook System Limitation

The `hooks/voice-call.js` returns the answer as `permissionDecision: 'deny'` which causes the Copilot agent to potentially retry. The hook should use a more appropriate response pattern:

```javascript
// Current (problematic)
return JSON.stringify({ 
  permissionDecision: 'deny', 
  reason: `User answered via voice call: ${result.result?.summary}` 
});

// Proposed — if the hooks API supports it, use 'allow' with modified content
// Otherwise, document this as a known limitation and provide a workaround
return JSON.stringify({ 
  permissionDecision: 'allow',
  // Override the tool parameters with the voice answer
  updatedParameters: {
    response: result.result?.summary || 'Answered via voice call'
  }
});
```

### 7.5 Add WebSocket Reconnection Logic

```javascript
// Proposed reconnection for elevenLabs.js
function startVoiceSession(callData, mainWindow, options = {}) {
  const maxReconnectAttempts = options.maxReconnectAttempts || 3;
  let reconnectAttempt = 0;
  
  function connect() {
    const ws = new WebSocket(ELEVENLABS_WS_URL);
    
    ws.on('close', (code) => {
      if (code !== 1000 && reconnectAttempt < maxReconnectAttempts) {
        reconnectAttempt++;
        const delay = Math.min(1000 * Math.pow(2, reconnectAttempt), 10000);
        console.log(`Reconnecting in ${delay}ms (attempt ${reconnectAttempt})`);
        setTimeout(connect, delay);
      }
    });
    
    ws.on('error', (error) => {
      console.error('WebSocket error:', error.message);
    });
    
    // ... rest of setup
  }
  
  connect();
}
```

### 7.6 Improve Type Safety

While `types.ts` defines types, they are not used at runtime. Add JSDoc type annotations to JavaScript files for IDE support:

```javascript
/**
 * @typedef {Object} CallRequest
 * @property {string} callId
 * @property {string} topic
 * @property {string} [context]
 * @property {string[]} [questions]
 * @property {number} timestamp
 */

/**
 * Process an incoming call request.
 * @param {CallRequest} callData
 * @returns {Promise<CallResult>}
 */
async function processCall(callData) {
  // ...
}
```

### 7.7 Replace `ScriptProcessorNode` with `AudioWorklet`

The call renderer uses the deprecated `ScriptProcessorNode` for audio capture. This should be replaced with `AudioWorkletNode` for better performance and to avoid main-thread audio processing:

```javascript
// Current (deprecated)
const processor = micAudioContext.createScriptProcessor(4096, 1, 1);
processor.onaudioprocess = (event) => { /* ... */ };

// Proposed (AudioWorklet)
// 1. Create audio-processor.js worklet
// 2. Register and use AudioWorkletNode
await micAudioContext.audioWorklet.addModule('audio-processor.js');
const workletNode = new AudioWorkletNode(micAudioContext, 'pcm-processor');
workletNode.port.onmessage = (event) => {
  const base64Audio = event.data;
  window.copilotTasks.sendVoiceAudio(currentCall.callId, base64Audio);
};
```

---

## 8. Phase 7 — Summary & Recommendations

### Priority Matrix

| Priority | Item | Impact | Effort |
|----------|------|--------|--------|
| **P0** | Add HTTP API authentication | High | Low |
| **P0** | Add input validation | High | Low |
| **P1** | Decompose `main.js` | High | Medium |
| **P1** | Add test suite | High | High |
| **P1** | Implement structured logging | Medium | Low |
| **P2** | Cross-platform process discovery | Medium | Medium |
| **P2** | Remove unused dependencies | Low | Trivial |
| **P2** | Add graceful shutdown | Medium | Low |
| **P3** | WebSocket reconnection logic | Medium | Low |
| **P3** | Replace ScriptProcessorNode | Low | Medium |
| **P3** | State machine for call lifecycle | Medium | Medium |
| **P4** | Plugin system | Low | High |
| **P4** | Cloud architecture | Low | Very High |

## Open Questions for Maintainer

1. Is the HTTP API intentionally unauthenticated for local-only usage?
2. Are React and Fluent UI dependencies planned for a future renderer refactor?
3. Is macOS-first support a deliberate strategic decision?
4. Would you be open to introducing structured logging as a baseline?
5. What is the long-term vision: local power tool or cloud-ready platform?


### Recommended Execution Order

1. **Security first**: PR #2 (API auth + validation) — immediate security improvement.
2. **Foundation**: PR #4 (structured logging) — enables better debugging for all subsequent work.
3. **Quality**: PR #5 (test suite) — establishes quality baseline before refactoring.
4. **Architecture**: PR #1 (decompose main.js) — enables maintainable growth.
5. **Platform**: PR #3 (cross-platform) — widens the user base.

### Overall Assessment

**Copilot Tasks** is an innovative prototype demonstrating the potential of voice-integrated AI development assistance. The codebase is functional and well-crafted for its stage, with a polished UI and creative demo capabilities. However, to transition from prototype to production-quality open-source project, the primary focus should be on:

1. **Security hardening** — The unauthenticated HTTP API is the most critical issue.
2. **Architectural decomposition** — The monolithic main process will become a bottleneck for contributions.
3. **Test infrastructure** — Zero test coverage makes refactoring risky and contributions fragile.
4. **Cross-platform support** — macOS-only features limit adoption.
5. **Documentation** — Architecture diagrams, API docs, and contributing guidelines will lower the barrier for contributors.

The project has strong potential and a clear value proposition. With the improvements outlined in this audit, it can evolve into a robust, community-driven tool that meaningfully enhances AI-assisted development workflows.

---

I would be happy to implement one or more of the proposed improvements in a focused Pull Request if aligned with the project's direction.

---

*End of audit.*
