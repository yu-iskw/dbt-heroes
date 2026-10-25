# CLAUDE.md

@AGENTS.md

<!-- AGENTS.md is canonical for every coding agent. Put only Claude Code–specific notes below. -->

## Claude Code

- **Auto-loaded rules:** `.claude/rules/*.md` load at session start (repository-wide; plugin rules load when the plugin is active).
- **Slash-command skills (shared):** `/implement-plugin`, `/plugin-verification`, `/implement-agent-skills`, `/implement-sub-agents`, `/manage-adr`, `/implement-claude-extensions`, `/lint-fix`
- **Complexity and extension choice:** see [.claude/ANTHROPIC_BEST_PRACTICES.md](.claude/ANTHROPIC_BEST_PRACTICES.md)
- **Confirm loaded context:** `/context` or `/memory` after changing `CLAUDE.md` or `AGENTS.md`
