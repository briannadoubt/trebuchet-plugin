# Trebuchet Skills

This directory contains skills that provide Trebuchet framework expertise to Claude Code.

## Available Skills

### Core Framework

- **getting-started** - Installation, setup, and basic usage of Trebuchet distributed actors. First actor definition, server/client patterns, and fundamental concepts.

- **distributed-actors** - Best practices for defining distributed actors with `@Trebuchet` macro, method requirements, stateful actors, and error handling patterns.

### Real-time Features

- **streaming** - Real-time state streaming with `@StreamedState`, `@ObservedActor`, stream resumption, filtering, and delta encoding for bandwidth optimization.

### Client Integration

- **swiftui-integration** - SwiftUI integration with observable connections, `@RemoteActor` property wrapper, connection state management, multi-server scenarios, and auto-reconnection.

### Cloud Deployment

- **cloud-deployment** - Serverless deployment architecture with CloudGateway, CloudProvider abstraction, ServiceRegistry, ActorStateStore, and multi-cloud strategies.

- **aws-lambda** - AWS Lambda deployment with CLI, DynamoDB state storage, CloudMap service discovery, WebSocket streaming, and cost optimization.

### Tools & CLI

- **cli** - Trebuchet CLI commands for initializing projects, deploying actors, checking status, local development server, and infrastructure management.

### Production Features

- **security** - Production-grade security with authentication (API keys, JWT), authorization (RBAC), rate limiting, and request validation.

- **observability** - Production-grade observability with structured logging, metrics collection, distributed tracing, and CloudWatch integration.

## How Skills Work

Claude Code automatically invokes relevant skills when:
- You ask questions about Trebuchet topics
- Claude encounters Trebuchet code in your project
- You explicitly request help with specific features

Skills provide:
- Comprehensive documentation
- Code examples and patterns
- Best practices
- Troubleshooting guidance
- Cross-references to related topics

## Skill Structure

Each skill is organized as:
```
skills/
├── skill-name/
│   └── SKILL.md          # Must be named exactly "SKILL.md"
```

The directory name becomes the skill identifier (e.g., `skills/getting-started/SKILL.md` → `/trebuchet:getting-started`).

## Using Skills

Skills are automatically available when the plugin is installed. Claude will invoke them as needed when you:

- Ask: "How do I get started with Trebuchet?"
- Ask: "How do I add streaming to my actor?"
- Ask: "Deploy my actors to AWS Lambda"
- Work with Trebuchet code in your project

You can also explicitly invoke the trebuchet-expert agent for comprehensive guidance: "Use the trebuchet-expert to help me design my actor system"
