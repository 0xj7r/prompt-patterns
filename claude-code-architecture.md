# Claude Code: Systems Architecture Patterns

An analysis of the key architectural patterns used across the Claude Code TypeScript codebase.

---

## 1. Priority Command Queue

**Files:** `src/utils/messageQueueManager.ts`, `src/utils/queueProcessor.ts`, `src/hooks/useQueueProcessor.ts`, `src/hooks/useCommandQueue.ts`

A module-level priority queue manages all commands flowing through the system. User input, task notifications, and orphaned permissions all converge into a single ordered queue. Three priority levels (`now > next > later`) ensure user input is never starved by system messages.

```ts
// src/utils/messageQueueManager.ts
const PRIORITY_ORDER: Record<QueuePriority, number> = {
  now: 0,
  next: 1,
  later: 2,
}

export function enqueue(command: QueuedCommand): void {
  commandQueue.push({ ...command, priority: command.priority ?? 'next' })
  notifySubscribers()
}

export function dequeue(filter?: (cmd: QueuedCommand) => boolean): QueuedCommand | undefined {
  let bestIdx = -1
  let bestPriority = Infinity
  for (let i = 0; i < commandQueue.length; i++) {
    const cmd = commandQueue[i]!
    if (filter && !filter(cmd)) continue
    const priority = PRIORITY_ORDER[cmd.priority ?? 'next']
    if (priority < bestPriority) {
      bestIdx = i
      bestPriority = priority
    }
  }
  // ...splice and return
}
```

The queue processor (`src/utils/queueProcessor.ts`) applies batching logic: slash commands and bash-mode commands are processed individually (for error isolation), while other commands with the same mode are drained as a batch.

React components subscribe via `useSyncExternalStore` to bypass React batching delays that caused missed notifications in the Ink terminal UI.

---

## 2. Signal Pattern (Lightweight Pub/Sub)

**Files:** `src/utils/signal.ts`

A tiny listener-set primitive used ~15 times across the codebase for pure event notification without stored state. Distinct from the store pattern (below) because there's no snapshot or getState.

```ts
// src/utils/signal.ts
export function createSignal<Args extends unknown[] = []>(): Signal<Args> {
  const listeners = new Set<(...args: Args) => void>()
  return {
    subscribe(listener) {
      listeners.add(listener)
      return () => { listeners.delete(listener) }
    },
    emit(...args) {
      for (const listener of listeners) listener(...args)
    },
    clear() { listeners.clear() },
  }
}
```

Used by `Mailbox`, `QueryGuard`, `messageQueueManager`, and other modules to decouple producers from consumers without pulling in a full event emitter.

---

## 3. Minimal Reactive Store

**Files:** `src/state/store.ts`, `src/state/AppStateStore.ts`, `src/state/selectors.ts`

A hand-rolled store with the exact same contract as React's `useSyncExternalStore`. No middleware, no reducers, no action types. State is updated with a pure function `(prev) => next`, listeners fire on referential change, and selectors derive computed views.

```ts
// src/state/store.ts
export function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

`AppState` (~450 fields) holds everything: MCP connections, plugin state, task registry, notification queue, speculation state, bridge configuration, permission context, and more. The `onChange` callback fires side effects (e.g., pushing state to CCR external metadata).

---

## 4. State Machine Guard (QueryGuard)

**Files:** `src/utils/QueryGuard.ts`

A synchronous three-state machine (`idle`, `dispatching`, `running`) prevents concurrent API queries. Compatible with `useSyncExternalStore` so React components re-render on transitions.

```
idle → dispatching  (reserve — queue processor claims the slot)
dispatching → running  (tryStart — API call begins)
idle → running  (tryStart — direct user submit, no queue)
running → idle  (end/forceEnd — query complete or cancelled)
dispatching → idle  (cancelReservation — nothing to process)
```

The `dispatching` state closes the async gap between dequeuing a command and starting the API call, preventing double-dequeue race conditions that would otherwise occur across React render cycles.

---

## 5. Mailbox Actor Pattern

**Files:** `src/utils/mailbox.ts`, `src/context/mailbox.tsx`

An actor-model mailbox for inter-agent communication. Supports three patterns:
- **Fire-and-forget:** `send(msg)` enqueues and returns
- **Synchronous poll:** `poll(predicate)` checks without blocking
- **Async receive:** `receive(predicate)` returns a Promise that resolves when a matching message arrives (or immediately if one is already queued)

```ts
// src/utils/mailbox.ts
export class Mailbox {
  private queue: Message[] = []
  private waiters: Waiter[] = []

  send(msg: Message): void {
    const idx = this.waiters.findIndex(w => w.fn(msg))
    if (idx !== -1) {
      const waiter = this.waiters.splice(idx, 1)[0]
      waiter.resolve(msg)  // Direct delivery to waiting receiver
      return
    }
    this.queue.push(msg)  // Buffer if no one is waiting
  }

  receive(fn: (msg: Message) => boolean = () => true): Promise<Message> {
    const idx = this.queue.findIndex(fn)
    if (idx !== -1) {
      return Promise.resolve(this.queue.splice(idx, 1)[0])
    }
    return new Promise(resolve => {
      this.waiters.push({ fn, resolve })
    })
  }
}
```

Messages have typed sources (`user`, `teammate`, `system`, `tick`, `task`) enabling selective receives. Provided to the React tree via `MailboxProvider`.

---

## 6. Notification Queue with Priority, Folding, and Invalidation

**Files:** `src/context/notifications.tsx`

Notifications use a priority-ordered queue (`low`, `medium`, `high`, `immediate`) with two advanced features:

- **Folding:** Notifications with the same `key` are merged via a `fold(accumulator, incoming)` reducer, preventing notification spam.
- **Invalidation:** A notification can declare `invalidates: ['key1', 'key2']` to remove stale notifications (e.g., a success notification invalidates the in-progress one).

Immediate-priority notifications bypass the queue and display instantly, clearing any active timeout.

---

## 7. Tool Definition Factory (buildTool)

**Files:** `src/Tool.ts`

All 60+ tools follow a single `Tool` interface with a `buildTool()` factory that applies defaults. Each tool is a plain object conforming to the type, not a class hierarchy.

```ts
// src/Tool.ts
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

Key capabilities on each tool:
- `isConcurrencySafe(input)` — can this run in parallel with other tools?
- `isReadOnly(input)` — safe for read-only mode?
- `isDestructive(input)` — delete, overwrite, send?
- `interruptBehavior()` — `'cancel'` or `'block'` when user submits mid-execution?
- `backfillObservableInput(input)` — mutate copies before observers see them (SDK stream, hooks, permissions)
- `validateInput()` → `isPermitted()` → `call()` — three-phase execution pipeline

This is a **Strategy pattern** where each tool defines its own execution, validation, permission, and display logic behind a uniform interface.

---

## 8. Retry with Exponential Backoff, Model Fallback, and Persistent Mode

**Files:** `src/services/api/withRetry.ts`

API calls go through an async generator `withRetry()` that implements:

- **Exponential backoff with jitter:** `baseDelay * 2^(attempt-1) + random(0, 0.25 * baseDelay)`
- **Model fallback:** After 3 consecutive 529 (overloaded) errors, falls back to a cheaper model
- **Fast-mode cooldown:** Short retry-after headers keep fast mode active (preserve prompt cache), long ones trigger a timed cooldown to standard speed
- **Persistent retry mode:** For unattended sessions, retries 429/529 indefinitely with chunked sleeps that yield heartbeat messages to prevent session timeout
- **Auth refresh:** Automatic OAuth token refresh on 401, AWS credential refresh on Bedrock auth errors, GCP refresh on Vertex errors
- **Stale connection recovery:** Detects ECONNRESET/EPIPE and disables HTTP keep-alive

```ts
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client: Anthropic, attempt: number, context: RetryContext) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T> {
  // Yields SystemAPIErrorMessage during waits (shown to user + keeps session alive)
  // Returns T on success, throws CannotRetryError on exhaustion
}
```

The async generator pattern lets callers consume retry status messages as they occur, rather than waiting for the entire retry sequence to resolve.

---

## 9. DOM-like Event System with Capture/Bubble Phases

**Files:** `src/ink/events/dispatcher.ts`, `src/ink/events/emitter.ts`, `src/ink/events/event-handlers.ts`

The terminal UI implements a full DOM-style event dispatch system:

- **Capture phase** (root → target): handlers fire top-down
- **Bubble phase** (target → root): handlers fire bottom-up
- **Propagation control:** `stopPropagation()`, `stopImmediatePropagation()`, `preventDefault()`
- **React priority scheduling:** events map to `DiscreteEventPriority` (keyboard, click), `ContinuousEventPriority` (resize, scroll), or `DefaultEventPriority`

```ts
// src/ink/events/dispatcher.ts
function collectListeners(target: EventTarget, event: TerminalEvent): DispatchListener[] {
  // Walk target → root
  // Capture handlers unshift (root-first)
  // Bubble handlers push (target-first)
  // Result: [root-cap, ..., parent-cap, target-cap, target-bub, parent-bub, ..., root-bub]
}
```

The `Dispatcher` class owns dispatch state and is read by the React reconciler's host config for `resolveUpdatePriority` and `resolveEventType` — mirroring how `react-dom` reads `ReactDOMSharedInternals`.

---

## 10. Custom React Reconciler (Ink)

**Files:** `src/ink/reconciler.ts`, `src/ink/dom.ts`, `src/ink/ink.tsx`, `src/ink/layout/`

The terminal UI runs on a custom React reconciler (via `react-reconciler`) that renders to a virtual DOM of terminal nodes. This gives full React component model (hooks, context, suspense) while targeting a terminal instead of a browser.

Key subsystems:
- **Yoga layout engine** (`src/ink/layout/yoga.ts`): CSS flexbox layout for terminal
- **Throttled rendering** (`src/ink/ink.tsx`): lodash `throttle` at `FRAME_INTERVAL_MS`, with immediate renders for test environments
- **Virtual scrolling** (`src/hooks/useVirtualScroll.ts`): only renders visible messages
- **Hit testing** (`src/ink/hit-test.ts`): click detection on terminal nodes

---

## 11. Discriminated Union Task System

**Files:** `src/tasks/types.ts`, `src/tasks/*/`

Background work is modeled as a discriminated union of task states:

```ts
export type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

Each variant has its own state shape and lifecycle. Tasks live in `AppState.tasks` (keyed by ID) and can be backgrounded, foregrounded, or viewed. The `isBackgroundTask()` type guard narrows the union for UI components.

Input routing uses a selector (`getActiveAgentForInput`) that returns a discriminated union:

```ts
export type ActiveAgentForInput =
  | { type: 'leader' }
  | { type: 'viewed'; task: InProcessTeammateTaskState }
  | { type: 'named_agent'; task: LocalAgentTaskState }
```

---

## 12. Hook Lifecycle System

**Files:** `src/utils/hooks.ts`, `src/utils/hooks/hookHelpers.ts`, `src/query/stopHooks.ts`

External shell commands hook into the tool execution pipeline at defined points:

| Event | When | Can Block? |
|---|---|---|
| `PreToolUse` | Before tool execution | Yes |
| `PostToolUse` | After successful execution | Yes |
| `PostToolUseFailure` | After failed execution | Yes |
| `Stop` | When assistant completes | Yes |
| `SubagentStop` | When a subagent completes | Yes |
| `UserPromptSubmit` | When user submits input | Yes |
| `PermissionRequest` | When permission is needed | Yes |
| `TaskCompleted` | When a background task finishes | No |
| `TeammateIdle` | When a teammate becomes idle | No |

Hooks receive JSON context on stdin (tool name, input, result) and can return decisions (block, allow, modify input). This is the extension mechanism for custom CI/CD integration, security policies, and workflow automation.

---

## 13. API Context Management (Microcompact)

**Files:** `src/services/compact/apiMicrocompact.ts`

Server-side context management via declarative edit strategies:

- **`clear_tool_uses_20250919`:** When input tokens exceed a threshold, clear old tool use/result pairs while keeping the N most recent. Configurable per-tool exclusions and input clearing.
- **`clear_thinking_20251015`:** Prune thinking blocks from earlier turns (keep only recent N thinking turns, or all).

```ts
export type ContextEditStrategy =
  | { type: 'clear_tool_uses_20250919'; trigger?: {...}; keep?: {...}; clear_at_least?: {...} }
  | { type: 'clear_thinking_20251015'; keep: { type: 'thinking_turns'; value: number } | 'all' }
```

This replaces client-side context window management with API-level instructions, letting the server make optimal decisions about what to preserve.

---

## 14. MCP Client Multiplexing

**Files:** `src/services/mcp/client.ts`

Multiple MCP (Model Context Protocol) servers are managed through a connection multiplexer. Each server gets its own `Client` instance with transport abstraction (stdio, SSE, StreamableHTTP). Tools from all connected servers are flattened into the unified tool list, with names prefixed by server (`mcp__server__tool`) to prevent collisions.

Resources, prompts, and tools are dynamically discovered and reconciled when servers connect/disconnect. OAuth authentication flows are handled per-server.

---

## 15. Plugin Reconciliation

**Files:** `src/utils/plugins/reconciler.ts`, `src/utils/plugins/pluginLoader.ts`, `src/services/plugins/PluginInstallationManager.ts`

Plugins are reconciled between desired state (settings) and actual state (installed packages). The reconciler detects drift, installs missing plugins, and tracks installation status with a state machine (`pending → installing → installed | failed`). A `needsRefresh` flag in AppState signals when plugin state on disk has diverged from in-memory state.

---

## 16. Speculative Execution

**Files:** `src/state/AppStateStore.ts` (SpeculationState), `src/services/PromptSuggestion/speculation.ts`

The system speculatively executes tool calls before the user confirms, tracking state as a discriminated union:

```ts
export type SpeculationState =
  | { status: 'idle' }
  | { status: 'active'; id: string; abort: () => void; startTime: number;
      messagesRef: { current: Message[] };
      writtenPathsRef: { current: Set<string> };
      boundary: CompletionBoundary | null; ... }
```

Speculated results are tracked in mutable refs (not state) to avoid array spreading on every message. Written file paths are tracked in an overlay set. If the user accepts, the speculated results are promoted; if rejected, `abort()` cancels and state resets to idle.

---

## Summary Table

| Pattern | Purpose | Key Primitive |
|---|---|---|
| Priority Queue | Command ordering + batching | `messageQueueManager.ts` |
| Signal | Decoupled notification | `createSignal()` |
| Reactive Store | Global state + React integration | `createStore()` + `useSyncExternalStore` |
| State Machine Guard | Concurrent query prevention | `QueryGuard` |
| Actor Mailbox | Inter-agent messaging | `Mailbox` class |
| Notification Queue | UI notifications with fold/invalidate | `useNotifications()` |
| Tool Factory | Uniform tool interface | `buildTool()` |
| Retry Generator | Resilient API calls | `withRetry()` async generator |
| DOM Event System | Terminal UI event dispatch | `Dispatcher` + capture/bubble |
| Custom Reconciler | React for terminal | `react-reconciler` + Yoga |
| Discriminated Unions | Type-safe task/state variants | `TaskState`, `ActiveAgentForInput` |
| Hook Lifecycle | External extensibility | Shell hook events |
| Context Management | Server-side token pruning | `ContextEditStrategy` |
| MCP Multiplexing | Multi-server tool aggregation | `@modelcontextprotocol/sdk` |
| Plugin Reconciliation | Desired vs actual state sync | Reconciler + state machine |
| Speculative Execution | Pre-confirm tool execution | `SpeculationState` + abort |
