# Trebuchet Plugin for Claude Code

Expert assistance for Trebuchet distributed actor framework development.

## What is Trebuchet?

[Trebuchet](https://github.com/briannadoubt/Trebuchet) is a Swift 6.2 distributed actor framework that makes RPC stupid simple. Define actors once and use them seamlessly whether they're local or remote, in-process or deployed to AWS Lambda.

## What This Plugin Provides

This plugin gives Claude Code deep expertise in Trebuchet through:

### 📚 Skills (9 total)
Documentation organized as discoverable skills that Claude automatically uses when relevant:

- **getting-started** - Installation, setup, basic actor patterns
- **distributed-actors** - Best practices for defining actors
- **streaming** - Real-time state synchronization with @StreamedState
- **swiftui-integration** - Reactive SwiftUI integration
- **cloud-deployment** - Serverless deployment architecture
- **aws-lambda** - AWS Lambda deployment guide
- **cli** - Trebuchet CLI commands and workflows
- **security** - Authentication, authorization, rate limiting
- **observability** - Logging, metrics, distributed tracing

### 🤖 Specialized Agent
The **trebuchet-expert** agent provides comprehensive guidance on:
- Setting up new projects
- Debugging distributed actor issues
- Designing actor architectures
- Configuring cloud deployments
- Optimizing performance and costs
- Implementing security and observability

Claude automatically delegates Trebuchet questions to this agent.

## Installation

### Using Claude Code CLI

```bash
# Clone the plugin repository
git clone https://github.com/briannadoubt/trebuchet-plugin

# Install locally
claude plugin add ./trebuchet-plugin
```

### Enable in Settings

Add to your Claude Code settings:

```json
{
  "enabledPlugins": {
    "trebuchet@briannadoubt": true
  }
}
```

## Usage

### Automatic Skill Invocation

Claude automatically uses relevant skills when you:

```
Ask: "How do I get started with Trebuchet?"
→ Invokes getting-started skill

Ask: "How do I add streaming to my actor?"
→ Invokes streaming skill

Ask: "Deploy my actors to AWS"
→ Invokes aws-lambda and cli skills
```

### Explicit Agent Invocation

For comprehensive guidance, explicitly invoke the expert agent:

```
"Use trebuchet-expert to help me design my actor system"
"Ask the trebuchet-expert about optimizing my deployment"
```

### Working with Your Code

When you're working in a Trebuchet project, Claude will:
- Automatically recognize Trebuchet code
- Use relevant skills for context
- Suggest best practices
- Help debug issues
- Assist with deployments

## Example Interactions

### Setting Up a New Project

**You**: "Help me create a new Trebuchet project for a real-time game server"

**Claude** (using getting-started and distributed-actors skills):
- Guides you through installation
- Helps define your actors
- Sets up server and client code
- Suggests architecture patterns

### Adding Streaming

**You**: "Add real-time state streaming to my GameRoom actor"

**Claude** (using streaming skill):
- Adds @StreamedState to your actor
- Shows SwiftUI integration with @ObservedActor
- Configures stream resumption
- Suggests filtering and delta encoding for optimization

### Deploying to AWS

**You**: "Deploy my actors to AWS Lambda"

**Claude** (using aws-lambda and cli skills):
- Creates or updates trebuchet.yaml
- Explains AWS resources
- Runs `trebuchet deploy --dry-run`
- Executes deployment
- Provides client connection instructions

### Adding Security

**You**: "Add API key authentication to my actors"

**Claude** (using security skill):
- Adds TrebuchetSecurity dependency
- Configures APIKeyAuthenticator
- Shows middleware integration
- Adds rate limiting

## Skills Reference

Each skill provides comprehensive documentation on its topic:

| Skill | Description |
|-------|-------------|
| **getting-started** | Installation, first actors, server/client setup |
| **distributed-actors** | @Trebuchet macro, method requirements, error handling |
| **streaming** | @StreamedState, @ObservedActor, resumption, filtering |
| **swiftui-integration** | Connection management, @RemoteActor, reconnection |
| **cloud-deployment** | CloudGateway, providers, service discovery, state stores |
| **aws-lambda** | AWS deployment, DynamoDB, CloudMap, WebSocket streaming |
| **cli** | trebuchet init/deploy/status/undeploy/dev commands |
| **security** | Authentication, authorization, rate limiting, validation |
| **observability** | Logging, metrics, tracing, CloudWatch integration |

## Plugin Structure

```
trebuchet-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   ├── SKILLS.md                # Skills index
│   ├── getting-started/
│   │   └── SKILL.md
│   ├── distributed-actors/
│   │   └── SKILL.md
│   ├── streaming/
│   │   └── SKILL.md
│   ├── swiftui-integration/
│   │   └── SKILL.md
│   ├── cloud-deployment/
│   │   └── SKILL.md
│   ├── aws-lambda/
│   │   └── SKILL.md
│   ├── cli/
│   │   └── SKILL.md
│   ├── security/
│   │   └── SKILL.md
│   └── observability/
│       └── SKILL.md
├── agents/
│   └── trebuchet-expert.md      # Specialized agent
├── README.md
└── LICENSE
```

## Contributing

Contributions are welcome! To add or improve skills:

1. Fork this repository
2. Create or edit skill files in `skills/`
3. Ensure each SKILL.md has proper frontmatter
4. Test with Claude Code
5. Submit a pull request

## Resources

- **Trebuchet GitHub**: https://github.com/briannadoubt/Trebuchet
- **Documentation**: https://briannadoubt.github.io/Trebuchet/documentation/trebuchet/
- **Plugin Issues**: https://github.com/briannadoubt/trebuchet-plugin/issues

## License

MIT License - See LICENSE file for details

## Support

For help with:
- **Trebuchet framework**: Open an issue on [Trebuchet repo](https://github.com/briannadoubt/Trebuchet/issues)
- **This plugin**: Open an issue on [trebuchet-plugin repo](https://github.com/briannadoubt/trebuchet-plugin/issues)
- **Claude Code**: Visit [claude.ai/code](https://claude.ai/code)
