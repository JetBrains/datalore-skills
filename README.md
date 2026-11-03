# Datalore skills for AI agents

This repository contains official AI skills for working with [JetBrains Datalore](https://datalore.jetbrains.com/).

## /datalore-notebook

Work with Datalore notebooks and workspaces from your AI agent.

In a notebook, the agent can:

- read, create, edit, run, and delete cells (code, SQL, Markdown, and controls) and inspect their outputs;
- explore attached databases (schemas, tables, search) and notebook files, upload and download files;
- manage the kernel: check its status, interrupt it, or stop the computation;
- attach and detach workspace resources: databases, data sources, environments, and Git repositories;
- build report layouts and publish static or interactive reports.

In a workspace, the agent can:

- list workspaces, browse folders and notebooks, create folders and notebooks, rename, move, and trash entries;
- manage workspace resources: databases, data sources, environments, Git repositories, and SSH keys;
- run SQL queries against workspace databases.

Access is granted per target: a single notebook, a workspace, or basic user information for listing workspaces.
Workspace access additionally has a level (read, create, or full), so the agent gets only the permissions the task needs.

### Supported versions

- **Datalore Cloud**: use the latest skill version from the `main` branch. This is what the default install commands below install. Versioned branches are not intended for Datalore Cloud and may not work with it.
- **Datalore On-Premises**: use the skill version matching the major version of your Datalore installation. Each major version has its own [branch](https://github.com/JetBrains/datalore-skills/branches) in this repository, for example, `2026.4` for Datalore 2026.4.x. Skill versions older than your Datalore installation may work, but this is not guaranteed. Skill versions newer than your Datalore installation are not supported. When you upgrade Datalore to a new major version, switch the skill to the matching branch.

The minimum supported Datalore On-Premises version is 2026.2.

### Installation

The `datalore` script requires [uv](https://docs.astral.sh/uv/) to be installed on the machine.

#### Via [skills](https://github.com/vercel-labs/skills) to any supporting agent

For Datalore Cloud, install the latest skill version from `main`:

```bash
npx skills add https://github.com/JetBrains/datalore-skills --skill datalore-notebook --global
```

For Datalore On-Premises, install from the branch matching your Datalore major version (see [Supported versions](#supported-versions)), for example, `2026.4`:

```bash
npx skills add https://github.com/JetBrains/datalore-skills/tree/2026.4 --skill datalore-notebook --global
```

#### Claude Code plugin

For Datalore Cloud, install the latest skill version from `main`:

```
/plugin install datalore-skills --marketplace JetBrains/datalore-skills
```

For Datalore On-Premises, install from the branch matching your Datalore major version (see [Supported versions](#supported-versions)), for example, `2026.4`:

```
/plugin install datalore-skills --marketplace JetBrains/datalore-skills#2026.4
```

On Claude Code versions earlier than 2.1.275, add the marketplace first, then install the plugin (for Datalore Cloud, omit `#2026.4`):

```
/plugin marketplace add JetBrains/datalore-skills#2026.4
/plugin install datalore-skills@jetbrains-datalore
```

Claude Code prefixes plugin skills with the plugin name, so the skill's command is `/datalore-skills:datalore-notebook`. You usually don't need to type it: the agent picks up the skill when you mention a Datalore notebook or workspace URL (see [Usage](#usage)).

To receive updates automatically, enable auto-update for the `jetbrains-datalore` marketplace in `/plugin` → **Marketplaces**. Otherwise, update manually with `claude plugin update datalore-skills@jetbrains-datalore`.

When you upgrade Datalore On-Premises to 2026.4 from an earlier major version, remove the marketplace pinned to the previous branch and install the skill from the `2026.4` branch:

```
/plugin marketplace remove jetbrains-datalore
/plugin install datalore-skills --marketplace JetBrains/datalore-skills#2026.4
```

### Usage

Ask an agent to do something, mentioning a notebook or workspace URL. For example:

- "In Datalore notebook https://datalore.jetbrains.com/notebook/qwerty/asdfgh analyze the attached csv"
- "In Datalore workspace https://datalore.jetbrains.com/qwerty/asdfgh/notebooks create a notebook that queries the orders database"
- "List my workspaces on https://datalore.jetbrains.com"

You will be asked in the browser to confirm access to the notebook or workspace. After confirmation, the token is stored in the system keychain and the session data in the `.datalore-session` file in the current directory. To delete the token from the keychain, ask the agent to log out of Datalore from the same directory.

Instead of browser authorization, you can provide a token in the `DATALORE_API_TOKEN` environment variable.

The token is scoped to the requested notebook or workspace and is issued for a limited time.

