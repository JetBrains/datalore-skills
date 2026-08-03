# Datalore skills for AI agents

This repository contains official AI skills for working with [JetBrains Datalore](https://datalore.jetbrains.com/).

## /datalore-notebook

Work within a notebook from your AI agent.

The skill is limited to working with a single existing notebook. It cannot create notebooks or use databases or data sources that are not attached yet.
These restrictions limit potential impact and reduce review effort.

### Supported versions

The minimum required Datalore On-Premises version is 2026.2.
The skill works in Datalore Cloud.

### Installation

The `datalore` script requires [uv](https://docs.astral.sh/uv/) to be installed on the machine.

#### Via [skills](https://github.com/vercel-labs/skills) to any supporting agent

Install the skill with `npx skills`:

```bash
npx skills add https://github.com/JetBrains/datalore-skills/tree/2026.2.2 --skill datalore-notebook --global
```

#### Claude Plugin

This skill can be installed through the plugin marketplace:

```
/plugin marketplace add https://github.com/JetBrains/datalore-skills.git#2026.2.2
/plugin install datalore-skills@jetbrains-datalore
```

### Usage

Ask an agent to do something, mentioning a notebook URL. For example, "In Datalore notebook https://datalore.jetbrains.com/notebook/qwerty/asdfgh analyze the attached csv".

You will be asked to confirm access to the notebook. After confirmation, the token is stored in the system keychain and the notebook data in the `.datalore-session` file in the current directory. Run `datalore logout` from the same directory to delete the token from the keychain.

The token is per-notebook (scoped to public notebook API) and lives for 7 days.
