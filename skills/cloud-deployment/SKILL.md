---
name: cloud-deployment
description: Serverless deployment architecture with CloudGateway, CloudProvider abstraction, ServiceRegistry, ActorStateStore, and local development setup. Use when users need to understand cloud deployment concepts or deploy actors to serverless platforms.
---

# Cloud Deployment Overview

Learn how Trebuchet enables serverless deployment of distributed actors.

## Overview

TrebuchetCloud provides the abstractions needed to deploy Swift distributed actors to serverless platforms. Instead of running a persistent server, your actors execute on-demand in response to invocations.

## Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                      Cloud Environment                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              CloudGateway                               │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌────────────┐   │  │
│  │  │ UserService │    │  GameRoom   │    │   Lobby    │   │  │
│  │  └─────────────┘    └─────────────┘    └────────────┘   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              ↓                                │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐    │
│  │  StateStore  │  │   Registry   │  │   HTTP Endpoint   │    │
│  │ (DynamoDB)   │  │  (CloudMap)  │  │  (API Gateway)    │    │
│  └──────────────┘  └──────────────┘  └───────────────────┘    │
└───────────────────────────────────────────────────────────────┘
```

## Key Components

### CloudGateway

The `CloudGateway` is the entry point for hosting actors in cloud environments. It handles:
- Actor registration and exposure
- Incoming invocation routing
- State persistence coordination
- Service registry integration

```swift
import TrebuchetCloud

let gateway = CloudGateway(configuration: .init(
    host: "0.0.0.0",
    port: 8080,
    stateStore: myStateStore,
    registry: myRegistry
))

try await gateway.expose(myActor, as: "my-actor")
try await gateway.run()
```

### CloudProvider Protocol

The `CloudProvider` protocol abstracts cloud platform specifics:

```swift
public protocol CloudProvider: Sendable {
    associatedtype FunctionConfig: Sendable
    associatedtype DeploymentResult: CloudDeployment

    func deploy<A: DistributedActor>(
        _ actorType: A.Type,
        as actorID: String,
        config: FunctionConfig,
        factory: @Sendable (TrebuchetActorSystem) -> A
    ) async throws -> DeploymentResult

    func transport(for deployment: DeploymentResult) async throws -> any TrebuchetTransport
}
```

Implementations:
- **AWSProvider**: AWS Lambda + API Gateway
- **GCPProvider**: Google Cloud Functions
- **AzureProvider**: Azure Functions
- **LocalProvider**: Local development

### ServiceRegistry

The `ServiceRegistry` protocol enables actor discovery:

```swift
public protocol ServiceRegistry: Sendable {
    func register(
        actorID: String,
        endpoint: CloudEndpoint,
        metadata: [String: String],
        ttl: Duration?
    ) async throws

    func resolve(actorID: String) async throws -> CloudEndpoint?

    func deregister(actorID: String) async throws
}
```

Implementations:
- **CloudMapRegistry**: AWS Cloud Map
- **InMemoryRegistry**: Local development

### ActorStateStore

The `ActorStateStore` protocol provides state persistence:

```swift
public protocol ActorStateStore: Sendable {
    func load<State: Codable & Sendable>(
        for actorID: String,
        as type: State.Type
    ) async throws -> State?

    func save<State: Codable & Sendable>(
        _ state: State,
        for actorID: String
    ) async throws

    func delete(for actorID: String) async throws
}
```

Implementations:
- **DynamoDBStateStore**: AWS DynamoDB
- **PostgreSQLStateStore**: PostgreSQL
- **InMemoryStateStore**: Local development

## Stateful Actors

Actors that need persistent state implement `StatefulActor`:

```swift
import TrebuchetCloud

@Trebuchet
distributed actor ShoppingCart: StatefulActor {
    typealias PersistentState = CartState

    let stateStore: ActorStateStore
    var persistentState = CartState()

    init(actorSystem: TrebuchetActorSystem, stateStore: ActorStateStore) async throws {
        self.actorSystem = actorSystem
        self.stateStore = stateStore
        try await loadState(from: stateStore)
    }

    distributed func addItem(_ item: Item) async throws -> CartState {
        persistentState.items.append(item)
        try await saveState(to: stateStore)
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

## Actor-to-Actor Communication

Actors deployed to cloud functions can discover and invoke each other in two ways:

### 1. Via Transport (Client Resolution)

```swift
import TrebuchetCloud
import TrebuchetAWS

// From within an actor running in Lambda
@Trebuchet
distributed actor GameRoom {
    let client: TrebuchetCloudClient

    distributed func notifyLobby() async throws {
        // Resolve another actor via service registry
        let lobby = try await client.resolve(Lobby.self, id: "lobby")

        // Invoke it (triggers another Lambda invocation)
        try await lobby.onGameStarted(roomID: id.id)
    }
}
```

### 2. Via CloudGateway.process() (Direct Invocation)

**Added in v0.3.0** - For programmatic actor invocation without HTTP overhead:

```swift
import TrebuchetCloud

let gateway = CloudGateway(configuration: .init(
    stateStore: stateStore,
    registry: registry
))

// Register actors
try await gateway.expose(myActor, as: "my-actor")

// Programmatic invocation (actor-to-actor, no HTTP)
let envelope = InvocationEnvelope(
    callID: UUID(),
    actorID: TrebuchetActorID(id: "my-actor"),
    targetIdentifier: "myMethod(param:)",
    genericSubstitutions: [],
    arguments: [try JSONEncoder().encode("value")]
)

let response = await gateway.process(envelope)
```

**Use cases:**
- Lambda-to-Lambda communication without API Gateway overhead
- Local actor-to-actor calls in same process
- Testing actor methods programmatically
- Building custom transport layers

## Local Development

For local development, use the in-memory implementations:

```swift
import TrebuchetCloud

let gateway = CloudGateway.development(host: "localhost", port: 8080)
```

This creates a gateway with:
- `InMemoryStateStore` for state persistence
- `InMemoryRegistry` for service discovery
- Local HTTP server on specified port

### Local Development Example

```swift
import Trebuchet
import TrebuchetCloud

@main
struct DevServer {
    static func main() async throws {
        // Create local gateway
        let gateway = CloudGateway.development(host: "localhost", port: 8080)

        // Create actors
        let room = GameRoom(actorSystem: gateway.system)
        let lobby = Lobby(actorSystem: gateway.system)

        // Expose actors
        try await gateway.expose(room, as: "game-room")
        try await gateway.expose(lobby, as: "lobby")

        print("Local development server running on http://localhost:8080")
        try await gateway.run()
    }
}
```

## Deployment Workflow

1. **Define actors** with `@Trebuchet` macro
2. **Configure deployment** in `trebuchet.yaml`
3. **Deploy** with CLI: `trebuchet deploy --provider aws`
4. **Invoke** via HTTP endpoint or from other actors

## Configuration File

Create a `trebuchet.yaml` to configure deployment:

```yaml
name: my-game-server
version: "1"

defaults:
  provider: aws
  region: us-east-1
  memory: 512
  timeout: 30

actors:
  GameRoom:
    memory: 1024
    stateful: true
  Lobby:
    memory: 256

state:
  type: dynamodb

discovery:
  type: cloudmap
  namespace: my-game
```

## Middleware Support

CloudGateway supports middleware for cross-cutting concerns:

```swift
import TrebuchetCloud
import TrebuchetSecurity
import TrebuchetObservability

let gateway = CloudGateway(configuration: .init(
    middlewares: [
        ValidationMiddleware(...),      // Request validation
        AuthenticationMiddleware(...),  // Authentication
        AuthorizationMiddleware(...),   // Authorization
        RateLimitingMiddleware(...),    // Rate limiting
        TracingMiddleware(...)          // Distributed tracing
    ],
    stateStore: stateStore,
    registry: registry
))
```

## Benefits of Serverless Deployment

### Cost Efficiency
- Pay only for actual invocations
- No cost for idle time
- Automatic scaling to zero

### Scalability
- Automatic horizontal scaling
- Handle traffic spikes effortlessly
- No capacity planning needed

### Operational Simplicity
- No server management
- Automatic patching and updates
- Built-in high availability

### Developer Experience
- Deploy with single command
- Focus on business logic
- Infrastructure as code

## Best Practices

### Minimize Cold Start Latency
- Use provisioned concurrency for critical paths
- Keep dependencies minimal
- Use ARM64 for 20% faster cold starts

### State Management
- Use DynamoDB for shared state
- Cache frequently accessed data
- Implement proper TTL for cleanup

### Error Handling
- Implement retry logic for transient failures
- Use dead-letter queues for failed invocations
- Monitor error rates with CloudWatch

### Testing
- Test locally with development gateway
- Use integration tests with real cloud resources
- Implement canary deployments

## See Also

- AWS Lambda deployment guide for AWS-specific setup
- CLI guide for deployment commands
- Security guide for authentication and authorization
- Observability guide for logging and metrics
