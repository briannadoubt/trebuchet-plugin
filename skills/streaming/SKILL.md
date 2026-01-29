---
name: streaming
description: Real-time state streaming with @StreamedState, @ObservedActor, stream resumption, filtering, and delta encoding. Use when users need automatic state synchronization, reactive UI updates, or bandwidth optimization for large state objects.
---

# Realtime State Streaming

Stream state changes from distributed actors to clients in realtime with automatic synchronization, reconnection, filtering, and bandwidth optimization.

## Overview

Trebuchet's streaming feature allows distributed actors to expose reactive state that automatically updates all connected clients in realtime. This eliminates manual polling and provides a seamless, reactive experience with production-ready features like graceful reconnection, server-side filtering, and delta encoding.

## Quick Start

### Defining a Streaming Actor

Use the `@StreamedState` macro to make a property automatically notify subscribers:

```swift
@Trebuchet
public distributed actor TodoList {
    @StreamedState public var state: State = State()

    public distributed func addTodo(title: String) -> TodoItem {
        let todo = TodoItem(title: title)
        state.todos.append(todo)  // Automatically notifies all subscribers
        return todo
    }
}

public struct State: Codable, Sendable {
    var todos: [TodoItem] = []
}
```

### SwiftUI Integration

Use `@ObservedActor` to automatically subscribe to state streams:

```swift
struct TodoListView: View {
    @ObservedActor("todos", observe: \TodoList.observeState)
    var state

    var body: some View {
        if let currentState = state {
            List(currentState.todos) { todo in
                Text(todo.title)
            }
        } else if $state.isConnecting {
            ProgressView("Connecting...")
        }
    }
}
```

## How It Works

### @StreamedState Macro

The `@StreamedState` macro transforms a property into a streaming state property with automatic change tracking. It generates:

- Backing storage (`_state_storage`)
- Continuation array for subscribers (`_state_continuations`)
- Computed property with getter/setter
- Change notification method
- Observe method (`observeState()`)
- Stream accessor for server-side iteration

**Example expansion:**

```swift
// From:
@StreamedState var state: State = State()

// To:
private var _state_storage: State = State()
private var _state_continuations: [AsyncStream<State>.Continuation] = []

var state: State {
    get { _state_storage }
    set {
        _state_storage = newValue
        _notifyStateChange()
    }
}

private func _notifyStateChange() {
    for continuation in _state_continuations {
        continuation.yield(_state_storage)
    }
}

public func observeState() -> AsyncStream<State> {
    AsyncStream { continuation in
        _state_continuations.append(continuation)
        continuation.yield(_state_storage)  // Send initial state

        continuation.onTermination = { [weak self] _ in
            Task {
                await self?._removeStateContinuation(continuation)
            }
        }
    }
}
```

### @ObservedActor Property Wrapper

The `@ObservedActor` property wrapper provides:

- Automatic subscription on connection
- State updates trigger view re-renders
- Access to the actor via `$state.actor`
- Connection status via `$state.isConnecting`, `$state.error`

## Wire Protocol

Streaming uses a multi-envelope protocol:

1. **StreamStartEnvelope** - Sent when stream is initiated
   - `streamID`: Unique identifier for this stream
   - `callID`: Correlates with the original invocation
   - `actorID`: The actor being observed
   - `targetIdentifier`: The observe method name

2. **StreamDataEnvelope** - Sent for each state update
   - `streamID`: Stream identifier
   - `sequenceNumber`: Monotonic counter for deduplication
   - `data`: Encoded state value
   - `timestamp`: When the update was generated

3. **StreamEndEnvelope** - Sent when stream completes
   - `streamID`: Stream identifier
   - `reason`: Why the stream ended (completed, error, etc.)

4. **StreamErrorEnvelope** - Sent on error
   - `streamID`: Stream identifier
   - `errorMessage`: Error description

5. **StreamResumeEnvelope** - Sent by client to resume after reconnection
   - `streamID`: Stream to resume
   - `lastSequence`: Last sequence number received
   - `actorID`: The actor to observe
   - `targetIdentifier`: The observe method name

### Flow Diagram

```
Client                          Server
  │                               │
  ├─ InvocationEnvelope ────────>│  (call observeState())
  │  callID: abc-123              │
  │  target: "observeState"       │
  │                               │
  │<─ StreamStartEnvelope ────────┤  (stream initiated)
  │  streamID: xyz-789            │
  │  callID: abc-123              │
  │                               │
  │<─ StreamDataEnvelope ─────────┤  (initial state)
  │  streamID: xyz-789            │
  │  sequenceNumber: 1            │
  │                               │
  │     [state changes on server] │
  │                               │
  │<─ StreamDataEnvelope ─────────┤  (updated state)
  │  streamID: xyz-789            │
  │  sequenceNumber: 2            │
```

## Stream Resumption & Reconnection

**Implementation Status**: ✅ Fully Implemented (as of PR #22, January 2026)

Gracefully handles connection loss with automatic stream resumption, ensuring clients don't miss updates during brief disconnections.

**Core Implementation** (Trebuchet PR #22):
- `TrebuchetActorSystem.remoteCallStream()` returns `(streamID: UUID, stream: AsyncStream<Res>)` with real stream IDs
- `StreamRegistry.createResumedStream()` creates streams with specific streamIDs from checkpoints
- `TrebuchetClient.resumeStream()` sends `StreamResumeEnvelope` to server on reconnection
- `ObservedActor.attemptStreamResume()` handles automatic client-side stream resumption
- Server-side replay logic via `WebSocketLambdaHandler` with `ServerStreamBuffer`

### How It Works

1. **Normal Operation**:
   - Server buffers recent stream data (100 items default, 5-minute TTL)
   - Client receives StreamData with sequence numbers
   - Client tracks last sequence in checkpoint

2. **On Disconnection**:
   - Client saves checkpoint (streamID, lastSequence, actorID, method)
   - Server maintains buffer for reconnection window
   - Stream continuations are cleaned up

3. **On Reconnection**:
   - Client sends StreamResumeEnvelope with checkpoint info
   - Server checks if buffered data exists:
     - **Buffer available**: Replays missed updates from buffer
     - **Buffer expired**: Sends StreamStart and current state

### Configuration

```swift
// Server-side: Configure buffer size and TTL
let server = TrebuchetServer(/* ... */)
// Default: maxBufferSize: 100, ttl: 300 seconds

// For AWS Lambda
let handler = WebSocketLambdaHandler(/* ... */)
// Default: maxBufferSize: 100, ttl: 300 seconds
```

### Example Flow

```
Client loses connection at sequence 42
Client reconnects 30 seconds later

Client → Server: StreamResumeEnvelope {
    streamID: xyz-789
    lastSequence: 42
    actorID: "todos"
    targetIdentifier: "observeState"
}

Server checks buffer:
- Has sequences: 43, 44, 45, 46

Server → Client: StreamDataEnvelope (seq: 43)
Server → Client: StreamDataEnvelope (seq: 44)
Server → Client: StreamDataEnvelope (seq: 45)
Server → Client: StreamDataEnvelope (seq: 46)

Client now caught up!
```

### AWS Lambda Considerations

For serverless deployments, buffer replay works when the same Lambda container handles reconnection (common due to warm containers). If a different container handles the request, the stream restarts from current state. This is a correct fallback behavior with no data loss.

## Filtered Streams

**Implementation Status**: ✅ Fully Implemented

Server-side filtering reduces bandwidth and client-side processing by only sending relevant updates.

### Filter Types

1. **All** (default): No filtering, pass through all updates
2. **Predefined**: Use named filters with parameters
3. **Custom**: Client-defined filter logic (extensible via Filterable protocol)

### Implemented Predefined Filters

#### Changed Filter
Only sends updates when the value actually changes from the previous value.

```swift
// Client subscribes with changed filter
let filter = StreamFilter.predefined("changed")
let stream = await todoList.observeState(filter: filter)
// Only receives updates when state changes (bytewise comparison)
```

#### NonEmpty Filter
Only sends updates for non-empty collections, strings, or dictionaries.

```swift
// Only receive updates when list has items
let filter = StreamFilter.predefined("nonEmpty")
let stream = await todoList.observeState(filter: filter)
```

#### Threshold Filter
Only sends updates when numeric values cross a threshold.

```swift
// Only receive when count exceeds 100
let filter = StreamFilter.predefined("threshold", parameters: [
    "value": "100",
    "comparison": "gt",  // gt, gte, lt, lte, eq, neq
    "field": "count"     // optional: for nested values
])
let stream = await counter.observeState(filter: filter)
```

**Supported comparisons**:
- `gt` or `>`: Greater than
- `gte` or `>=`: Greater than or equal
- `lt` or `<`: Less than
- `lte` or `<=`: Less than or equal
- `eq` or `==`: Equal
- `neq` or `!=`: Not equal

### Benefits

- **Reduced network traffic**: Skip redundant or irrelevant updates
- **Lower client-side processing**: Clients only handle meaningful changes
- **Battery savings**: Fewer wake-ups on mobile devices
- **Better scalability**: Less data to broadcast to concurrent clients

## Delta Encoding

Sends only changed fields to optimize bandwidth for large state objects.

### How It Works

1. **Server Side**:
   - DeltaStreamManager tracks last sent value
   - Computes delta from previous to current
   - Sends delta if available, otherwise full state

2. **Client Side**:
   - DeltaStreamApplier maintains current value
   - Applies deltas incrementally
   - Falls back to full state when needed

### Implementation

```swift
// Make state support delta encoding
extension TodoList.State: DeltaCodable {
    func delta(from previous: TodoList.State) -> TodoList.State? {
        // Only include changed todos
        let changedTodos = todos.filter { todo in
            !previous.todos.contains(todo)
        }

        guard !changedTodos.isEmpty || pendingCount != previous.pendingCount else {
            return nil // No changes
        }

        return State(todos: changedTodos, pendingCount: pendingCount)
    }

    func applying(delta: TodoList.State) -> TodoList.State {
        var updated = self
        // Merge changed todos
        for todo in delta.todos {
            if let index = updated.todos.firstIndex(where: { $0.id == todo.id }) {
                updated.todos[index] = todo
            } else {
                updated.todos.append(todo)
            }
        }
        updated.pendingCount = delta.pendingCount
        return updated
    }
}

// Server uses delta manager
let manager = DeltaStreamManager<TodoList.State>()
let delta = try await manager.encode(newState)
// Automatically sends delta when possible

// Client applies deltas
let applier = DeltaStreamApplier<TodoList.State>()
let currentState = try await applier.apply(delta)
```

### When to Use Delta Encoding

- Large state objects (> 10KB)
- Frequent small updates to large collections
- Mobile or bandwidth-constrained clients
- High-frequency updates

### Trade-offs

- Added complexity in delta computation
- Requires careful implementation of merge logic
- Must handle edge cases (concurrent updates, conflicts)

## Advanced Usage

### Multiple Streamed Properties

```swift
@Trebuchet
public distributed actor GameServer {
    @StreamedState public var gameState: GameState = GameState()
    @StreamedState public var metrics: Metrics = Metrics()

    // Macro generates:
    // - observeGameState() -> AsyncStream<GameState>
    // - observeMetrics() -> AsyncStream<Metrics>
}
```

### Manual Stream Subscription

```swift
let client = TrebuchetClient(transport: .webSocket(host: "localhost", port: 8080))
try await client.connect()

let todoList = try client.resolve(TodoList.self, id: "todos")
let stream = await todoList.observeState()

for await state in stream {
    print("Todos: \(state.todos.count)")
}
```

### SwiftUI with Multiple Streams

```swift
struct GameView: View {
    @ObservedActor("game", observe: \GameServer.observeGameState)
    var gameState

    @ObservedActor("game", observe: \GameServer.observeMetrics)
    var metrics

    var body: some View {
        if let state = gameState, let metrics = metrics {
            VStack {
                Text("Score: \(state.score)")
                Text("Players: \(metrics.playerCount)")

                Button("Next Level") {
                    Task {
                        try? await $gameState.actor?.advanceLevel()
                    }
                }
            }
        } else if $gameState.isConnecting {
            ProgressView("Connecting...")
        }
    }
}
```

## Persistent State with Streaming

Seamlessly integrate persistent state storage with realtime streaming for serverless deployments.

### StatefulStreamingActor

Combines persistent state storage with automatic streaming updates:

```swift
import Trebuchet
import TrebuchetCloud

@Trebuchet
public distributed actor TodoList: StatefulStreamingActor {
    public typealias PersistentState = State

    private let stateStore: ActorStateStore

    @StreamedState public var state = State()

    public var persistentState: State {
        get { state }
        set { state = newValue }
    }

    public init(
        actorSystem: TrebuchetActorSystem,
        stateStore: ActorStateStore
    ) async throws {
        self.actorSystem = actorSystem
        self.stateStore = stateStore
        try await loadState(from: stateStore)
    }

    public func loadState(from store: any ActorStateStore) async throws {
        if let loaded = try await store.load(for: id.id, as: State.self) {
            state = loaded  // Triggers stream update to all clients
        }
    }

    public func saveState(to store: any ActorStateStore) async throws {
        try await store.save(state, for: id.id)
    }

    public distributed func addTodo(title: String) async throws -> TodoItem {
        let todo = TodoItem(title: title)
        var newState = state
        newState.todos.append(todo)
        state = newState  // 1. Streams to all clients

        try await saveState(to: stateStore)  // 2. Persists to storage

        return todo
    }
}
```

### Helper Methods

The `StatefulStreamingActor` protocol provides convenience methods:

```swift
// Single field updates
try await updateState(\.count, to: state.count + 1, store: stateStore)

// Complex transformations
public distributed func completeTodo(_ id: UUID) async throws {
    try await transformState(store: stateStore) { currentState in
        var newState = currentState
        if let index = newState.todos.firstIndex(where: { $0.id == id }) {
            newState.todos[index].completed = true
        }
        newState.lastUpdated = Date()
        return newState
    }
    // Automatically streams AND persists
}
```

## Database Change Stream Integration

Synchronize actor state across multiple instances using database change streams.

### PostgreSQL LISTEN/NOTIFY

**Implementation Status**: ✅ Fully Implemented

Complete PostgreSQL integration with state storage and LISTEN/NOTIFY for multi-instance synchronization.

```swift
import TrebuchetPostgreSQL

// State Store for actor persistence
let stateStore = try await PostgreSQLStateStore(
    host: "localhost",
    database: "trebuchet",
    username: "postgres",
    password: "password"
)

// Stream Adapter for multi-instance synchronization
let adapter = try await PostgreSQLStreamAdapter(
    host: "localhost",
    database: "trebuchet",
    username: "postgres"
)

let notificationStream = try await adapter.start()

// Process state change notifications
for await change in notificationStream {
    print("Actor \(change.actorID) updated to sequence \(change.sequenceNumber)")
    // Reload actor state from PostgreSQL
    try await reloadActor(id: change.actorID)
}
```

## Performance Considerations

### Bandwidth

- Only changed state is sent (entire state object per update)
- Sequence numbers add minimal overhead (8 bytes per message)
- Use delta encoding to optimize large state objects
- Server-side filtering reduces unnecessary updates

### Memory

- Each subscriber holds a continuation in the actor's array
- Continuations are weak-referenced and cleaned up on termination
- Stream registry holds active streams until explicitly removed
- Stream buffers use TTL to prevent memory leaks

### Concurrency

- All stream operations are actor-isolated
- No manual locking needed
- SwiftUI updates happen on MainActor

## Troubleshooting

### Streams Not Updating

**Problem**: Views don't update when state changes

**Solutions**:
- Ensure property is marked with `@StreamedState`
- Verify mutations use property setter (not direct storage access)
- Check connection state in SwiftUI view

### Connection Issues

**Problem**: `$state.isConnecting` stays true

**Solutions**:
- Verify server is running and accessible
- Check WebSocket endpoint configuration
- Look for errors in `$state.error`

### Build Errors

**Problem**: "Cannot find 'observeState' in scope"

**Solutions**:
- Ensure `@Trebuchet` macro is applied to actor
- Verify `@StreamedState` is applied to property
- Clean build folder and rebuild

## See Also

- SwiftUI integration guide for complete `@ObservedActor` documentation
- Cloud deployment for AWS Lambda streaming with WebSocket API Gateway
- PostgreSQL adapter for multi-instance synchronization
