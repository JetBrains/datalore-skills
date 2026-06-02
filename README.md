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

To work within a notebook, authenticate for it first.

We recommend using per-notebook tokens with short expiration dates.

1. Open the notebook (let's say, it's `https://datalore.jetbrains.com/notebook/abcd/efgh`)
2. Open a directory in your shell. It will contain the `.datalore-session` file.
3. From that directory, run, depending on your installation path:
   ```bash
   ~/.claude/skills/datalore-notebook/scripts/datalore init https://datalore.jetbrains.com/notebook/abcd/efgh
   # or
   ~/.agents/skills/datalore-notebook/scripts/datalore init https://datalore.jetbrains.com/notebook/abcd/efgh
   ```
4. In the Datalore notebook UI, open "Tools > API Tokens", create a token, copy it, and paste it into the terminal.
5. Open your AI agent interface, and prompt your request: for example, "In Datalore notebook https://datalore.jetbrains.com/notebook/abcd/efgh, analyze the attached csv file."
