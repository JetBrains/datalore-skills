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

Skill versions are published as [tags](https://github.com/JetBrains/datalore-skills/tags) of this repository.

- **Datalore Cloud**: we recommend using the latest version of the skill. Older skill versions may not work with Datalore Cloud.
- **Datalore On-Premises**: we recommend using the latest skill version with the same major version as your Datalore installation. For example, for Datalore 2026.3.x, use the latest 2026.3 skill version. Older skill versions may work with newer Datalore versions, but this is not guaranteed. Skill versions newer than your Datalore installation are not supported. When you upgrade Datalore to a new major version, update the skill as well.

The minimum supported Datalore On-Premises version is 2026.2.

### Installation

The `datalore` script requires [uv](https://docs.astral.sh/uv/) to be installed on the machine.

#### Via [skills](https://github.com/vercel-labs/skills) to any supporting agent

For Datalore Cloud, install the latest skill version with `npx skills`:

```bash
npx skills add https://github.com/JetBrains/datalore-skills/tree/2026.2.2 --skill datalore-notebook --global
```

For Datalore On-Premises, install the skill version matching your Datalore major version (see [Supported versions](#supported-versions)), for example, `2026.3`:

```bash
npx skills add https://github.com/JetBrains/datalore-skills/tree/2026.3 --skill datalore-notebook --global
```

#### Claude Plugin

This skill can be installed through the plugin marketplace.

For Datalore Cloud, add the marketplace with the latest skill version:

```
/plugin marketplace add https://github.com/JetBrains/datalore-skills.git#2026.2.2
/plugin install datalore-skills@jetbrains-datalore
```

For Datalore On-Premises, add the marketplace pinned to the skill version matching your Datalore major version (see [Supported versions](#supported-versions)), for example, `2026.3`:

```
/plugin marketplace add https://github.com/JetBrains/datalore-skills.git#2026.3
/plugin install datalore-skills@jetbrains-datalore
```

### Usage

Ask an agent to do something, mentioning a notebook or workspace URL. For example:

- "In Datalore notebook https://datalore.jetbrains.com/notebook/qwerty/asdfgh analyze the attached csv"
- "In Datalore workspace https://datalore.jetbrains.com/qwerty/asdfgh/notebooks create a notebook that queries the orders database"
- "List my workspaces on https://datalore.jetbrains.com"

You will be asked in the browser to confirm access to the notebook or workspace. After confirmation, the token is stored in the system keychain and the session data in the `.datalore-session` file in the current directory. Run `datalore logout` from the same directory to delete the token from the keychain.

Instead of browser authorization, you can provide a token in the `DATALORE_API_TOKEN` environment variable.

The token is scoped to the requested notebook or workspace and is issued for a limited time.

