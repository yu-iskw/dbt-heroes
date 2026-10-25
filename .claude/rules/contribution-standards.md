# Contribution Standards

## Architecture Decision Records (ADRs)

1. **Architectural Changes Require ADRs**: Any change that affects:
   - Plugin structure or organization
   - Shared resource architecture (skills, agents, rules)
   - Safety enforcement mechanisms
   - Cross-tool compatibility
   - Testing or validation strategies

   These changes MUST be documented in an ADR using `/manage-adr create`.

2. **ADR Linkage**: When creating a new ADR, link to related ADRs to maintain decision traceability.

3. **ADR Updates**: If a decision changes, create a new ADR marking the old one as "Superseded" rather than modifying the original.

## Code Quality Standards

1. **Pre-Commit Checks**: Before committing code:
   - Run `make format` to apply consistent formatting
   - Run `make lint` to catch issues early
   - Fix all linting errors (don't bypass linters)

2. **Commit Message Format**:
   - Use clear, imperative commit messages (e.g., "Add safety hook for delete operations")
   - Reference related ADRs or issues when relevant
   - Keep commits focused on single logical changes
   - Include "Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>" for AI-assisted commits

3. **No Commented-Out Code**: Remove dead code rather than commenting it out. Git history preserves old code.

## Testing Standards

1. **Integration Test Requirement**: Before opening a PR:
   - Run `make test-integration-docker` successfully
   - Document test evidence in PR description
   - For new features, add integration test coverage

2. **Eval-Driven Development**: For complex features (complexity score 4+):
   - Define 20-50 test cases BEFORE implementation
   - Test cases should represent real user scenarios
   - Include both success and failure scenarios
   - Document eval criteria in the skill/agent definition

## Pull Request Guidelines

1. **PR Scope**: Keep PRs focused and reviewable:
   - Single logical change per PR
   - Include context in PR description (what, why, how)
   - Link to related ADRs or issues
   - Document test evidence (commands run + outcomes)

2. **Documentation Updates**: When changing behavior:
   - Update relevant README.md files
   - Update plugin documentation in `docs/plugins/`
   - Update skill instructions if skill behavior changes
   - Update `AGENTS.md` if project-wide patterns change

## Plugin Development Standards

1. **Plugin Naming**: Use lowercase with hyphens (e.g., `lightdash-development`, not `lightdash_development` or `LightdashDevelopment`)

2. **Component Organization**:
   - Plugin-specific components go in `plugins/{name}/`
   - Shared components go in `.claude/skills/` or `.agents/agents/`
   - Don't duplicate shared logic across plugins

3. **Manifest Completeness**: Every plugin MUST have:
   - Valid `plugin.json` with name, description, and component references
   - README.md with installation and usage instructions
   - Appropriate safety hooks if the plugin has destructive tools

## Complexity Management

1. **Start Simple Philosophy**: Before implementing a complex solution:
   - Score complexity using the decision framework (0-10 scale)
   - Ask "Can this be simpler?"
   - Prefer simple skills (0-3) over sub-agents (7-8) when possible
   - Only use agent teams (9-10) when coordination is truly necessary

2. **Progressive Disclosure**: Structure instructions to provide:
   - Role/purpose first
   - Task requirements second
   - Detailed guidance last
   - Reference deep documentation rather than duplicating it
