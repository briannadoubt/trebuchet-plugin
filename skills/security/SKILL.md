---
name: security
description: Production-grade security with authentication (API keys, JWT), authorization (RBAC), rate limiting (token bucket, sliding window), request validation, and middleware integration. Use when securing actors, implementing access control, or preventing abuse.
---

# TrebuchetSecurity

Production-grade security for distributed actors.

## Overview

TrebuchetSecurity provides comprehensive security features for Trebuchet distributed actors:

- **Authentication**: API keys and JWT tokens
- **Authorization**: Role-based access control (RBAC)
- **Rate Limiting**: Token bucket and sliding window algorithms
- **Request Validation**: Payload size limits and input validation

## Quick Start

```swift
import TrebuchetCloud
import TrebuchetSecurity

// Create security components
let apiAuth = APIKeyAuthenticator(keys: [
    .init(key: "sk_live_abc123", principalId: "service-1", roles: ["service"])
])

let policy = RoleBasedPolicy(rules: [
    .adminFullAccess,
    .userReadOnly
])

let limiter = TokenBucketLimiter(
    requestsPerSecond: 100,
    burstSize: 200
)

let validator = RequestValidator(configuration: .strict)

// Create middleware stack
let authMiddleware = AuthenticationMiddleware(provider: apiAuth)
let authzMiddleware = AuthorizationMiddleware(policy: policy)
let rateLimitMiddleware = RateLimitingMiddleware(limiter: limiter)
let validationMiddleware = ValidationMiddleware(validator: validator)

// Configure gateway with full security
let gateway = CloudGateway(configuration: .init(
    middlewares: [
        validationMiddleware,   // Validate first
        authMiddleware,         // Authenticate
        authzMiddleware,        // Authorize
        rateLimitMiddleware     // Rate limit
    ],
    stateStore: stateStore,
    registry: registry
))
```

## Authentication

Verify user and service identities.

### API Key Authentication

Simple, secure authentication for services:

```swift
import TrebuchetSecurity

// Configure API keys
let apiAuth = APIKeyAuthenticator(keys: [
    .init(
        key: "sk_live_abc123",
        principalId: "service-worker-1",
        roles: ["service", "worker"]
    ),
    .init(
        key: "sk_admin_xyz789",
        principalId: "admin-bot",
        roles: ["admin"],
        expiresAt: Date().addingTimeInterval(86400) // 24 hours
    )
])

// Authenticate a request
let credentials = Credentials.apiKey(key: "sk_live_abc123")
let principal = try await apiAuth.authenticate(credentials)

print(principal.id) // "service-worker-1"
print(principal.roles) // ["service", "worker"]
```

### Dynamic Key Management

```swift
// Create authenticator
let apiAuth = APIKeyAuthenticator()

// Register new keys at runtime
await apiAuth.register(.init(
    key: "sk_new_key",
    principalId: "new-service",
    roles: ["service"]
))

// Revoke compromised keys
await apiAuth.revoke("sk_old_key")
```

### JWT Authentication

**Security Warning**: The included JWT implementation does NOT validate signatures and is for testing only.

For production, use a proper JWT library:
- [swift-jwt](https://github.com/Kitura/Swift-JWT)
- [JWTKit](https://github.com/vapor/jwt-kit)

```swift
// Configure JWT validator (testing only)
let jwtAuth = JWTAuthenticator(configuration: .init(
    issuer: "https://auth.example.com",
    audience: "https://api.example.com",
    clockSkew: 60 // Allow 60 seconds clock drift
))

// Authenticate bearer token
let credentials = Credentials.bearer(token: jwtToken)
let principal = try await jwtAuth.authenticate(credentials)
```

### Principal

The `Principal` struct represents an authenticated identity:

```swift
public struct Principal: Sendable, Codable {
    public let id: String                    // Unique identifier
    public let type: PrincipalType           // user, service, system
    public let roles: Set<String>            // Assigned roles
    public let attributes: [String: String]  // Custom attributes
    public let authenticatedAt: Date         // Authentication time
    public let expiresAt: Date?              // Optional expiration
}

// Check roles
if principal.hasRole("admin") {
    // Allow admin action
}

if principal.hasAnyRole(["admin", "moderator"]) {
    // Allow privileged action
}
```

## Authorization

Control access to actors and methods with RBAC.

### Basic RBAC

```swift
import TrebuchetSecurity

// Define access rules
let policy = RoleBasedPolicy(rules: [
    // Admins can do anything
    .init(role: "admin", actorType: "*", method: "*"),

    // Users can only read
    .init(role: "user", actorType: "*", method: "get*"),

    // Workers can invoke GameRoom
    .init(role: "worker", actorType: "GameRoom", method: "*")
])

// Check authorization
let action = Action(actorType: "GameRoom", method: "join")
let resource = Resource(type: "game", id: "room-123")

let allowed = try await policy.authorize(
    principal,
    action: action,
    resource: resource
)

if !allowed {
    throw AuthorizationError.accessDenied
}
```

### Pattern Matching

Rules support wildcard patterns:

```swift
// Prefix matching: methods starting with "get"
.init(role: "user", actorType: "*", method: "get*")

// Suffix matching: methods ending with "Status"
.init(role: "monitor", actorType: "*", method: "*Status")

// Exact match: specific actor and method
.init(role: "admin", actorType: "AdminPanel", method: "reset")

// Wildcard: all actors and methods
.init(role: "admin", actorType: "*", method: "*")
```

### Predefined Rules

```swift
let policy = RoleBasedPolicy(rules: [
    .adminFullAccess,  // Admin has full access
    .userReadOnly,     // User can only read
    .serviceInvoke     // Service can invoke any method
])
```

### Multi-Role Example

```swift
let gamePolicy = RoleBasedPolicy(rules: [
    // Admins can do everything
    .init(role: "admin", actorType: "*", method: "*"),

    // Players can join and leave games
    .init(role: "player", actorType: "GameRoom", method: "join"),
    .init(role: "player", actorType: "GameRoom", method: "leave"),

    // Premium players can create private rooms
    .init(role: "premium", actorType: "GameRoom", method: "create"),

    // Moderators can kick players
    .init(role: "moderator", actorType: "GameRoom", method: "kick"),

    // All users can view lobbies
    .init(role: "user", actorType: "Lobby", method: "get*")
])
```

## Rate Limiting

Protect actors from abuse.

### Token Bucket Limiter

Allow controlled bursts while maintaining average rate:

```swift
import TrebuchetSecurity

let limiter = TokenBucketLimiter(
    requestsPerSecond: 100,  // Average rate
    burstSize: 200           // Allow bursts up to 200
)

// Check rate limit
let result = try await limiter.checkLimit(key: principal.id)

if !result.allowed {
    throw RateLimitError.limitExceeded(retryAfter: result.retryAfter!)
}
```

### Sliding Window Limiter

Precise per-window limits with smooth transitions:

```swift
let limiter = SlidingWindowLimiter(
    maxRequests: 1000,
    windowSize: .seconds(60)
)

let result = try await limiter.checkLimit(key: principal.id)
```

### Rate Limit Result

```swift
public struct RateLimitResult {
    let allowed: Bool           // Whether request is allowed
    let remaining: Int          // Remaining requests
    let retryAfter: Duration?   // When to retry if denied
}
```

## Request Validation

Validate incoming requests.

### Configuration

```swift
let validator = RequestValidator(configuration: .init(
    maxPayloadSize: 1024 * 1024,        // 1MB
    maxActorIDLength: 256,
    maxMethodNameLength: 128,
    allowedMethodNamePattern: "[a-zA-Z][a-zA-Z0-9_]*"
))

// Validate request
try await validator.validate(invocation)
```

### Preset Configurations

```swift
// Strict validation (recommended for production)
let validator = RequestValidator(configuration: .strict)

// Lenient validation (for development)
let validator = RequestValidator(configuration: .lenient)
```

## Middleware Integration

Security components integrate via middleware:

```swift
import TrebuchetCloud
import TrebuchetSecurity

// Middleware executes in order
let middlewares: [CloudMiddleware] = [
    ValidationMiddleware(validator: validator),     // 1. Validate request
    AuthenticationMiddleware(provider: apiAuth),    // 2. Authenticate
    AuthorizationMiddleware(policy: policy),        // 3. Authorize
    RateLimitingMiddleware(limiter: limiter)        // 4. Rate limit
]

let gateway = CloudGateway(configuration: .init(
    middlewares: middlewares,
    stateStore: stateStore,
    registry: registry
))
```

## Security Flow

```
1. Request arrives
   ↓
2. ValidationMiddleware validates payload
   ↓
3. AuthenticationMiddleware verifies credentials → Principal
   ↓
4. AuthorizationMiddleware checks permissions
   ↓
5. RateLimitingMiddleware checks rate limits
   ↓
6. Request processed by actor
```

## Security Presets

### Development

Permissive settings for local development:

```swift
let devAuth = APIKeyAuthenticator(keys: [
    .init(key: "dev_test", principalId: "dev", roles: ["admin"])
])

let devPolicy = RoleBasedPolicy(rules: [
    .init(role: "admin", actorType: "*", method: "*")
], denyByDefault: false)

let devLimiter = TokenBucketLimiter(
    requestsPerSecond: 1000,
    burstSize: 2000
)

let devValidator = RequestValidator(configuration: .init(
    maxPayloadSize: 10 * 1024 * 1024,  // 10MB
    maxActorIDLength: 1000,
    maxMethodNameLength: 500
))
```

### Production

Strict settings for production:

```swift
let prodAuth = JWTAuthenticator(configuration: .init(
    issuer: "https://auth.example.com",
    audience: "https://api.example.com"
))

let prodPolicy = RoleBasedPolicy(rules: [
    .adminFullAccess,
    .userReadOnly
])

let prodLimiter = TokenBucketLimiter(
    requestsPerSecond: 10,
    burstSize: 20
)

let prodValidator = RequestValidator(configuration: .strict)
```

## Error Handling

```swift
do {
    try await processSecureRequest()
} catch AuthenticationError.invalidCredentials {
    return .unauthorized("Invalid credentials")
} catch AuthenticationError.expired {
    return .unauthorized("Credentials expired")
} catch AuthorizationError.accessDenied {
    return .forbidden("Access denied")
} catch RateLimitError.limitExceeded(let retryAfter) {
    return .tooManyRequests(retryAfter: retryAfter)
} catch ValidationError.payloadTooLarge(let size, let limit) {
    return .requestTooLarge("Payload \(size) exceeds limit \(limit)")
}
```

## Best Practices

### Defense in Depth

Use multiple security layers:

```swift
// ✅ Multiple security checks
let middleware = [
    ValidationMiddleware(...),      // Validate inputs
    AuthenticationMiddleware(...),  // Verify identity
    AuthorizationMiddleware(...),   // Check permissions
    RateLimitingMiddleware(...)     // Prevent abuse
]
```

### Least Privilege

Grant minimum necessary permissions:

```swift
// ✅ Specific permissions
.init(role: "user", actorType: "GameRoom", method: "join")

// ❌ Overly broad permissions
.init(role: "user", actorType: "*", method: "*")
```

### Audit Logging

Log all security decisions:

```swift
logger.info("Authorization decision", metadata: [
    "principal": principal.id,
    "action": "\(action.actorType).\(action.method)",
    "resource": resource.id,
    "allowed": "\(allowed)"
])
```

### Secure Defaults

Start strict, relax as needed:

```swift
// ✅ Start strict
let validator = RequestValidator(configuration: .strict)

// ❌ Start permissive
let validator = RequestValidator(configuration: .lenient)
```

## Custom Providers

Implement custom authentication:

```swift
actor CustomAuthenticator: AuthenticationProvider {
    func authenticate(_ credentials: Credentials) async throws -> Principal {
        guard case .custom(let type, let value) = credentials,
              type == "device-token" else {
            throw AuthenticationError.malformed(reason: "Expected device token")
        }

        let deviceInfo = try await validateDeviceToken(value)

        return Principal(
            id: deviceInfo.deviceId,
            type: .system,
            roles: ["device", "iot"]
        )
    }
}
```

Implement custom authorization:

```swift
actor TimeBasedPolicy: AuthorizationPolicy {
    let allowedHours: ClosedRange<Int>

    func authorize(
        _ principal: Principal,
        action: Action,
        resource: Resource
    ) async throws -> Bool {
        let hour = Calendar.current.component(.hour, from: Date())
        guard allowedHours.contains(hour) else {
            return false
        }
        return principal.hasRole("admin") || principal.hasRole("user")
    }
}
```

## See Also

- Cloud deployment guide for CloudGateway integration
- Observability guide for security event logging
- AWS Lambda guide for production deployment
