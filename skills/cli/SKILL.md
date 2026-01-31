---
name: cli
description: Trebuchet CLI commands for initializing projects, deploying actors, checking status, local development, and infrastructure management. Use when users need to deploy, configure, or manage Trebuchet projects from the command line.
---

# Trebuchet CLI

Command-line interface for deploying and managing Trebuchet distributed actors.

## Overview

The `trebuchet` CLI provides tools for:
- Initializing new projects
- Deploying actors to cloud platforms
- Managing deployments
- Running local development servers
- Generating infrastructure configuration
- **Automatic Xcode project support** (since v0.4.0)
- **Intelligent dependency analysis** for minimal deployments

## Xcode Project Support

**Zero configuration required!** The CLI automatically detects and works with:
- Xcode projects (`.xcodeproj`)
- Xcode workspaces (`.xcworkspace`)
- Swift Package Manager projects (`Package.swift`)

### Automatic Detection

The CLI automatically:
1. Detects project type (Xcode vs SPM)
2. Extracts package names from `Package.swift` when available
3. Generates appropriate package manifests
4. Adapts behavior across all commands (`dev`, `deploy`, `generate server`)

### Automatic Dependency Analysis

The CLI uses SwiftSyntax to intelligently analyze actor dependencies:

**What it analyzes:**
- Actor method signatures (parameters and return types)
- Transitive dependencies (e.g., `PlayerInfo` → `GameStatus`)
- Complex types: generics (`Array<T>`), optionals (`T?`), nested types
- Only types actually used by actors (symbol-scoped analysis)

**What it filters out:**
- Standard library types (`String`, `Int`, `UUID`, etc.)
- Unrelated types in the same file as dependencies
- Files that actors don't depend on

**Result:** Only 1-5 files typically copied vs. entire app

### Cascade Prevention

The CLI prevents copying your entire codebase:
- **Symbol-level analysis**: Only analyzes types actually used by actors
- **Smart filtering**: Doesn't cascade through unrelated dependencies
- **20-60x improvement** over naive file-level analysis

Example: If your actor uses `PlayerInfo` from `Models.swift`, but `Models.swift` also contains `UnrelatedType`, the CLI won't cascade to `UnrelatedType`'s dependencies.

### Works Everywhere

Xcode support works across all commands:

```bash
# Local development - zero config
trebuchet dev

# Generate server package - automatic dependency copying
trebuchet generate server --output ./my-server

# Deploy to cloud - works with Xcode projects
trebuchet deploy --provider aws
```

## Installation

### Using Mint (Recommended)

```bash
mint install briannadoubt/Trebuchet
```

### Build from Source

```bash
git clone https://github.com/briannadoubt/Trebuchet.git
cd Trebuchet
swift build -c release --product trebuchet
cp .build/release/trebuchet /usr/local/bin/
```

### Verify Installation

```bash
trebuchet --version
```

## Commands

### trebuchet init

Initialize a new Trebuchet project configuration.

```bash
trebuchet init --name my-project --provider aws
```

**Options:**
- `--name` (required): Project name
- `--provider`: Cloud provider (aws, fly)
- `--region`: Default deployment region
- `--output`: Output file path (default: trebuchet.yaml)

**Example:**

```bash
trebuchet init --name game-server --provider aws --region us-east-1
```

Creates `trebuchet.yaml`:

```yaml
name: game-server
version: "1"

defaults:
  provider: aws
  region: us-east-1
  memory: 512
  timeout: 30

actors: {}
state:
  type: dynamodb
discovery:
  type: cloudmap
```

### trebuchet deploy

Deploy actors to the cloud.

```bash
trebuchet deploy --provider aws --region us-east-1
```

**Options:**
- `--provider`: Cloud provider (aws, fly)
- `--region`: Deployment region
- `--environment`: Environment name (production, staging, etc.)
- `--dry-run`: Preview deployment without making changes
- `--verbose`: Show detailed output
- `--config`: Path to config file (default: trebuchet.yaml)

**Examples:**

```bash
# Deploy to AWS in us-east-1
trebuchet deploy --provider aws --region us-east-1

# Deploy to Fly.io
trebuchet deploy --provider fly --region iad

# Dry run to preview changes
trebuchet deploy --dry-run --verbose

# Deploy to staging environment
trebuchet deploy --environment staging

# Use custom config file
trebuchet deploy --config custom.yaml
```

**Output:**

```
Discovering actors...
  ✓ GameRoom (Sources/Actors/GameRoom.swift)
  ✓ Lobby (Sources/Actors/Lobby.swift)

Building for Lambda (arm64)...
  Building executable...
  Packaging bootstrap...
  ✓ Package built (14.2 MB)

Deploying to AWS...
  Creating Lambda function...
  ✓ Lambda: arn:aws:lambda:us-east-1:123:function:game-server
  Creating API Gateway...
  ✓ API Gateway: https://abc123.execute-api.us-east-1.amazonaws.com
  Creating DynamoDB table...
  ✓ DynamoDB: game-server-actor-state
  Creating CloudMap namespace...
  ✓ CloudMap: game-server namespace

Deployment complete!
Endpoint: https://abc123.execute-api.us-east-1.amazonaws.com
```

### trebuchet status

Check the status of deployed actors.

```bash
trebuchet status
```

**Options:**
- `--verbose`: Show detailed information
- `--provider`: Filter by provider
- `--config`: Path to config file

**Example:**

```bash
trebuchet status --verbose
```

**Output:**

```
Project: game-server
Provider: aws
Region: us-east-1

Actors:
  GameRoom
    Status: Active
    Memory: 1024 MB
    Timeout: 60s
    Last Update: 2026-01-27 10:30:00 UTC
    Invocations (24h): 1,234

  Lobby
    Status: Active
    Memory: 256 MB
    Timeout: 30s
    Last Update: 2026-01-27 10:30:00 UTC
    Invocations (24h): 567

Infrastructure:
  Lambda: arn:aws:lambda:us-east-1:123:function:game-server
  API Gateway: https://abc123.execute-api.us-east-1.amazonaws.com
  DynamoDB: game-server-actor-state
  CloudMap: game-server
```

### trebuchet undeploy

Remove deployed infrastructure.

```bash
trebuchet undeploy
```

**Options:**
- `--provider`: Cloud provider
- `--region`: Deployment region
- `--force`: Skip confirmation prompt
- `--config`: Path to config file

**Example:**

```bash
trebuchet undeploy --force
```

**Output:**

```
⚠️  This will delete all infrastructure and data!

Removing infrastructure...
  ✓ Lambda function deleted
  ✓ API Gateway deleted
  ✓ DynamoDB table deleted
  ✓ CloudMap namespace deleted

Undeployment complete.
```

### trebuchet dev

Run actors locally for development.

```bash
trebuchet dev --port 8080
```

**Options:**
- `--port` / `-p`: HTTP server port (default: 8080)
- `--host` / `-h`: Host to bind to (default: localhost)
- `--verbose` / `-v`: Enable verbose build output

**Examples:**

```bash
# Start on default port (8080)
trebuchet dev

# Custom port and host
trebuchet dev --port 3000 --host 0.0.0.0

# Verbose output
trebuchet dev --verbose
```

**What it does:**
1. Discovers all `@Trebuchet` actors in your project
2. Builds your project with `swift build` (skipped for Xcode projects)
3. Analyzes actor dependencies and copies only required files
4. Generates a local development runner in `.trebuchet/`
5. Starts an HTTP server using `CloudGateway.development()`
6. Exposes actors at `/invoke` endpoint
7. Provides health check at `/health` endpoint

**Note:** For Xcode projects, the CLI skips `swift build` since Xcode handles compilation. The dev server uses the generated package in `.trebuchet/` instead.

**Output:**

```
Starting local development server...

Found actors:
  • GameRoom
  • Lobby

Building project...
✓ Build succeeded

Generating development server...
✓ Runner generated

Starting server on localhost:8080...

Starting local development server...

  ✓ Exposed: GameRoom
  ✓ Exposed: Lobby

Server running on http://localhost:8080
Health check: http://localhost:8080/health
Invocation endpoint: http://localhost:8080/invoke

Press Ctrl+C to stop
```

### trebuchet generate

Generate deployment artifacts.

#### trebuchet generate server

Generate a standalone server package from your actors.

```bash
trebuchet generate server --output ./my-server
```

**Options:**
- `--output`: Output directory for generated server (default: .trebuchet/server)
- `--config`: Path to trebuchet.yaml
- `--verbose` / `-v`: Enable verbose output
- `--force`: Force regeneration even if server exists

**Examples:**

```bash
# Generate server in default location
trebuchet generate server

# Generate in custom directory
trebuchet generate server --output ./deploy/server

# Force regeneration
trebuchet generate server --force --verbose
```

**What it does:**
1. Discovers all `@Trebuchet` actors
2. Generates a standalone Swift package
3. Creates Package.swift with dependencies
4. Generates main.swift with actor bootstrapping
5. Ready to deploy or run independently

**Output:**

```
Loading configuration...

Discovering actors...
  ✓ GameRoom
  ✓ Lobby

Generating server package...
  ✓ Package manifest created
  ✓ Bootstrap code generated
  ✓ Server configured

✓ Server package generated at .trebuchet/server

To deploy:
  trebuchet deploy

To run locally:
  cd .trebuchet/server
  swift run
```

## Configuration File (trebuchet.yaml)

### Basic Structure

```yaml
name: my-project
version: "1"

defaults:
  provider: aws
  region: us-east-1
  memory: 512
  timeout: 30

actors: {}
environments: {}
state: {}
discovery: {}
```

### Actor Configuration

```yaml
actors:
  GameRoom:
    memory: 1024          # Memory in MB
    timeout: 60           # Timeout in seconds
    stateful: true        # Enable state persistence
    isolated: true        # Dedicated Lambda
    environment:          # Environment variables
      LOG_LEVEL: debug
      MAX_PLAYERS: "10"

  Lobby:
    memory: 256
    timeout: 30
```

### Environment Configuration

```yaml
environments:
  production:
    region: us-west-2
    memory: 2048
    environment:
      LOG_LEVEL: warn
      ENABLE_METRICS: "true"

  staging:
    region: us-east-1
    memory: 512
    environment:
      LOG_LEVEL: debug
```

### State Configuration

```yaml
state:
  type: dynamodb
  tableName: custom-table-name    # Optional
```

Supported types:
- `dynamodb`: AWS DynamoDB (for AWS deployments)
- `postgresql` or `postgres`: PostgreSQL database (for Fly.io or custom deployments)
- `memory`: In-memory (development only)

### Discovery Configuration

```yaml
discovery:
  type: cloudmap
  namespace: my-namespace
```

Supported types:
- `cloudmap`: AWS Cloud Map
- `memory`: In-memory (development only)

### WebSocket Configuration

```yaml
websocket:
  enabled: true
  stage: production
  routes:
    - $connect
    - $disconnect
    - $default

connections:
  type: dynamodb
  table: my-app-connections
```

## Actor Discovery

The CLI automatically discovers actors in your codebase:

```swift
// Sources/Actors/GameRoom.swift
@Trebuchet
distributed actor GameRoom {
    distributed func join(player: Player) -> RoomState
}
```

The CLI scans:
- `Sources/` directory
- All `.swift` files
- Actors marked with `@Trebuchet`
- Actor annotations in comments

## Build Process

When deploying to AWS Lambda, the CLI:

1. **Detects project type** (Xcode vs SPM)
2. **Discovers actors** using SwiftSyntax
3. **Analyzes dependencies** to identify required types
4. **Copies actor files** and dependencies (1-5 files typically)
5. **Generates bootstrap code** for Lambda handler
6. **Builds for ARM64** using Docker
7. **Packages binary** with swift-lambda-runtime
8. **Generates Terraform** for infrastructure
9. **Applies infrastructure** changes
10. **Uploads Lambda package**

### Dependency Analysis Performance

- **Analysis time:** 50-100ms for typical projects
- **Cascade prevention:** Avoids copying 200+ unnecessary files in worst case
- **File copying:** Only 1-5 files typically copied vs entire app

### Docker Build

The CLI uses Docker to cross-compile for Lambda ARM64:

```dockerfile
FROM swift:latest
RUN swift build -c release --arch arm64
```

## Terraform Integration

The CLI generates Terraform configuration in `.trebuchet/terraform/`:

```
.trebuchet/
└── terraform/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── lambda.zip
```

### Customizing Terraform

You can customize the generated Terraform:

```bash
cd .trebuchet/terraform
terraform plan -var="lambda_memory=2048"
terraform apply
```

### Using Existing Terraform

Skip CLI-generated Terraform and use your own:

```bash
trebuchet deploy --skip-terraform
```

## Common Workflows

### Initial Setup

```bash
# 1. Initialize project
trebuchet init --name my-project --provider aws

# 2. Edit trebuchet.yaml to configure actors
vi trebuchet.yaml

# 3. Test locally
trebuchet dev --port 8080

# 4. Deploy to staging
trebuchet deploy --environment staging

# 5. Deploy to production
trebuchet deploy --environment production
```

### Iterative Development

```bash
# 1. Run local development server
trebuchet dev --port 8080

# 2. Make changes to actors

# 3. Restart dev server to test changes
# (Ctrl+C to stop, then run trebuchet dev again)

# 4. Deploy to staging
trebuchet deploy --environment staging

# 5. Verify in staging
trebuchet status --environment staging

# 6. Deploy to production
trebuchet deploy --environment production
```

### Debugging Deployment Issues

```bash
# Preview changes without applying
trebuchet deploy --dry-run --verbose

# Check current deployment status
trebuchet status --verbose

# View generated Terraform
cat .trebuchet/terraform/main.tf

# Manually apply Terraform
cd .trebuchet/terraform
terraform plan
terraform apply
```

## Environment Variables

The CLI respects these environment variables:

- `AWS_ACCESS_KEY_ID`: AWS access key
- `AWS_SECRET_ACCESS_KEY`: AWS secret key
- `AWS_REGION`: Default AWS region
- `AWS_PROFILE`: AWS credentials profile
- `TREBUCHET_CONFIG`: Path to config file

## Exit Codes

- `0`: Success
- `1`: General error
- `2`: Configuration error
- `3`: Build error
- `4`: Deployment error
- `5`: Connection error

## Troubleshooting

### Actor Discovery Fails

**Problem**: CLI doesn't find actors

**Solutions**:
- Ensure actors are marked with `@Trebuchet`
- Check actors are in `Sources/` directory
- Verify Swift files are valid syntax
- Use `--verbose` flag to see discovery details

### Build Fails

**Problem**: Docker build fails

**Solutions**:
- Ensure Docker is running
- Check Docker has sufficient memory (4GB+)
- Verify internet connection for package downloads
- Use `--verbose` to see full build output

### Deployment Fails

**Problem**: Terraform apply fails

**Solutions**:
- Check AWS credentials are valid
- Verify IAM permissions
- Check region is correct
- Review `.trebuchet/terraform/` for errors
- Use `trebuchet deploy --dry-run` first

### Connection Timeout

**Problem**: CLI can't connect to AWS

**Solutions**:
- Verify internet connection
- Check AWS credentials
- Try different region
- Check firewall settings

## See Also

- Cloud deployment guide for deployment concepts
- AWS Lambda guide for AWS-specific deployment
- Configuration reference for trebuchet.yaml options
