---
name: distributed-actors
description: Best practices for defining distributed actors with @Trebuchet macro, method requirements, stateful actors, annotations, and error handling. Use when users need help structuring actors, managing state persistence, or understanding actor deployment configuration.
---

# Defining Distributed Actors

Best practices for creating distributed actors with Trebuchet.

## Overview

Distributed actors in Trebuchet follow Swift's distributed actor model with additional considerations for network communication and cloud deployment.

## The @Trebuchet Macro

The simplest way to create a Trebuchet actor is with the `@Trebuchet` macro:

```swift
@Trebuchet
distributed actor GameRoom {
    distributed func join(player: Player) -> RoomState
}
```

This is equivalent to:

```swift
distributed actor GameRoom {
    typealias ActorSystem = TrebuchetActorSystem

    distributed func join(player: Player) -> RoomState
}
```

## Method Requirements

All distributed methods must:

1. Be marked with `distributed`
2. Have parameters that conform to `Codable`
3. Have a return type that conforms to `Codable` (or be `Void`)

```swift
@Trebuchet
distributed actor UserService {
    // ✅ Good - Codable parameter and return type
    distributed func getUser(id: UUID) -> User

    // ✅ Good - Void return type
    distributed func deleteUser(id: UUID)

    // ✅ Good - Throws errors
    distributed func updateUser(_ user: User) throws -> User

    // ❌ Bad - Non-Codable parameter
    // distributed func process(stream: AsyncStream<Data>)
}
```

## Stateful Actors

For actors that need persistent state in serverless environments, conform to `StatefulActor`:

```swift
import TrebuchetCloud

@Trebuchet
distributed actor ShoppingCart: StatefulActor {
    typealias PersistentState = CartState

    var persistentState = CartState()

    distributed func addItem(_ item: Item) -> CartState {
        persistentState.items.append(item)
        return persistentState
    }

    func loadState(from store: any ActorStateStore) async throws {
        if let state = try await store.load(for: id.id, as: CartState.self) {
            persistentState = state
        }
    }

    func saveState(to store: any ActorStateStore) async throws {
        try await store.save(persistentState, for: id.id)
    }
}

struct CartState: Codable, Sendable {
    var items: [Item] = []
}
```

## Actor Annotations

Use comments to configure actor deployment:

```swift
// @trebuchet:memory=1024
// @trebuchet:timeout=60
// @trebuchet:isolated=true
@Trebuchet
distributed actor HeavyProcessor {
    distributed func process(data: Data) -> ProcessedResult
}
```

These annotations are read by the CLI during deployment and configure:

- **memory**: Memory allocation in MB (128-10240)
- **timeout**: Execution timeout in seconds (1-900)
- **isolated**: Run in dedicated Lambda function (vs shared)

## Error Handling

Distributed methods can throw errors that are serialized across the network:

```swift
enum GameError: Error, Codable {
    case roomFull
    case invalidPlayer
    case gameAlreadyStarted
}

@Trebuchet
distributed actor GameRoom {
    distributed func join(player: Player) throws -> RoomState {
        guard players.count < maxPlayers else {
            throw GameError.roomFull
        }
        // ...
    }
}
```

Error handling on the client:

```swift
do {
    let state = try await room.join(player: me)
} catch GameError.roomFull {
    print("Room is full")
} catch {
    print("Unexpected error: \(error)")
}
```

## Private State

Actors can have non-distributed state for local computation:

```swift
@Trebuchet
distributed actor GameRoom {
    // Private state (not distributed)
    private var players: [Player] = []
    private var startTime: Date?

    // Public distributed methods
    distributed func join(player: Player) -> RoomState {
        players.append(player)
        return RoomState(players: players)
    }

    distributed func getPlayerCount() -> Int {
        return players.count
    }
}
```

## Initialization

Actors must be initialized with an actor system:

```swift
@Trebuchet
distributed actor GameRoom {
    let maxPlayers: Int

    init(maxPlayers: Int, actorSystem: TrebuchetActorSystem) {
        self.maxPlayers = maxPlayers
        self.actorSystem = actorSystem
    }
}

// Server
let server = TrebuchetServer(transport: .webSocket(port: 8080))
let room = GameRoom(maxPlayers: 10, actorSystem: server.actorSystem)
await server.expose(room, as: "main-room")
```

## Actor Types

Different actor types have different characteristics:

### Request-Response Actors

Simple synchronous operations:

```swift
@Trebuchet
distributed actor Calculator {
    distributed func add(a: Int, b: Int) -> Int {
        return a + b
    }
}
```

### Stateful Actors

Actors that maintain state between invocations:

```swift
@Trebuchet
distributed actor Counter: StatefulActor {
    typealias PersistentState = Int

    var persistentState = 0

    distributed func increment() -> Int {
        persistentState += 1
        return persistentState
    }

    func loadState(from store: any ActorStateStore) async throws {
        if let state = try await store.load(for: id.id, as: Int.self) {
            persistentState = state
        }
    }

    func saveState(to store: any ActorStateStore) async throws {
        try await store.save(persistentState, for: id.id)
    }
}
```

### Streaming Actors

Actors that broadcast state changes:

```swift
@Trebuchet
distributed actor TodoList {
    @StreamedState var state: State = State()

    distributed func addTodo(title: String) -> TodoItem {
        let todo = TodoItem(title: title)
        state.todos.append(todo)  // Automatically notifies subscribers
        return todo
    }
}
```

## Best Practices

### Keep Methods Small

```swift
// ✅ Good - focused method
distributed func join(player: Player) -> RoomState

// ❌ Bad - doing too much
distributed func joinAndStartGameAndNotifyAll(player: Player) -> ComplexResult
```

### Use Value Types for Parameters

```swift
// ✅ Good - value type
struct Player: Codable, Sendable {
    let id: UUID
    let name: String
}

// ❌ Bad - class type (not Sendable)
class Player: Codable {
    let id: UUID
    let name: String
}
```

### Validate Input

```swift
distributed func join(player: Player) throws -> RoomState {
    guard !player.name.isEmpty else {
        throw GameError.invalidPlayer
    }
    guard players.count < maxPlayers else {
        throw GameError.roomFull
    }
    // ... proceed
}
```

### Document Public Methods

```swift
@Trebuchet
distributed actor GameRoom {
    /// Adds a player to the game room.
    ///
    /// - Parameter player: The player to add
    /// - Returns: The updated room state
    /// - Throws: `GameError.roomFull` if the room is at capacity
    distributed func join(player: Player) throws -> RoomState {
        // ...
    }
}
```

## See Also

- Getting started guide for basic actor setup
- Streaming guide for @StreamedState usage
- Cloud deployment guide for deploying actors to AWS Lambda
- SwiftUI integration guide for @RemoteActor usage
