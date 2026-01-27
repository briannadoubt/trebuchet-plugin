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
- `--provider`: Cloud provider (aws, gcp, azure, local)
- `--region`: Default AWS region
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
- `--provider`: Cloud provider (aws, gcp, azure)
- `--region`: Deployment region
- `--environment`: Environment name (production, staging, etc.)
- `--dry-run`: Preview deployment without making changes
- `--verbose`: Show detailed output
- `--config`: Path to config file (default: trebuchet.yaml)

**Examples:**

```bash
# Deploy to AWS in us-east-1
trebuchet deploy --provider aws --region us-east-1

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
- `--port`: HTTP server port (default: 8080)
- `--host`: Host to bind to (default: localhost)
- `--watch`: Auto-reload on file changes
- `--config`: Path to config file

**Example:**

```bash
trebuchet dev --port 8080 --watch
```

**Output:**

```
Starting local development server...

Discovering actors...
  ✓ GameRoom
  ✓ Lobby

Starting server on http://localhost:8080

Actors available:
  GameRoom: http://localhost:8080/actors/GameRoom
  Lobby: http://localhost:8080/actors/Lobby

Watching for changes...
Press Ctrl+C to stop
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
- `dynamodb`: AWS DynamoDB
- `postgresql`: PostgreSQL database
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

1. **Discovers actors** using SwiftSyntax
2. **Generates bootstrap code** for Lambda handler
3. **Builds for ARM64** using Docker
4. **Packages binary** with swift-lambda-runtime
5. **Generates Terraform** for infrastructure
6. **Applies infrastructure** changes
7. **Uploads Lambda package**

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
# 1. Run local development server with auto-reload
trebuchet dev --watch

# 2. Make changes to actors

# 3. Test changes locally

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
