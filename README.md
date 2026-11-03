# Datalore skills for AI agents

This repository contains official AI skills for working with [JetBrains Datalore](https://datalore.jetbrains.com/).

## /datalore-notebook

Work within a notebook from your AI agent.

The skill is limited to working with a single existing notebook. It cannot create notebooks or use databases or data sources that are not attached yet.
These restrictions limit potential impact and reduce review effort.

### Supported versions

- **Datalore Cloud**: use the latest skill version from the `main` branch. This is what the default install commands below install. Versioned branches are not intended for Datalore Cloud and may not work with it.
- **Datalore On-Premises**: use the skill version matching the major version of your Datalore installation. Each major version has its own [branch](https://github.com/JetBrains/datalore-skills/branches) in this repository, for example, `2026.2` for Datalore 2026.2.x. Skill versions older than your Datalore installation may work, but this is not guaranteed. Skill versions newer than your Datalore installation are not supported. When you upgrade Datalore to a new major version, switch the skill to the matching branch.

The minimum supported Datalore On-Premises version is 2026.2.

### Installation

The `datalore` script requires [uv](https://docs.astral.sh/uv/) to be installed on the machine.

#### Via [skills](https://github.com/vercel-labs/skills) to any supporting agent

For Datalore Cloud, install the latest skill version from `main`:

```bash
npx skills add https://github.com/JetBrains/datalore-skills --skill datalore-notebook --global
```

For Datalore On-Premises, install from the branch matching your Datalore major version (see [Supported versions](#supported-versions)), for example, `2026.2`:

```bash
npx skills add https://github.com/JetBrains/datalore-skills/tree/2026.2 --skill datalore-notebook --global
```

#### Claude Code plugin

For Datalore Cloud, install the latest skill version from `main`:

```
/plugin install datalore-skills --marketplace JetBrains/datalore-skills
```

For Datalore On-Premises, install from the branch matching your Datalore major version (see [Supported versions](#supported-versions)), for example, `2026.2`:

```
/plugin install datalore-skills --marketplace JetBrains/datalore-skills#2026.2
```

On Claude Code versions earlier than 2.1.275, add the marketplace first, then install the plugin (for Datalore Cloud, omit `#2026.2`):

```
/plugin marketplace add JetBrains/datalore-skills#2026.2
/plugin install datalore-skills@jetbrains-datalore
```

Claude Code prefixes plugin skills with the plugin name, so the skill's command is `/datalore-skills:datalore-notebook`. You usually don't need to type it: the agent picks up the skill when you mention a Datalore notebook URL (see [Usage](#usage)).

To receive updates automatically, enable auto-update for the `jetbrains-datalore` marketplace in `/plugin` → **Marketplaces**. Otherwise, update manually with `claude plugin update datalore-skills@jetbrains-datalore`.

When you upgrade Datalore On-Premises to 2026.2 from an earlier major version, remove the marketplace pinned to the previous branch and install the skill from the `2026.2` branch:

```
/plugin marketplace remove jetbrains-datalore
/plugin install datalore-skills --marketplace JetBrains/datalore-skills#2026.2
```

### Usage

Ask an agent to do something, mentioning a notebook URL. For example:

- "In Datalore notebook https://datalore.jetbrains.com/notebook/qwerty/asdfgh analyze the attached csv"

You will be asked in the browser to confirm access to the notebook. After confirmation, the token is stored in the system keychain and the session data in the `.datalore-session` file in the current directory. To delete the token from the keychain, ask the agent to log out of Datalore from the same directory.

Instead of browser authorization, you can provide a token in the `DATALORE_API_TOKEN` environment variable.

The token is scoped to the requested notebook.
