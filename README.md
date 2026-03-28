# dbt Heros

Accelerate your dbt lifecycle with specialized Claude Code plugins. This repository provides high-performance agents and skills for mastering dbt Core, dbt Cloud, dbt Fusion, Elementary, and Lightdash.

## Available Plugins

<!-- START_PLUGIN_TABLE -->

| Category           | Plugin                                                       | Description                                                    | Status  |
| :----------------- | :----------------------------------------------------------- | :------------------------------------------------------------- | :------ |
| **Semantic Layer** | [lightdash-development](./plugins/lightdash-development)     | dbt 1.10+ metrics, dimensions, and semantic layer modeling.    | ✅ GA   |
| **Analysis**       | [lightdash-analysis](./plugins/lightdash-analysis)           | Data discovery and insight generation from Lightdash explores. | ✅ GA   |
| **Content Ops**    | [lightdash-content-admin](./plugins/lightdash-content-admin) | Space organization and content validation for Lightdash.       | ✅ GA   |
| **Admin**          | [lightdash-org-admin](./plugins/lightdash-org-admin)         | Lightdash organization management (users, groups, projects).   | ✅ GA   |
| **AgentOps**       | [lightdash-agentops](./plugins/lightdash-agentops)           | Build, manage, and autonomously tune Lightdash AI agents.      | 🧪 Beta |

<!-- END_PLUGIN_TABLE -->

## Core Principles

- **Agentic Workflows**: Move beyond static documentation into interactive, goal-oriented collaboration.
- **Unified Interface**: A single repository for all dbt ecosystem extensions.
- **Safety First**: Deterministic safety modes (`read-only`, `write-idempotent`, `write-destructive`) for all tools.

## Quickstart

### 1. Prerequisites

Ensure you have [Claude Code](https://code.claude.com) installed and authenticated.

### 2. Configuration

Refer to the [Product Documentation](#product-documentation) for detailed configuration instructions for each plugin.

## Product Documentation

<!-- START_DOCS_LIST -->

- [Lightdash Plugins Documentation](./docs/plugins/lightdash_plugins.md)

<!-- END_DOCS_LIST -->

### 3. Installation

The most efficient way to use these plugins is by adding this repository as a plugin marketplace. This enables discovery and management of all specialized plugins through the `/plugin` interface.

#### Claude Code

##### Option A: Remote Marketplace (Recommended)

Add the repository as a marketplace to enable easy discovery and updates:

```bash
# Within Claude Code
/plugin marketplace add yu-iskw/dbt-heros
```

Once added, you can browse available plugins via `/plugin discover` or install them directly:

```bash
# Within Claude Code
/plugin install lightdash-analysis@dbt-heros
```

##### Option B: Direct Plugin Installation

You can also install specific plugins directly using their GitHub path:

```bash
# Within Claude Code
/plugin install yu-iskw/dbt-heros/plugins/lightdash-analysis
```

##### Option C: Local Installation (Development)

If you have cloned the repository locally, you can add it as a local marketplace or install individual plugins from local paths:

```bash
# Add as a local marketplace (at the repository root)
/plugin marketplace add .

# Or install an individual plugin
claude plugin install ./plugins/lightdash-analysis
```

#### Codex

Each plugin ships a `.codex-plugin/plugin.json` manifest. The repo-scoped marketplace catalog is at `.agents/plugins/marketplace.json`.

##### Option A: Repo-scoped Marketplace (Recommended)

When working inside this repository, Codex auto-detects `.agents/plugins/marketplace.json` and surfaces all five plugins in the plugin directory. No extra setup is needed. To enable a specific plugin:

```bash
# Within Codex
/plugin install lightdash-analysis@dbt-heroes
```

Codex caches installed plugins under `~/.codex/plugins/cache/dbt-heroes/<plugin-name>/local/`. Each plugin can be enabled or disabled individually; state is stored in `~/.codex/config.toml`.

##### Option B: Personal Marketplace (cross-repo access)

To make the plugins available from any repository, copy or symlink the marketplace file to the personal catalog location:

```bash
mkdir -p ~/.agents/plugins

# Copy
cp .agents/plugins/marketplace.json ~/.agents/plugins/marketplace.json

# Or symlink (stays in sync with repo updates)
ln -s "$(pwd)/.agents/plugins/marketplace.json" ~/.agents/plugins/marketplace.json
```

After restarting Codex, the dbt-heroes plugins will appear in the plugin directory regardless of the working directory.

## Development

This repository is a monorepo. To add a new plugin or contribute to existing ones, refer to our [Development Guide](./docs/development.md) (coming soon).

- **Standard Layout**: Every plugin follows the `.claude-plugin/plugin.json` manifest (Claude Code) and `.codex-plugin/plugin.json` manifest (Codex) standards.
- **CI/CD**: Unified linting and integration testing via `make lint` and `make test-integration-docker`.
- **Codex**: Project instructions are in `AGENTS.md`; shared skills are in `.agents/skills` (symlink to `.claude/skills`).

---

License: Apache 2.0
