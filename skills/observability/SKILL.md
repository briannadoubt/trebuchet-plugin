---
name: observability
description: Production-grade observability with structured logging, metrics collection (counters, gauges, histograms), distributed tracing, CloudWatch integration, and sensitive data redaction. Use when monitoring actors, debugging issues, or tracking performance in production.
---

# TrebuchetObservability

Production-grade observability for distributed actors.

## Overview

TrebuchetObservability provides comprehensive observability features:

- **Structured Logging**: Rich metadata with sensitive data redaction
- **Metrics Collection**: Counters, gauges, and histograms
- **Distributed Tracing**: Request tracking across actor boundaries
- **CloudWatch Integration**: Export to AWS CloudWatch

## Quick Start

```swift
import TrebuchetObservability
import TrebuchetCloud

// Configure logging
let logger = TrebuchetLogger(
    label: "game-server",
    configuration: .init(
        level: .info,
        sensitiveKeys: ["password", "token", "secret"]
    ),
    formatter: JSONFormatter()
)

// Configure metrics
let metrics = InMemoryMetricsCollector()

// Configure tracing
let spanExporter = InMemorySpanExporter()

// Inject into CloudGateway
let gateway = CloudGateway(configuration: .init(
    middlewares: [
        TracingMiddleware(exporter: spanExporter)
    ],
    loggingConfiguration: .init(
        level: .info,
        sensitiveKeys: ["password", "token"]
    ),
    metricsCollector: metrics,
    stateStore: stateStore,
    registry: registry
))

// All invocations are now logged, traced, and metered
```

## Structured Logging

High-performance structured logging with rich metadata.

### Basic Usage

```swift
import TrebuchetObservability

let logger = TrebuchetLogger(label: "my-component")

await logger.info("Server started", metadata: [
    "port": "8080",
    "environment": "production"
])
```

### Log Levels

```swift
let config = LoggingConfiguration(level: .info)
let logger = TrebuchetLogger(label: "app", configuration: config)

await logger.debug("Debug message")    // Filtered out
await logger.info("Info message")      // Logged
await logger.warning("Warning!")       // Logged
await logger.error("Error occurred")   // Logged
await logger.critical("Critical!")     // Logged
```

### Structured Metadata

```swift
await logger.info("User login", metadata: [
    "user_id": "12345",
    "ip_address": "192.168.1.1",
    "method": "oauth"
])
```

### Sensitive Data Redaction

```swift
let config = LoggingConfiguration(
    sensitiveKeys: ["password", "token", "secret", "api_key"]
)
let logger = TrebuchetLogger(label: "auth", configuration: config)

await logger.info("Authentication", metadata: [
    "username": "alice",
    "password": "secret123",  // Will be [REDACTED]
    "token": "abc123"         // Will be [REDACTED]
])
```

### Correlation IDs

Track related log messages:

```swift
let correlationID = UUID()

await logger.info("Request received", correlationID: correlationID)
await logger.info("Processing request", correlationID: correlationID)
await logger.info("Request completed", correlationID: correlationID)
```

### Log Formatters

#### JSON Formatter (Production)

```swift
let logger = TrebuchetLogger(
    label: "api",
    formatter: JSONFormatter(prettyPrint: false)
)

await logger.info("API call", metadata: ["endpoint": "/users"])
// Output: {"timestamp":"2026-01-27T...","level":"INFO","label":"api","message":"API call","metadata":{"endpoint":"/users"}}
```

#### Console Formatter (Development)

```swift
let logger = TrebuchetLogger(
    label: "app",
    formatter: ConsoleFormatter(colorEnabled: true)
)

await logger.info("Server running", metadata: ["port": "8080"])
// Output: [2026-01-27 19:00:00.000] INFO  app: Server running | port=8080
```

### Actor Logging

```swift
@Trebuchet
distributed actor GameRoom {
    let logger = TrebuchetLogger(label: "GameRoom")

    distributed func join(player: Player) async throws {
        await logger.info("Player joining", metadata: [
            "playerID": player.id,
            "roomID": id.id
        ])

        // ... handle join ...

        await logger.info("Player joined successfully")
    }
}
```

## Metrics Collection

Track performance and behavior with standard metrics.

### Counters

Track cumulative values that only increase:

```swift
let metrics = InMemoryMetricsCollector()

// Increment counter
await metrics.incrementCounter("page_views", by: 1, tags: ["page": "home"])

// With dimensions
await metrics.incrementCounter("requests", tags: [
    "method": "POST",
    "status": "200"
])
```

### Gauges

Track point-in-time values that can go up or down:

```swift
// Set gauge value
await metrics.recordGauge("active_connections", value: 42.0, tags: [:])

// Update over time
await metrics.recordGauge("memory_usage", value: 512.0, tags: ["region": "us-east"])
```

### Histograms

Track distributions of values (typically latencies):

```swift
// Record latencies
await metrics.recordHistogram("response_time", value: .milliseconds(100), tags: ["endpoint": "/api"])

// Get statistics
let histogram = await metrics.histogram("response_time")
if let stats = await histogram?.statistics(for: ["endpoint": "/api"]) {
    print("Mean: \(stats.mean)ms")
    print("P50: \(stats.p50)ms")
    print("P95: \(stats.p95)ms")
    print("P99: \(stats.p99)ms")
}
```

### Recording Metrics in Actors

```swift
@Trebuchet
distributed actor GameRoom {
    let metrics: MetricsCollector

    distributed func join(player: Player) async throws {
        // Increment join counter
        await metrics.incrementCounter("game.joins", tags: [
            "room": id.id
        ])

        let startTime = Date()

        // ... handle join ...

        // Record join latency
        let duration = Date().timeIntervalSince(startTime)
        await metrics.recordHistogram("game.join_latency", value: duration)

        // Update active player count
        await metrics.recordGauge("game.active_players", value: Double(players.count))
    }
}
```

### CloudWatch Reporter

Export metrics to AWS CloudWatch:

```swift
import TrebuchetObservability
import TrebuchetAWS

let cloudWatch = CloudWatchReporter(
    namespace: "Trebuchet/GameServer",
    region: "us-east-1"
)

// Record metrics
await cloudWatch.incrementCounter("Invocations", by: 1, dimensions: [
    "ActorType": "GameRoom",
    "Method": "join"
])

await cloudWatch.recordHistogram("Latency", value: 42.5, unit: .milliseconds, dimensions: [
    "ActorType": "GameRoom"
])
```

### Tag Cardinality Warning

Keep unique tag combinations under 1000 for optimal performance:

```swift
// ✅ Good tags (low cardinality)
["method": "GET", "status": "200"]           // ~10 × 10 = 100 combinations
["actor_type": "GameRoom", "operation": "join"]  // Bounded by code

// ❌ Bad tags (high cardinality)
["request_id": "uuid-here"]    // Unbounded (millions)
["user_id": "123"]             // Unbounded (scales with users)
["timestamp": "2026-01-27..."] // Unbounded (infinite)
```

## Distributed Tracing

Track requests across actor boundaries.

### Basic Usage

```swift
import TrebuchetObservability

@Trebuchet
distributed actor GameRoom {
    let spanExporter: any SpanExporter

    distributed func join(player: Player) async throws {
        // Create span for this operation
        var span = Span(
            context: TraceContext(),
            name: "GameRoom.join",
            kind: .server,
            startTime: Date()
        )

        // Add span attributes
        span.setAttribute("player.id", value: player.id)
        span.setAttribute("room.id", value: id.id)

        do {
            // ... handle join ...

            span.addEvent(SpanEvent(name: "Player validated", timestamp: Date()))

            // Mark success and export
            span.end(status: .ok)
            try await spanExporter.export([span])
        } catch {
            // Mark error
            span.setAttribute("error.type", value: String(describing: type(of: error)))
            span.end(status: .error)
            try? await spanExporter.export([span])
            throw error
        }
    }
}
```

### Automatic Tracing with Middleware

```swift
import TrebuchetCloud
import TrebuchetObservability

let spanExporter = InMemorySpanExporter()
let tracingMiddleware = TracingMiddleware(exporter: spanExporter)

let gateway = CloudGateway(configuration: .init(
    middlewares: [tracingMiddleware],
    stateStore: stateStore,
    registry: registry
))

// Gateway automatically:
// - Extracts or creates TraceContext
// - Creates spans for invocations
// - Propagates context across actors
// - Includes trace_id in all logs
```

### Trace Context Propagation

Trace context automatically propagates across actor boundaries:

```swift
// Client calls actor
let gameRoom = try client.resolve(GameRoom.self, id: "room-123")
try await gameRoom.join(player: me)

// TracingMiddleware:
// 1. Extracts or creates TraceContext
// 2. Creates Span for invocation
// 3. Propagates context in InvocationEnvelope
// 4. Server continues the trace

// All operations share the same trace_id
```

## CloudGateway Integration

CloudGateway automatically provides observability:

```swift
import TrebuchetCloud
import TrebuchetObservability

let logger = TrebuchetLogger(label: "gateway")
let metrics = InMemoryMetricsCollector()
let spanExporter = InMemorySpanExporter()

let gateway = CloudGateway(configuration: .init(
    middlewares: [TracingMiddleware(exporter: spanExporter)],
    loggingConfiguration: .init(level: .info),
    metricsCollector: metrics,
    stateStore: stateStore,
    registry: registry
))

// Gateway automatically tracks:
// - invocations.total (counter)
// - invocations.duration (histogram)
// - invocations.active (gauge)
// - invocations.errors (counter)
```

### Automatic Metrics

CloudGateway records:

```
invocations.total { actor_type, method, result }
invocations.duration { actor_type, method }
invocations.active
invocations.errors { actor_type, method, error_type }
invocations.payload_size { direction }
```

### Automatic Logging

```swift
// Invocation start
logger.info("Invocation started", metadata: [
    "trace_id": span.traceId,
    "actor_type": "GameRoom",
    "method": "join",
    "actor_id": "room-123"
])

// Invocation complete
logger.info("Invocation completed", metadata: [
    "trace_id": span.traceId,
    "duration_ms": 42.5,
    "result": "success"
])
```

## Best Practices

### Structured Metadata

Use consistent metadata keys:

```swift
// ✅ Consistent naming
await logger.info("Player action", metadata: [
    "player_id": player.id,
    "action_type": "join",
    "room_id": room.id
])

// ❌ Inconsistent naming
await logger.info("Player action", metadata: [
    "playerID": player.id,
    "actionType": "join",
    "room": room.id
])
```

### Log Levels

Use appropriate levels:

```swift
await logger.debug("Processing player input")     // Detailed diagnostics
await logger.info("Player joined room")            // General info
await logger.warning("Room near capacity")         // Potentially problematic
await logger.error("Failed to save state")         // Error conditions
await logger.critical("Database connection lost")  // Severe errors
```

### Metric Naming

Use consistent dot notation:

```swift
// ✅ Consistent
"game.joins"
"game.leaves"
"game.join_latency"

// ❌ Inconsistent
"gameJoins"
"game_leaves"
"JoinLatency"
```

### Trace Spans

Create spans for significant operations:

```swift
// ✅ Meaningful spans
var span = Span(context: traceContext, name: "GameRoom.join")
var span = Span(context: traceContext, name: "validatePlayer")
var span = Span(context: traceContext, name: "databaseQuery")

// ❌ Too granular
var span = Span(context: traceContext, name: "increment counter")
var span = Span(context: traceContext, name: "if statement")
```

## Performance

TrebuchetObservability is designed for production:

- **Async Logging**: Non-blocking log writes
- **Efficient Metrics**: In-memory aggregation with batched exports
- **Minimal Overhead**: <1ms per operation
- **Backpressure Handling**: Drops logs under extreme load

## Configuration Presets

### Development

```swift
let logger = TrebuchetLogger(
    label: "app",
    configuration: .development
)
// Debug logging, no redaction, console formatter
```

### Production

```swift
let logger = TrebuchetLogger(
    label: "app",
    configuration: .default
)
// Info logging, sensitive data redaction, JSON formatter
```

## See Also

- Security guide for audit logging
- Cloud deployment guide for CloudGateway integration
- AWS Lambda guide for CloudWatch integration
