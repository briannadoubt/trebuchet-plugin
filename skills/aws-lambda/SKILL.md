---
name: aws-lambda
description: AWS Lambda deployment with trebuchet CLI, DynamoDB state storage, CloudMap service discovery, WebSocket streaming via API Gateway, Lambda handler implementation, connection management, cost optimization, and production configuration. Use when deploying to AWS or implementing WebSocket realtime features.
---

# Deploying to AWS Lambda

Deploy distributed actors to AWS Lambda with the trebuchet CLI and enable realtime WebSocket streaming.

## Overview

Trebuchet provides streamlined AWS Lambda deployment with:
- Automated infrastructure provisioning (Lambda, API Gateway, DynamoDB)
- Cross-compilation for Lambda ARM64
- WebSocket support for realtime streaming
- DynamoDB Streams for multi-instance coordination
- CloudMap service discovery for actor-to-actor communication

## Prerequisites

Before deploying, ensure you have:

1. **AWS CLI** configured with appropriate credentials
2. **Docker** installed (for cross-compilation)
3. **Terraform** installed (optional, for infrastructure management)

```bash
# Verify prerequisites
aws sts get-caller-identity
docker --version
terraform --version
```

## Quick Start

### 1. Initialize Configuration

```bash
trebuchet init --name my-game-server --provider aws
```

This creates `trebuchet.yaml`:

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

### 2. Deploy to AWS

```bash
trebuchet deploy --provider aws --region us-east-1
```

Output:
```
Discovering actors...
  ✓ GameRoom
  ✓ Lobby

Building for Lambda (arm64)...
  ✓ Package built (14.2 MB)

Deploying to AWS...
  ✓ Lambda: arn:aws:lambda:us-east-1:123:function:my-game-actors
  ✓ API Gateway: https://abc123.execute-api.us-east-1.amazonaws.com
  ✓ DynamoDB: my-game-actor-state
  ✓ CloudMap: my-game namespace

Ready! Actors can discover each other automatically.
```

## Production-Ready Features (v0.3.0+)

Trebuchet's AWS integration is built on the official **Soto SDK** (AWS SDK for Swift) for production reliability:

### Soto AWS SDK Integration

- **DynamoDB**: Actor state persistence with optimistic locking
- **CloudWatch**: Metrics and observability
- **Cloud Map**: Service discovery
- **Lambda**: Serverless actor deployment
- **IAM**: Role management
- **API Gateway WebSocket**: Connection management

### State Versioning

Actor state uses optimistic concurrency control to prevent lost writes:

```swift
@Trebuchet
distributed actor Counter: StatefulActor {
    typealias PersistentState = CounterState

    var persistentState = CounterState()

    distributed func increment() async throws -> Int {
        persistentState.count += 1
        persistentState.version += 1  // Automatic version tracking
        return persistentState.count
    }
}

struct CounterState: Codable, Sendable {
    var count: Int = 0
    var version: Int = 0
}
```

DynamoDB conditional updates ensure version conflicts are detected:
- Updates only succeed if the version matches
- Prevents concurrent writes from overwriting each other
- Automatic version increment on each state change

### Protocol Versioning

Client-server compatibility is handled automatically:
- Version negotiation for distributed systems
- Graceful degradation support
- Forward and backward compatibility

### Graceful Shutdown

Proper actor lifecycle management:
- Clean resource cleanup
- In-flight request completion
- Connection draining
- Zero data loss on Lambda shutdown

### TCP Transport for Multi-Instance

For multi-machine deployments (e.g., Fly.io), use TCP transport:

```swift
// Server-to-server communication
let transport = TrebuchetTransport.tcp(
    host: "actor-service.internal",
    port: 9001
)

let client = TrebuchetClient(transport: transport)
try await client.connect()
```

**TCP Transport Features:**
- Length-prefixed message framing (4-byte big-endian)
- Connection pooling with stale connection cleanup
- Idle timeout (5 minutes) to prevent resource leaks
- Backpressure handling with 30-second write timeout
- Optimized EventLoopGroup (2-4 threads for I/O)
- **Security**: Designed for trusted networks only (no TLS)

### LocalStack Integration Testing

Test your AWS integration locally with LocalStack:

```bash
# Start LocalStack with all services
docker-compose -f docker-compose.localstack.yml up -d

# Run integration tests
swift test --filter TrebuchetAWSTests

# Cleanup
docker-compose -f docker-compose.localstack.yml down -v
```

**LocalStack Services Simulated:**
- Lambda - function deployment/invocation
- DynamoDB - actor state persistence
- DynamoDB Streams - real-time state broadcasting
- Cloud Map - service discovery
- IAM - role management
- API Gateway WebSocket - connection management

**Test Features:**
- Graceful skipping when LocalStack unavailable
- Automatic test isolation via unique actor IDs
- Cleanup in defer blocks to prevent resource leaks
- Healthcheck verification before tests

## AWS Resources Created

The deployment creates:

| Resource | Purpose |
|----------|---------|
| Lambda Function | Hosts your actors |
| Lambda Function URL | HTTP endpoint for invocations |
| DynamoDB Table (State) | Actor state persistence |
| DynamoDB Table (Connections) | WebSocket connection tracking |
| CloudMap Namespace | Service discovery |
| IAM Role | Lambda execution permissions |
| CloudWatch Log Group | Logging |
| API Gateway WebSocket API | Realtime streaming connections |

## WebSocket Streaming on AWS

Trebuchet supports production-grade realtime streaming using:
- **API Gateway WebSocket API** for persistent connections
- **AWS Lambda** for serverless actor execution
- **DynamoDB** for connection tracking
- **DynamoDB Streams** for broadcasting state changes

### Architecture

```
Client (WebSocket)
    ↓
API Gateway WebSocket API
    ↓
Lambda (WebSocket Handler)
    ├─→ DynamoDB (Connection Table)
    └─→ Lambda (Actor Invocation)
        └─→ DynamoDB (Actor State Table)
            └─→ DynamoDB Stream
                └─→ Lambda (Stream Processor)
                    └─→ Broadcast to clients
```

### Configuration for WebSocket

Add WebSocket configuration to `trebuchet.yaml`:

```yaml
websocket:
  enabled: true
  stage: production
  routes:
    - $connect     # New connection handler
    - $disconnect  # Disconnection handler
    - $default     # Message router

connections:
  type: dynamodb
  table: my-app-connections
```

### Lambda Handler Implementation

```swift
import AWSLambdaRuntime
import Trebuchet
import TrebuchetCloud
import TrebuchetAWS

@main
struct WebSocketHandler: SimpleLambdaHandler {
    let handler: WebSocketLambdaHandler

    init(context: LambdaInitializationContext) async throws {
        // Configure state store
        let stateStore = DynamoDBStateStore(
            tableName: env("STATE_TABLE"),
            region: env("AWS_REGION") ?? "us-east-1"
        )

        // Configure connection storage
        let connectionStorage = DynamoDBConnectionStorage(
            tableName: env("CONNECTION_TABLE"),
            region: env("AWS_REGION") ?? "us-east-1"
        )

        // Configure connection sender (API Gateway Management API)
        let connectionSender = APIGatewayConnectionSender(
            endpoint: env("WEBSOCKET_ENDPOINT"),
            region: env("AWS_REGION") ?? "us-east-1"
        )

        // Create connection manager
        let connectionManager = ConnectionManager(
            storage: connectionStorage,
            sender: connectionSender
        )

        // Create cloud gateway
        let gateway = CloudGateway(configuration: .init(
            stateStore: stateStore,
            registry: CloudMapRegistry(namespace: env("NAMESPACE"))
        ))

        // Register actors
        let todoList = try await TodoList(
            actorSystem: gateway.system,
            stateStore: stateStore
        )
        try await gateway.expose(todoList, as: "todos")

        // Create WebSocket handler
        handler = WebSocketLambdaHandler(
            gateway: gateway,
            connectionManager: connectionManager
        )
    }

    func handle(
        _ event: APIGatewayWebSocketEvent,
        context: LambdaContext
    ) async throws -> APIGatewayWebSocketResponse {
        try await handler.handle(event)
    }
}
```

## Invoking Actors

### From External Clients

```swift
import Trebuchet

let client = TrebuchetClient(transport: .https(
    host: "abc123.execute-api.us-east-1.amazonaws.com"
))
try await client.connect()

let room = try client.resolve(GameRoom.self, id: "game-room")
let state = try await room.join(player: me)
```

### From Other Lambda Functions

```swift
import TrebuchetAWS

let client = TrebuchetCloudClient.aws(
    region: "us-east-1",
    namespace: "my-game-server"
)

let lobby = try await client.resolve(Lobby.self, id: "lobby")
let players = try await lobby.getPlayers()
```

## Configuration Options

### Actor Configuration

```yaml
actors:
  GameRoom:
    memory: 1024        # Memory in MB (128-10240)
    timeout: 60         # Timeout in seconds (1-900)
    stateful: true      # Enable state persistence
    isolated: true      # Run in dedicated Lambda
    environment:        # Environment variables
      LOG_LEVEL: debug
```

### Environment Overrides

```yaml
environments:
  production:
    region: us-west-2
    memory: 2048
  staging:
    region: us-east-1
```

Deploy to specific environment:

```bash
trebuchet deploy --environment production
```

## Managing Deployments

### Check Status

```bash
trebuchet status --verbose
```

### Undeploy

```bash
trebuchet undeploy
```

## Cost Analysis

AWS Lambda pricing is based on:
- **Requests**: $0.20 per 1M requests
- **Duration**: $0.0000166667 per GB-second (x86), $0.0000133334 (ARM64)

API Gateway WebSocket:
- **Connection Minutes**: $0.25 per million
- **Messages**: $1.00 per million (first 1 billion)

DynamoDB (On-Demand):
- **Read**: $0.25 per million reads
- **Write**: $1.25 per million writes

### Example: 10,000 Connections (1 hour)

```
API Gateway WebSocket:
  600,000 connection-minutes × $0.25/M = $0.15
  600,000 messages × $1.00/M = $0.60
  Total: $0.75

Lambda:
  620,000 invocations (under free tier)
  31,000 GB-seconds × $0.0000166667 = $0.52
  Total: $0.52

DynamoDB:
  620,000 writes × $1.25/M = $0.78
  600,000 reads × $0.25/M = $0.15
  Total: $0.93

CloudWatch: $0.50

Total: ~$2.70 per hour for 10,000 connections
```

### Production Costs (5,000 users 24/7)

**Monthly estimate: $762-$1,254**
- Per-user cost: **$0.15-$0.25/month**

With optimizations (provisioned DynamoDB, compression, caching):
- **$0.10-$0.15/month per user**

## Cost Optimization Strategies

### 1. Use Provisioned DynamoDB
50-70% reduction in DynamoDB costs for predictable workloads:

```yaml
state:
  type: dynamodb
  billing_mode: provisioned
  read_capacity: 500
  write_capacity: 1000
```

### 2. Use ARM64 Architecture
20% reduction in Lambda compute costs:

```yaml
defaults:
  architecture: arm64
```

### 3. Compress WebSocket Messages
60-80% reduction in bandwidth and message costs:

```swift
// Client-side compression
let compressed = try data.compressed(using: .zlib)
```

### 4. Cache Actor State
50% reduction in DynamoDB read costs:

```swift
// Global cache (persists across warm invocations)
private var stateCache: [String: (State, Date)] = [:]
```

### 5. Batch DynamoDB Operations
20-30% reduction in request costs:

```swift
// Use BatchWriteItem instead of individual writes
try await dynamoDB.batchWriteItem(items: connections)
```

## IAM Permissions

Required permissions for deployment:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:CreateFunction",
        "lambda:UpdateFunctionCode",
        "lambda:UpdateFunctionConfiguration",
        "lambda:GetFunction"
      ],
      "Resource": "arn:aws:lambda:*:*:function:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:CreateTable",
        "dynamodb:DescribeTable"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "servicediscovery:CreatePrivateDnsNamespace",
        "servicediscovery:CreateService"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole"
      ],
      "Resource": "arn:aws:iam::*:role/*"
    }
  ]
}
```

## Monitoring & Observability

### CloudWatch Metrics

Monitor these metrics:

```
AWS/ApiGateway:
  - ConnectCount
  - MessageCount
  - IntegrationLatency

AWS/Lambda:
  - Invocations
  - Duration
  - Errors
  - ConcurrentExecutions

AWS/DynamoDB:
  - ConsumedReadCapacityUnits
  - ConsumedWriteCapacityUnits
  - UserErrors (throttling)
```

### CloudWatch Logs

Lambda functions automatically log to:

```
/aws/lambda/my-app-websocket
/aws/lambda/my-app-stream-processor
```

## Troubleshooting

### Connections Not Persisting

**Problem**: WebSocket connects but immediately disconnects

**Solutions**:
- Check Lambda execution role has DynamoDB permissions
- Verify CONNECTION_TABLE environment variable is set
- Check CloudWatch logs for errors in $connect handler

### Messages Not Broadcasting

**Problem**: State changes don't reach clients

**Solutions**:
- Verify DynamoDB Streams is enabled on actor state table
- Check stream processor Lambda has EventSourceMapping
- Ensure API Gateway Management API permissions are granted
- Verify connection storage has correct actorId index

### High Latency

**Problem**: Updates take seconds to reach clients

**Solutions**:
- Enable Lambda Provisioned Concurrency
- Use DynamoDB Provisioned Capacity
- Check API Gateway POST-to-connection latency
- Consider regional deployment closer to users

### High Costs

**Problem**: AWS bill higher than expected

**Solutions**:
- Switch DynamoDB to Provisioned mode
- Implement message compression
- Add connection throttling and idle timeouts
- Cache frequently accessed actor state
- Use ARM64 for 20% compute savings

## Best Practices

### Cold Start Mitigation
- Use Provisioned Concurrency for critical paths
- Keep Lambda package size minimal
- Use ARM64 for faster cold starts

### State Management
- Implement proper TTL for DynamoDB records
- Use batch operations when possible
- Cache frequently accessed state

### Security
- Use IAM authentication for API Gateway
- Implement rate limiting
- Validate all inputs
- Use VPC endpoints for DynamoDB access

### Testing
- Test locally with development gateway first
- Use staging environment before production
- Implement integration tests with real AWS resources
- Monitor CloudWatch metrics and alarms

## See Also

- Cloud deployment guide for general concepts
- CLI guide for deployment commands
- Security guide for authentication
- Observability guide for logging and metrics
