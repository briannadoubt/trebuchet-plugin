---
name: getting-started
description: Installation, setup, and basic usage of Trebuchet distributed actors. Use when users need help starting a new Trebuchet project, creating their first actor, or understanding the fundamental server/client pattern.
---

# Getting Started with Trebuchet

Location-transparent distributed actors for Swift. Make RPC stupid simple.

## Overview

Trebuchet is a Swift 6.2 distributed actor framework that lets you define actors once and use them seamlessly whether they're local or remote. Your actors work the same whether they're in the same process or across network boundaries – Trebuchet handles the networking transparently.

## Installation

### Library

Add Trebuchet to your `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/briannadoubt/Trebuchet.git", from: "0.4.0")
]
```

Then add it to your target:

```swift
.target(
    name: "MyApp",
    dependencies: ["Trebuchet"]
)
```

### Xcode Projects

Trebuchet works seamlessly with Xcode projects (`.xcodeproj` or `.xcworkspace`):

1. **Add Package Dependency**: In Xcode, go to File → Add Package Dependencies
2. **Enter URL**: `https://github.com/briannadoubt/Trebuchet.git`
3. **Select Version**: 0.4.0 or later

The Trebuchet CLI automatically detects Xcode projects with **zero configuration required**. Just run `trebuchet dev` or `trebuchet deploy` in your project directory.

### CLI Tool

Install the `trebuchet` CLI for cloud deployment:

```bash
# Using Mint (recommended)
mint install briannadoubt/Trebuchet

# Or build from source
git clone https://github.com/briannadoubt/Trebuchet.git
cd Trebuchet
swift build -c release
cp .build/release/trebuchet /usr/local/bin/
```

## Requirements

- Swift 6.2+
- macOS 14+ / iOS 17+ / tvOS 17+ / watchOS 10+

## Creating Your First Distributed Actor

Use the `@Trebuchet` macro to mark an actor for distributed communication:

```swift
import Trebuchet

@Trebuchet
distributed actor Counter {
    private var count = 0

    distributed func increment() -> Int {
        count += 1
        return count
    }

    distributed func get() -> Int {
        return count
    }
}
```

The `@Trebuchet` macro automatically:
- Adds `typealias ActorSystem = TrebuchetActorSystem`
- Adds `TrebuchetActor` protocol conformance (requires `init(actorSystem:)`)

For actors with custom initialization, you can add both the required and custom initializers:

```swift
@Trebuchet
distributed actor Counter {
    private var count: Int

    // Required by TrebuchetActor protocol
    init(actorSystem: TrebuchetActorSystem) {
        self.count = 0
        self.actorSystem = actorSystem
    }

    // Optional: Custom initializer
    init(startCount: Int, actorSystem: TrebuchetActorSystem) {
        self.count = startCount
        self.actorSystem = actorSystem
    }
}
```

### Method Requirements

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

## Running a Server

Expose your actors on a server:

```swift
import Trebuchet

let server = TrebuchetServer(transport: .webSocket(port: 8080))
let counter = Counter(actorSystem: server.actorSystem)
await server.expose(counter, as: "counter")

print("Server running on port 8080")
try await server.run()
```

## Connecting as a Client

Resolve and call remote actors:

```swift
import Trebuchet

let client = TrebuchetClient(transport: .webSocket(host: "localhost", port: 8080))
try await client.connect()

let counter = try client.resolve(Counter.self, id: "counter")
let newValue = try await counter.increment()
print("Counter is now: \(newValue)")
```

## Complete Example

Here's a complete example showing a simple game room:

```swift
import Trebuchet

// Define the actor
@Trebuchet
distributed actor GameRoom {
    private var players: [Player] = []

    distributed func join(player: Player) -> RoomState {
        players.append(player)
        return RoomState(players: players)
    }

    distributed func leave(player: Player) {
        players.removeAll { $0.id == player.id }
    }
}

struct Player: Codable, Sendable {
    let id: UUID
    let name: String
}

struct RoomState: Codable, Sendable {
    let players: [Player]
}

// Server
@main
struct GameServer {
    static func main() async throws {
        let server = TrebuchetServer(transport: .webSocket(port: 8080))
        let room = GameRoom(actorSystem: server.actorSystem)
        await server.expose(room, as: "main-room")

        print("Game server running on port 8080")
        try await server.run()
    }
}

// Client
let client = TrebuchetClient(transport: .webSocket(host: "localhost", port: 8080))
try await client.connect()

let room = try client.resolve(GameRoom.self, id: "main-room")
let me = Player(id: UUID(), name: "Alice")
let state = try await room.join(player: me)  // Looks local, works remotely!
print("Joined room with \(state.players.count) players")
```

## Using with SwiftUI

Trebuchet provides SwiftUI integration for reactive actor connections:

```swift
import SwiftUI
import Trebuchet

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .trebuchet(transport: .webSocket(host: "api.example.com", port: 8080))
        }
    }
}

struct CounterView: View {
    @RemoteActor(id: "counter") var counter: Counter?

    var body: some View {
        switch $counter.state {
        case .loading:
            ProgressView()
        case .resolved(let counter):
            CounterContent(counter: counter)
        case .failed(let error):
            Text("Error: \(error.localizedDescription)")
        case .disconnected:
            Text("Disconnected")
        }
    }
}
```

## Next Steps

- Learn about defining actors with best practices
- Explore real-time streaming with `@StreamedState`
- Integrate with SwiftUI using `@RemoteActor` and `@ObservedActor`
- Deploy to the cloud with AWS Lambda, GCP, or Azure
- Add security with authentication and authorization
- Monitor with structured logging and metrics

## Common Patterns

### Error Handling

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

### Server Configuration

Configure server behavior:

```swift
let server = TrebuchetServer(
    transport: .webSocket(port: 8080),
    configuration: .init(
        maxMessageSize: 1024 * 1024,  // 1MB
        keepAlive: true,
        compressionEnabled: true
    )
)
```

### Client Lifecycle

Handle connection lifecycle:

```swift
let client = TrebuchetClient(transport: .webSocket(host: "localhost", port: 8080))

do {
    try await client.connect()
    defer { await client.disconnect() }

    let counter = try client.resolve(Counter.self, id: "counter")
    let value = try await counter.increment()

} catch {
    print("Connection failed: \(error)")
}
```

## Troubleshooting

### Actor Resolution Fails

If `client.resolve()` throws an error:
- Ensure the server has called `server.expose(actor, as: "id")`
- Check that the actor ID matches exactly
- Verify the client is connected: `try await client.connect()`

### Serialization Errors

If you get Codable errors:
- Ensure all distributed method parameters and return types conform to `Codable`
- Make custom types `Sendable` for Swift 6 concurrency
- Use `Codable` enums for error types

### Connection Issues

If the client can't connect:
- Verify the server is running and listening on the expected port
- Check firewall rules allow WebSocket connections
- Use `transport: .webSocket(host: "localhost", port: 8080)` for local testing

## Documentation

Full documentation is available at **[briannadoubt.github.io/Trebuchet](https://briannadoubt.github.io/Trebuchet/documentation/trebuchet/)**.
