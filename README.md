# Datalore skills for AI agents

This repository contains official AI skills for working with JetBrains Datalore.

## /datalore-notebook

Work within a notebook from your AI agent.

The skill is limited to working with a single existing notebook. It cannot create notebooks or use databases or data sources that are not attached yet.
These restrictions limit potential impact and reduce review effort.

### Supported versions

The minimum required Datalore On-Premises version is 2026.2.
The skill works in Datalore Cloud.

### Installation

The `datalore` script requires [uv](https://docs.astral.sh/uv/) to be installed on the machine.

#### Claude

##### Plugin

This skill can be installed through the plugin marketplace:

```
/plugin marketplace add JetBrains/datalore-skills
/plugin install datalore-skills@jetbrains-datalore
```

##### Manual

```bash
mkdir -p ~/.claude/skills
cp -R skills/datalore-notebook ~/.claude/skills/
```

#### Codex or other tool respecting `.agents`

```bash
mkdir -p ~/.agents/skills
cp -R skills/datalore-notebook ~/.agents/skills/
```

### Authentication

The CLI is authenticated with a PKCE flow. By default, the tokens are are per-notebook (scoped to public notebook API) and live for 7 days.