# symbiont-governed-agents

A Paperclip skill that teaches agents how to build and run governed Symbiont agents.

## Installation

Copy this directory into your Paperclip instance's `skills/` directory:

```bash
cp -r symbiont-governed-agents /path/to/paperclip/skills/
```

Or add it to a Paperclip company's skill configuration.

## Directory Structure

```
symbiont-governed-agents/
  SKILL.md                           # Paperclip-facing skill (decision-oriented)
  references/
    symbiont-skill.md                 # Full Symbiont SKILL.md (1,500 lines, DSL reference)
  README.md                          # This file
```

## How It Works

The `SKILL.md` is read by Paperclip agents at runtime. It tells them:

1. **When** to use Symbiont (sensitive data, audit trails, compliance)
2. **How** to scaffold a governed agent (DSL + Cedar policy)
3. **How** to wire it to Paperclip's heartbeat adapter

When the agent needs deep DSL syntax or policy pattern details, it loads
`references/symbiont-skill.md` which contains the complete Symbiont
agent development guide.

## Prerequisites

The Symbiont CLI (`symbi`) must be installed on the machine running
the Paperclip agent that will execute governed tasks.

## License

MIT
