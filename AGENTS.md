# dbt Heros

## Repository Agent Operating Guide

This file is the canonical repository-wide operational contract for Codex, Cursor, Claude Code (imported from `CLAUDE.md` via `@AGENTS.md`), and related coding agents.

## Project Identity

This is a **multi-plugin monorepo** providing specialized Claude Code plugins for the dbt ecosystem, with a primary focus on Lightdash integration.

**Plugins:**

- `lightdash-development`: dbt 1.10+ semantic layer modeling (metrics, dimensions)
- `lightdash-analysis`: Data discovery and insight generation from Lightdash explores
- `lightdash-content-admin`: Space organization and content validation
- `lightdash-org-admin`: Organization management (users, groups, projects)
- `lightdash-agentops`: Build, manage, and autonomously tune Lightdash AI agents (beta)

**Multi-Tool Support:** Claude Code, Codex, and Cursor use a symlinked shared resource layout (see [ADR-0013](docs/adr/0013-adopt-agents-root-for-shared-agent-resources.md)).

## When documentation and code disagree

When a repository's documentation and its existing code disagree, follow the conventions used in the existing code, and in your final reply point out each inconsistency you found between the documentation and the code.

"Documentation" includes this file, `CLAUDE.md`, editor rules, READMEs, `docs/`, ADRs, RFCs, and code comments. Treat what they say about the code as claims to check against the code, not as facts.

- **Documented target state wins only when it is explicit.** Follow the documentation instead of the code only when the documentation says the code is legacy or a migration is in progress. For example: a "migration in progress", "legacy", or "deprecated" marker, `@deprecated` on the code you would copy, or an ADR/RFC with status _Accepted_ that is not superseded and is newer than the code it replaces. Ignore ADRs/RFCs that are _Proposed_, _Rejected_, _Deprecated_, or _Superseded_. Follow the superseding one.
- **When you can't tell which is current,** compare recency (`git log -1 --format=%cs -- <doc>` vs the code paths). Prefer the newer source, and say that you did. If the choice changes behavior and is still unclear, ask before proceeding.
- **Scope.** Apply the convention you choose to the code you are writing or changing. Don't rewrite working code, and don't edit documentation just to make the two agree, unless the task asks for it.
- **Process and safety rules still apply.** Commands to run, quality gates, review/merge rules, and security constraints in this file are not descriptions of the code. Follow them.
- **Report.** In your final reply (and in the PR description, if you open one), list each inconsistency: the document and section, what the code actually does, which one you followed, and why.

## Architectural Philosophy

### 1. Safety First (ADR-0005)

All destructive operations are **deterministically enforced** through PreToolUse hooks, not just LLM prompts. Safety modes:

- `read-only`: No mutations allowed
- `write-idempotent`: Non-destructive writes only
- `write-destructive`: Requires explicit opt-in

**Rule:** When implementing or using tools, verify the safety mode enforcement is in place before allowing destructive operations.

### 2. Start Simple (`.claude/ANTHROPIC_BEST_PRACTICES.md`)

Use the **complexity scoring framework** (0-10 scale) before implementing:

- **0-3**: Simple Skill (deterministic, single task)
- **4-6**: Workflow Skill (multi-step, some branching)
- **7-8**: Sub-Agent (autonomous, model-driven decisions)
- **9-10**: Agent Team (multi-agent coordination)

**Rule:** Always ask "Can this be simpler?" before choosing a more complex extension type.

### 3. Shared Resources Architecture (ADR-0013)

Canonical shared skills live in `.claude/skills/`. Codex loads them via `.agents/skills` (symlink to `.claude/skills`). Shared sub-agent definitions for this repo live under `.claude/agents/`.

**Rule:** When creating shared skills or sub-agents, place skills in `.claude/skills/` (not in plugin directories). Do not duplicate content across paths; edit the canonical skill tree only.

### 4. ADR-Driven Architecture

All architectural decisions are documented in `docs/adr/`. Before making significant architectural changes:

1. Check existing ADRs for context
2. Use the `/manage-adr` skill to create new ADRs
3. Link to related ADRs

**Rule:** Architectural changes without ADRs are not accepted.

### 5. Eval-Driven Development

Before implementing agents or complex skills, create **20-50 test cases** representing real scenarios. Build evals BEFORE satisfying them.

**Rule:** Complex features (complexity score 4+) require documented test cases before implementation.

## Quality gates (format, lint, test)

All commands are defined in the root `Makefile` (`trunk fmt` / `trunk check` / Docker integration smoke).

- **Before commit:** `make format && make lint`
- **Before PR:** `make test-integration-docker` and document test evidence in the PR description
- **Manifest check (plugins):** `./integration_tests/validate-manifest.sh`

## Navigation

- **Architecture decisions:** [docs/adr/](docs/adr/) (16 ADR files; ADR-0003 is superseded by ADR-0004)
- **Best practices (Claude):** [.claude/ANTHROPIC_BEST_PRACTICES.md](.claude/ANTHROPIC_BEST_PRACTICES.md)
- **Plugin documentation:** [docs/plugins/lightdash_plugins.md](docs/plugins/lightdash_plugins.md)
- **Contributor workflow:** [CONTRIBUTING.md](CONTRIBUTING.md)
- **Key shared skills** (under `.claude/skills/`, also `.agents/skills`):
  - `implement-plugin` — package and distribute plugins with schema-safe manifests and validation
  - `plugin-verification` — verify extensions, manifests, structure, and runtime loading
  - `implement-agent-skills` — implement skills with valid frontmatter and progressive disclosure
  - `implement-sub-agents` — create sub-agents with correct permissions and boundaries
  - `manage-adr` — initialize, create, list, and link architecture decision records

  Read a skill's `SKILL.md` when the task matches its description.

## Project-Specific Patterns

### Plugin Structure

Each plugin follows this canonical layout:

```text
plugins/{plugin-name}/
├── .claude-plugin/
│   └── plugin.json         # Manifest (name, description, components)
├── agents/                 # Plugin-specific sub-agents
├── hooks/                  # PreToolUse, PostToolUse hooks
├── rules/                  # Plugin-specific rules (auto-loaded when plugin active)
├── skills/                 # Plugin-specific skills
└── README.md               # Plugin documentation
```

### Safety Mode Integration

When implementing tools that mutate state:

1. Define the tool in the plugin's MCP server or tool definition
2. Add a PreToolUse hook in `plugins/{plugin-name}/hooks/hooks.json`
3. Wire the hook to `plugins/{plugin-name}/hooks/safety.sh` (see existing plugins such as `lightdash-analysis`, `lightdash-development`, `lightdash-org-admin`, and `lightdash-agentops`)
4. Document the safety mode requirement in the tool description

### Monorepo Coordination

- **Shared skills** are in `.claude/skills/` and loaded by all plugins via symlinks
- **Plugin-specific skills** are in `plugins/{name}/skills/` and only load when that plugin is active
- **Rules** in `.claude/rules/` apply repository-wide; rules in `plugins/{name}/rules/` are plugin-specific

## When Working on This Repository

1. **Before creating new components:** Use `/implement-claude-extensions` (Claude Code) to determine whether you need a skill, sub-agent, hook, or team
2. **Before implementing:** Score complexity (0-10) and choose the simplest approach
3. **Before commit / PR:** Follow [Quality gates (format, lint, test)](#quality-gates-format-lint-test)
4. **For architectural changes:** Create an ADR using `/manage-adr`

## Common Workflows

### Adding a New Plugin

1. Use `/implement-plugin` skill
2. Follow the canonical plugin structure above
3. Add safety enforcement hooks if tools are destructive
4. Update the plugin table in `README.md`
5. Create plugin documentation in `docs/plugins/`
6. Create an ADR documenting the plugin decision

### Enhancing Existing Plugins

1. Read the plugin's `README.md` and `rules/` to understand constraints
2. Check related ADRs for architectural context
3. Verify your changes maintain safety mode enforcement
4. Update plugin documentation if behavior changes
5. Run plugin-specific integration tests as needed (see quality gates)

### Contributing Shared Skills

1. Place skill in `.claude/skills/{skill-name}/SKILL.md`
2. Ensure skill has proper YAML frontmatter
3. Test that skill is accessible via symlinks (`.agents/skills/`)
4. Mirror structure and frontmatter patterns from sibling skills in `.claude/skills/`
