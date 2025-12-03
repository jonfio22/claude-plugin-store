# Claude Plugin Store

A collection of Claude Code plugins for development workflows, agentic orchestration, and AI agent development.

## Plugins

### daw-workflow
Production-grade DAW development toolkit with:
- DSP mathematics (biquad filters, compressor curves, saturation)
- Elementary Audio patterns
- ITU-R BS.1770 LUFS metering
- Zustand state management
- Real-time audio programming constraints

### agentic-orchestration
Systematic orchestration framework for managing complex multi-task projects through agentic execution.

### google-adk
Google Agent Development Kit (ADK) toolkit for:
- Building AI agents
- Prompt engineering
- Tool design
- Multi-agent patterns
- Deployment workflows

## Installation

Add this marketplace to Claude Code:

```bash
claude plugins:add-marketplace github:jonfio22/claude-plugin-store
```

Then install individual plugins:

```bash
claude plugins:install daw-workflow@jonfio-plugins
claude plugins:install agentic-orchestration@jonfio-plugins
claude plugins:install google-adk@jonfio-plugins
```

## License

MIT
