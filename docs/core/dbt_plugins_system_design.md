# dbt Engineering Multi-Plugin System Design

## 1. Executive Summary

### 1.1 Goal

The goal of this system is to provide a comprehensive, AI-native development experience for dbt Core 1.10+ and dbt Fusion users. By leveraging the Claude Code plugin architecture, we automate the entire dbt development lifecycle—from initial resource scaffolding to semantic layer modeling and rigorous quality enforcement—without requiring dbt Cloud.

### 1.2 Scope

This design covers the implementation of three specialized plugins:

1. **dbt-resource-factory**: The daily authoring assistant for scaffolding SQL models, YAML, tests, and documentation.
2. **dbt-semantic-architect**: The specialist for MetricFlow-based semantic layers, metrics, and saved queries.
3. **dbt-quality-guardian**: The automated reviewer for linting, DAG auditing, and governance enforcement.

### 1.3 High-Level Architecture

The three plugins operate as a coordinated trio, handing off artifacts through the local filesystem and the dbt manifest.

- **Flow**: `dbt-resource-factory` creates the core resources (SQL/YAML) → `dbt-quality-guardian` validates them against standards → `dbt-semantic-architect` builds the metrics layer on top of validated models.
- **Infrastructure**: All plugins utilize the **dbt MCP Server** for project introspection and are optimized for **dbt Fusion** (local execution).

---

## 2. Per-Plugin System Design

### 2.1 dbt-resource-factory (Candidate A)

#### Design Philosophy

The "Factory Pattern" for dbt. It focuses on high-velocity authoring by translating developer intent into production-ready dbt resources. It follows opinionated layering (staging, intermediate, marts) and ensures all resources are created with their corresponding YAML metadata and tests.

#### Agent Hierarchy (L0→L3)

- **L0**: Central Orchestrator
- **L1**: `dbt-resource-factory-orchestrator`
  - **L2: sql-modeling-team**
    - **L3: model-architect**: Designs model structure and DAG placement.
    - **L3: sql-generator**: Writes model SQL using Jinja and dbt macros.
  - **L2: schema-team**
    - **L3: yaml-scaffolder**: Generates and updates schema.yml files.
    - **L3: test-suggester**: Recommends generic and singular tests based on semantics.
    - **L3: unit-test-author**: Writes dbt 1.8+ `unit_tests:` blocks.
  - **L2: resource-team**
    - **L3: source-definer**: Scaffolds `_sources.yml` from warehouse metadata.
    - **L3: snapshot-builder**: Creates SCD Type-2 snapshot files.
    - **L3: seed-manager**: Manages CSV seeds and their YAML configs.
    - **L3: udf-author**: Authors dbt 1.11+ SQL/Python functions.

#### Skills

| Skill Name              | Input                | Output                | When to Use                       |
| :---------------------- | :------------------- | :-------------------- | :-------------------------------- |
| `scaffold-model-sql`    | Intent, Table Schema | `.sql` file           | Creating new models               |
| `generate-schema-yaml`  | Model Name           | `.yml` entry          | Adding metadata/tests to models   |
| `suggest-generic-tests` | Column Metadata      | Test List             | Improving data quality            |
| `write-unit-tests`      | Logic Description    | `unit_tests:` YAML    | Validating complex SQL logic      |
| `scaffold-source-yaml`  | Source Table         | `_sources.yml`        | Onboarding new raw data           |
| `scaffold-snapshot`     | Source, Unique Key   | `.sql` snapshot       | Tracking SCD Type-2 history       |
| `generate-doc-blocks`   | Model/Column         | `{% docs %}` / `.yml` | Documenting the project           |
| `scaffold-udf`          | Function Logic       | `functions/*.sql`     | Creating reusable warehouse logic |

#### MCP Integration

- **Tools**: `get_model_lineage_dev`, `get_macro_details`, `get_seed_details`.
- **Usage**: Introspects existing macros to avoid duplication and uses lineage to suggest correct `ref()` calls.

#### Hooks

- **PreToolUse[Write]**: Validate generated YAML against dbt JSONSchema.
- **PreToolUse[Bash]**: Guard against `dbt run --full-refresh` in production targets.
- **PostToolUse[Write]**: Audit log all resource file writes.

#### Directory Structure

```text
plugins/dbt-resource-factory/
├── .mcp.json
├── README.md
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   ├── dbt-resource-factory-orchestrator.md
│   ├── sql-modeling-team.md
│   ├── schema-team.md
│   └── resource-team.md
├── hooks/
│   └── hooks.json
├── rules/
│   ├── layering-conventions.md
│   └── safety.md
└── skills/
    ├── scaffold-model-sql/
    ├── generate-schema-yaml/
    ├── write-unit-tests/
    └── ... (other skills)
```

#### Dependencies

- dbt Core 1.10+
- dbt Fusion
- dbt-mcp
- dbt-codegen (reference logic)

---

### 2.2 dbt-semantic-architect (Candidate E)

#### Design Philosophy

A specialist for the dbt Semantic Layer (MetricFlow). It bridges the gap between raw data models and business KPIs. It focuses on the complex YAML syntax required for entities, measures, dimensions, and metrics, ensuring "MetricFlow-ready" configurations.

#### Agent Hierarchy (L0→L3)

- **L0**: Central Orchestrator
- **L1**: `dbt-semantic-architect-orchestrator`
  - **L2: semantic-model-team**
    - **L3: entity-mapper**: Defines primary and foreign entities.
    - **L3: dimension-builder**: Configures time and categorical dimensions.
  - **L2: metrics-team**
    - **L3: metric-author**: Authors simple, ratio, and derived metrics.
    - **L3: saved-query-builder**: Creates reusable saved queries and exports.
    - **L3: metric-validator**: Validates syntax against MetricFlow spec.
  - **L2: governance-team**
    - **L3: lineage-tracer**: Traces dependencies from model to metric via MCP.
    - **L3: mcp-integrator**: Configures the local dbt MCP server.

#### Skills

| Skill Name                 | Input                 | Output                  | When to Use                     |
| :------------------------- | :-------------------- | :---------------------- | :------------------------------ |
| `generate-semantic-model`  | Model SQL             | `semantic_models:` YAML | Initial semantic definition     |
| `define-entities`          | Column Names          | Entity YAML             | Mapping joins and PKs           |
| `scaffold-time-dimensions` | Date Columns          | Dimension YAML          | Setting up time-series analysis |
| `author-simple-metric`     | Measure               | Metric YAML             | Defining basic KPIs             |
| `author-derived-metric`    | Base Metrics          | Metric YAML             | Defining complex/ratio KPIs     |
| `scaffold-saved-query`     | Metric/Dimension List | `saved_queries:` YAML   | Creating BI-ready views         |
| `validate-metricflow`      | Project State         | Validation Output       | Running `dbt sl validate`       |
| `embed-semantic-yaml`      | Legacy YAML           | Fusion-embedded YAML    | Migrating to dbt 1.10+ format   |

#### MCP Integration

- **Tools**: `get_semantic_layer_query`, `list_metrics`, `get_semantic_model_details`.
- **Usage**: Heavily relies on MCP to validate that metrics resolve against the underlying physical models.

#### Hooks

- **`PreToolUse[Write][*semantic*|*metric*|*saved_query*]`**: Validate MetricFlow YAML syntax before file write.
- **`PostToolUse[Bash][dbt sl validate]`**: Intercept `dbt sl validate` failures and route to `metric-validator` for auto-fixing.

#### Slash Commands

- `/dbt-semantic-architect:generate-semantic-model`: Model SQL → semantic_model YAML.
- `/dbt-semantic-architect:author-metrics`: Define metrics from business requirements.
- `/dbt-semantic-architect:validate`: Run MetricFlow validation.
- `/dbt-semantic-architect:embed-yaml`: Migrate to Fusion embedded YAML format.
- `/dbt-semantic-architect:setup-mcp`: Configure local dbt MCP server.
- `/dbt-semantic-architect:trace-lineage`: Trace model-to-metric lineage via MCP.

#### Directory Structure

```text
plugins/dbt-semantic-architect/
├── .mcp.json
├── README.md
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   ├── dbt-semantic-architect-orchestrator.md
│   ├── semantic-model-team.md
│   ├── metrics-team.md
│   └── governance-team.md
├── hooks/
│   └── hooks.json
├── rules/
│   ├── metricflow-standards.md
│   └── safety.md
└── skills/
    ├── generate-semantic-model/
    ├── author-simple-metric/
    ├── validate-metricflow/
    └── ... (other skills)
```

#### Dependencies

- dbt Core 1.10+
- dbt Fusion
- dbt-mcp (with Semantic Layer tools)
- MetricFlow

---

### 2.3 dbt-quality-guardian (Candidate B)

#### Design Philosophy

The "Quality Gatekeeper." It enforces engineering standards across the project. It integrates external tools like SQLFluff and dbt-project-evaluator to ensure the codebase remains clean, efficient, and ready for the dbt Fusion engine.

#### Agent Hierarchy (L0→L3)

- **L0**: Central Orchestrator
- **L1**: `dbt-quality-guardian-orchestrator`
  - **L2: lint-team**
    - **L3: sql-linter**: Runs SQLFluff for style, formatting, and complexity.
    - **L3: yaml-validator**: Validates YAML against JSONSchema and deprecation rules.
  - **L2: dag-audit-team**
    - **L3: structure-auditor**: Uses `dbt-project-evaluator` to find DAG smells.
    - **L3: coverage-inspector**: Analyzes test coverage and freshness gaps.
  - **L2: governance-team**
    - **L3: contract-enforcer**: Validates model contracts and access levels.
    - **L3: deprecation-scanner**: Scans for configs incompatible with Fusion.
    - **L3: naming-police**: Enforces stg/int/fct naming conventions.

#### Skills

| Skill Name             | Input          | Output             | When to Use                     |
| :--------------------- | :------------- | :----------------- | :------------------------------ |
| `run-sqlfluff-lint`    | SQL Files      | Lint Report        | Checking SQL style              |
| `audit-dag-structure`  | Project State  | Audit Report       | Identifying DAG violations      |
| `validate-yaml-schema` | YAML Files     | Validation Results | Ensuring Fusion readiness       |
| `check-test-coverage`  | Manifest       | Coverage Report    | Finding untested models         |
| `scan-deprecations`    | Project Config | Deprecation List   | Preparing for upgrades          |
| `enforce-naming-rules` | File Paths     | Violation Report   | Enforcing directory/file naming |
| `validate-contracts`   | Model YAML     | Contract Status    | Enforcing schema stability      |

#### MCP Integration

- **Tools**: `get_model_lineage_dev`.
- **Usage**: Determines the impact of contract violations on downstream consumers.

#### Hooks

- **`PreToolUse[Write][*.sql]`**: Auto-lint SQL files before save (SQLFluff).
- **`PreToolUse[Write][*.yml]`**: JSONSchema validation for all YAML files before writing.
- **`PostToolUse[Bash][dbt build]`**: Parse exit code + test results, recommend missing coverage.

#### Slash Commands

- `/dbt-quality-guardian:lint`: SQLFluff lint changed models.
- `/dbt-quality-guardian:audit`: Full `dbt-project-evaluator` run.
- `/dbt-quality-guardian:coverage`: Test coverage and freshness report.
- `/dbt-quality-guardian:check-fusion`: Fusion readiness scan.
- `/dbt-quality-guardian:enforce-naming`: Audit project naming conventions.

#### Directory Structure

```text
plugins/dbt-quality-guardian/
├── README.md
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   ├── dbt-quality-guardian-orchestrator.md
│   ├── lint-team.md
│   ├── dag-audit-team.md
│   └── governance-team.md
├── hooks/
│   └── hooks.json
├── rules/
│   ├── sqlfluff-config.md
│   ├── dag-best-practices.md
│   └── safety.md
└── skills/
    ├── run-sqlfluff-lint/
    ├── audit-dag-structure/
    ├── validate-yaml-schema/
    └── ... (other skills)
```

#### Dependencies

- dbt Core 1.10+
- dbt Fusion
- SQLFluff
- dbt-project-evaluator (package)
- dbt-autofix (for Fusion migration)

---

## 3. Cross-Cutting Decisions

### 3.1 Mode Safety

- **Read-Only vs Write-Safestructive**: Follow existing patterns with explicit mode declarations in `plugin.json`.
- **DISABLE_PLATFORM_FEATURES**: Always set to `true` for dbt MCP to ensure local-only (Fusion/Core) operation.

### 3.2 Target Guardrails

All plugins must respect the `target` configuration in `profiles.yml`.

- **Rules**: Hooks must block production target access except in explicit deploy commands. Any command modifying data or running SQL must block `--target prod` by default.

### 3.3 Fusion Readiness

- Plugins will default to **dbt 1.10+ syntax** (e.g., `config.meta` over `meta`, `unit_tests` over custom macros).
- **dbt-autofix integration**: `dbt-quality-guardian` offers a fix mode that runs `dbt-autofix` for Fusion migration.
- **Versioning**: Generated `dbt_project.yml` or configs must include `require-dbt-version: ">=1.10.0"`.

---

## 4. Implementation Roadmap

### Phase 1: Scaffolding (Weeks 1-2)

- Initialize plugin directories: `plugins/dbt-{resource-factory,semantic-architect,quality-guardian}`.
- Implement L0/L1 orchestrators and basic L2 team structures.
- Setup shared references in `shared/`.
- **Acceptance Criteria**: All three plugin skeletons exist and respond to basic discovery commands.

### Phase 2: Core Authoring & Skills (Weeks 3-5)

- Build `dbt-resource-factory` L3 agents.
- Implement SQL generation and YAML scaffolding skills.
- Integrate `dbt-mcp` for lineage-aware authoring.
- **Acceptance Criteria**: Can generate a staging model + YAML from a source table via `/dbt-resource-factory:new-model`.

### Phase 3: Semantic Layer & MCP Integration (Weeks 6-8)

- Build `dbt-semantic-architect` L3 agents.
- Implement MetricFlow YAML generation and `dbt sl validate` integration.
- Deepen MCP usage for semantic discovery.
- **Acceptance Criteria**: Produces YAML that passes `dbt sl validate` 100% of the time.

### Phase 4: Quality & Governance Gates (Weeks 9-11)

- Build `dbt-quality-guardian` L3 agents.
- Integrate SQLFluff and `dbt-project-evaluator`.
- Implement PreToolUse hooks for automated linting/validation.
- **Acceptance Criteria**: Blocks any write that violates JSONSchema or SQLFluff standards.

---

## 5. Handoffs and Interfaces

### 5.1 Factory → Guardian

When `dbt-resource-factory` writes a file, `dbt-quality-guardian` hooks automatically trigger (PreToolUse) to ensure the new content meets project standards (linting, naming, schema) before the write is finalized.

### 5.2 Factory → Semantic Architect

`dbt-resource-factory` marks models with `access: public` and `contracts` when they are ready to be consumed. `dbt-semantic-architect` uses these flags to identify eligible models for semantic layer mapping.

### 5.3 Semantic Architect → BI

The output of `dbt-semantic-architect` (metrics and saved queries) feeds metrics for Lightdash (the `lightdash-*` plugins), creating a seamless "Metric-as-Code" workflow. Shared conventions for naming and YAML schema are enforced across all three plugins.
