# dbt Development Lifecycle & Claude Code Plugin Design Analysis

> **For repository:** `yu-iskw/dbt-heros`
> **Target:** dbt Core 1.10+ and dbt Fusion (open-source only, no dbt Cloud)
> **Date:** February 14, 2026

---

## Part 1 — Deep Dive: The Complete dbt Development Lifecycle

### 1.1 What dbt Is (and What It Is Not)

dbt (Data Build Tool) is an **open-source transformation layer** that sits between your data warehouse and your BI/analytics tools. It transforms raw ingested data into clean, analytics-ready datasets using modular SQL — applying software engineering best practices (version control, modularity, testing, documentation, CI/CD) to analytics workflows.

dbt does **not** extract or load data (that is EL's job). It only **transforms** data that already lives in your warehouse.

Key architectural fact for this repo: we assume **no dbt Cloud**, relying entirely on **dbt Core 1.10+** (Python, open-source) and/or the new **dbt Fusion Engine** (Rust, source-available, local install).

---

### 1.2 The dbt Resource Taxonomy

Every file a developer writes in a dbt project is one of these resource types:

| Resource           | File Location                      | Purpose                                                   |
| ------------------ | ---------------------------------- | --------------------------------------------------------- |
| **Model**          | `models/**/*.sql`                  | SELECT query → materializes as view/table/incremental/etc |
| **Source**         | `models/**/_sources.yml`           | Declares raw upstream tables with freshness checks        |
| **Seed**           | `seeds/*.csv`                      | Static reference data loaded into warehouse               |
| **Snapshot**       | `snapshots/*.sql`                  | SCD Type-2 history tracking (timestamp or check strategy) |
| **Test**           | `tests/*.sql` + YAML `tests:` keys | Generic (YAML) or singular (SQL) data assertions          |
| **Unit Test**      | `unit_tests:` in YAML (≥ v1.8)     | Pure SQL logic tests with mock data, no warehouse         |
| **Macro**          | `macros/**/*.sql`                  | Reusable Jinja/SQL templates (DRY code)                   |
| **Analysis**       | `analyses/*.sql`                   | Ad-hoc SQL compiled but not materialized                  |
| **Documentation**  | `docs/*.md` + `description:` keys  | Docs blocks and column descriptions                       |
| **Semantic Model** | `semantic_models:` in YAML         | MetricFlow entity/measure/dimension definitions           |
| **Metric**         | `metrics:` in YAML                 | Business KPIs derived from semantic models                |
| **Saved Query**    | `saved_queries:` in YAML           | Reusable MetricFlow query configurations                  |
| **Exposure**       | `exposures:` in YAML               | Downstream consumers (dashboards, notebooks)              |
| **Function (UDF)** | `functions/*.sql` (≥ v1.11)        | User-defined SQL/Python warehouse functions               |
| **Catalog**        | `catalogs.yml` (≥ v1.10)           | Iceberg/external catalog integrations                     |

---

### 1.3 The Full Development Lifecycle (10 Phases)

#### Phase 1 — Project Initialization

```bash
dbt init <project-name>
```

Creates: `dbt_project.yml` (project config), `profiles.yml` (~/.dbt/), `packages.yml`, model directories.

Key config decisions at this stage:

- Target adapter (Snowflake, BigQuery, Redshift, Databricks, Postgres, DuckDB)
- `model-paths`, `seed-paths`, `snapshot-paths`, `macro-paths`
- Default materializations per directory
- `require-dbt-version: ">=1.10.0"`
- `flags:` behavior change opt-ins for Fusion readiness

**dbt Core 1.10+ specific:** `catalogs.yml` for Iceberg; `anchors:` top-level YAML key to avoid parser warnings; behavior flag `validate_macro_args`.

---

#### Phase 2 — Package Management

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]
  - package: dbt-labs/dbt_project_evaluator
    version: [">=0.8.0"]
  - package: elementary-data/elementary
    version: [">=0.13.0"]
  - package: calogica/dbt_expectations
    version: [">=0.10.0"]
  - git: https://github.com/dbt-labs/dbt-audit-helper.git
    revision: main
```

```bash
dbt deps   # install; Fusion does this automatically
dbt clean  # remove artifacts
```

Essential open-source packages:

- **`dbt_utils`** — surrogate keys, star schema macros, pivot, date logic
- **`dbt_expectations`** — Great Expectations-style schema tests
- **`dbt_project_evaluator`** — DAG structure and best-practice enforcement
- **`audit_helper`** — compare model outputs before/after refactor
- **`elementary`** — data observability (anomaly detection, test monitoring)
- **`dbt_date`** — date spine utilities
- **`dbt_constraints`** — database constraint generation from tests

---

#### Phase 3 — Data Modeling (The Core Loop)

**Directory structure (recommended layering):**

```text
models/
├── staging/           # stg_* — 1:1 with source tables, rename/recast only
│   ├── stripe/
│   │   ├── _stripe__sources.yml
│   │   ├── stg_stripe__customers.sql
│   │   └── stg_stripe__payments.sql
├── intermediate/      # int_* — business logic combinations
│   ├── int_orders_joined.sql
├── marts/             # dim_* fct_* — analytics-ready
│   ├── finance/
│   │   ├── dim_customers.sql
│   │   ├── fct_orders.sql
│   │   └── _finance__models.yml
├── semantic/          # semantic models + metrics
│   └── _semantic_models.yml
```

**Materializations:**

| Type                | When to Use                                                     |
| ------------------- | --------------------------------------------------------------- |
| `view`              | Default; always fresh; no storage cost                          |
| `table`             | Expensive queries; downstream consumers need speed              |
| `incremental`       | Large tables; append/merge new rows only                        |
| `ephemeral`         | CTEs inlined into downstream models; no object created          |
| `materialized_view` | Platform-native MVs (Snowflake, Databricks, BigQuery, Redshift) |
| `dynamic_table`     | Snowflake/Databricks near-real-time (Fusion-supported)          |

**Incremental strategies** (`append`, `merge`, `delete+insert`, `insert_overwrite`, `microbatch` ≥ v1.9):

```sql
-- models/marts/fct_orders.sql
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge',
    on_schema_change='append_new_columns'
  )
}}
SELECT ...
{% if is_incremental() %}
  WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**Model contracts (≥ v1.5):**

```yaml
models:
  - name: fct_orders
    contract:
      enforced: true
    access: public
    columns:
      - name: order_id
        data_type: varchar
        constraints:
          - type: not_null
```

**Model versioning (≥ v1.5):**

```yaml
models:
  - name: fct_orders
    latest_version: 2
    versions:
      - v: 2
      - v: 1
        defined_in: fct_orders_v1
```

---

#### Phase 4 — Sources & Freshness

```yaml
# models/staging/stripe/_stripe__sources.yml
version: 2
sources:
  - name: stripe
    database: raw
    schema: stripe
    freshness:
      warn_after: { count: 12, period: hour }
      error_after: { count: 24, period: hour }
    loaded_at_field: _fivetran_synced
    tables:
      - name: customers
        columns:
          - name: id
            tests:
              - not_null
              - unique
```

```bash
dbt source freshness   # check source lag
```

dbt Core 1.10+ allows `loaded_at_query` in source config for custom freshness SQL.

---

#### Phase 5 — Testing

**Generic tests (YAML):**

```yaml
models:
  - name: fct_orders
    columns:
      - name: order_id
        tests:
          - not_null
          - unique
      - name: customer_id
        tests:
          - relationships:
              to: ref('dim_customers')
              field: customer_id
      - name: status
        tests:
          - accepted_values:
              values: ["pending", "shipped", "cancelled"]
```

**Singular tests (SQL files in `tests/`):**

```sql
-- tests/assert_order_revenue_positive.sql
SELECT * FROM {{ ref('fct_orders') }}
WHERE revenue < 0
```

**Unit tests (≥ v1.8, no warehouse required):**

```yaml
unit_tests:
  - name: test_calculate_mrr
    model: fct_mrr
    given:
      - input: ref('stg_subscriptions')
        rows:
          - { sub_id: 1, plan: "pro", amount: 99 }
    expect:
      rows:
        - { sub_id: 1, mrr: 99 }
```

**Extended tests via packages:**

```yaml
# dbt_expectations
- dbt_expectations.expect_column_values_to_be_between:
    min_value: 0
    max_value: 1000
- dbt_expectations.expect_table_row_count_to_be_between:
    min_value: 1
```

---

#### Phase 6 — Snapshots

```sql
-- snapshots/scd_customers.sql
{% snapshot scd_customers %}
  {{
    config(
      target_schema='snapshots',
      unique_key='customer_id',
      strategy='timestamp',
      updated_at='updated_at',
      dbt_valid_to_current='9999-12-31'
    )
  }}
  SELECT * FROM {{ source('crm', 'customers') }}
{% endsnapshot %}
```

Snapshots implement SCD Type-2, preserving history with `dbt_valid_from`/`dbt_valid_to` columns.

---

#### Phase 7 — Documentation

```yaml
models:
  - name: fct_orders
    description: "{{ doc('fct_orders') }}"
    columns:
      - name: order_id
        description: Unique surrogate key for orders
```

```markdown
{% docs fct_orders %}

# Orders Fact Table

Central fact table for the order domain.
Grain: one row per order.
{% enddocs %}
```

```bash
dbt docs generate   # → manifest.json + catalog.json
dbt docs serve      # local docs site on :8080
```

---

#### Phase 8 — Semantic Layer (MetricFlow, ≥ v1.6)

```yaml
# In _semantic_models.yml (new Fusion-era embedded format)
models:
  - name: fct_orders
    semantic_models:
      - name: orders
        model: ref('fct_orders')
        entities:
          - name: order
            type: primary
            expr: order_id
          - name: customer
            type: foreign
            expr: customer_id
        dimensions:
          - name: order_date
            type: time
            type_params:
              time_granularity: day
        measures:
          - name: revenue
            agg: sum
            expr: amount

metrics:
  - name: monthly_revenue
    type: simple
    type_params:
      measure:
        name: revenue
    time_granularity: month

saved_queries:
  - name: revenue_by_month
    query_params:
      metrics: [monthly_revenue]
      group_by:
        - Dimension('order__order_date')
```

**dbt MCP Server** exposes semantic models, metrics, and lineage to AI agents (including Claude Code) via `execute_sql`, `list_metrics`, `get_semantic_layer_query`, and project introspection tools.

---

#### Phase 9 — User-Defined Functions (≥ v1.11)

```sql
-- functions/calculate_ltv.sql
{{
  config(
    language='sql',
    return_type='numeric',
    arguments=[
      {'name': 'revenue', 'data_type': 'numeric'},
      {'name': 'churn_rate', 'data_type': 'numeric'}
    ]
  )
}}
SELECT {{ revenue }} / NULLIF({{ churn_rate }}, 0)
```

```sql
-- Reference in models:
SELECT {{ function('calculate_ltv') }}(revenue, churn_rate) AS ltv
FROM {{ ref('fct_customers') }}
```

---

#### Phase 10 — CI/CD (Open-Source, No dbt Cloud)

**Slim CI pattern (GitHub Actions):**

```yaml
# .github/workflows/dbt-ci.yml
on:
  pull_request:
    branches: [main]

jobs:
  dbt-ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install dbt-core dbt-snowflake sqlfluff sqlfluff-templater-dbt

      # Download production manifest
      - run: |
          aws s3 cp s3://$ARTIFACT_BUCKET/manifest.json ./state/manifest.json

      # SQLFluff lint
      - run: sqlfluff lint models/ --dialect snowflake --templater dbt

      # Slim CI: build + test only modified nodes
      - run: |
          dbt build \
            --select state:modified+ \
            --defer \
            --state ./state/ \
            --profiles-dir ./ \
            --target ci

      # Upload new manifest
      - run: |
          aws s3 cp ./target/manifest.json s3://$ARTIFACT_BUCKET/manifest.json
```

**Key Slim CI concepts:**

- `state:modified+` — only build changed models and their downstream dependents
- `--defer --state ./state/` — use prod manifest to resolve unbuilt upstream models
- PR-scoped schemas: `dbt_cloud_pr_{PR_ID}_{ENV_ID}` or custom `generate_schema_name` macro
- Artifact storage: S3, GCS, or Azure Blob for `manifest.json` sharing across environments

**Other open-source orchestrators:** Apache Airflow (`DbtTaskGroup`), Prefect (`DbtCoreOperation`), Dagster (first-class dbt integration), Mage.

---

### 1.4 Key Toolchain (Open-Source)

| Tool                      | Purpose                                                | Integration Point                                  |
| ------------------------- | ------------------------------------------------------ | -------------------------------------------------- |
| **dbt Core**              | Python-based execution engine                          | `dbt run/build/test/docs`                          |
| **dbt Fusion CLI**        | Rust-based next-gen engine                             | `dbt build` (drop-in for Core)                     |
| **dbt VS Code Extension** | Fusion-powered IDE (IntelliSense, LSP)                 | Local development                                  |
| **SQLFluff**              | SQL linter + formatter                                 | Pre-commit / CI                                    |
| **dbt-project-evaluator** | DAG best-practice audit                                | `dbt build --select package:dbt_project_evaluator` |
| **dbt-osmosis**           | YAML management (sync columns, propagate descriptions) | `dbt-osmosis yaml refactor`                        |
| **dbt-autofix**           | Fusion migration (resolve deprecation warnings)        | `dbt-autofix run`                                  |
| **Elementary OSS**        | Data observability reports + alerting                  | `edr report`, `edr send-report`                    |
| **pre-commit**            | Git hooks for SQLFluff, YAML lint, secrets             | `.pre-commit-config.yaml`                          |
| **dbt-codegen**           | Generate YAML from warehouse schemas                   | `dbt run-operation generate_source`                |

---

### 1.5 dbt Fusion Engine — What Changes

| Aspect            | dbt Core                             | dbt Fusion                                          |
| ----------------- | ------------------------------------ | --------------------------------------------------- |
| Language          | Python                               | Rust (compiled binary)                              |
| Parse speed       | Baseline                             | ~10x faster                                         |
| SQL comprehension | None (pass-through)                  | Native multi-dialect SQL analysis + static analysis |
| YAML validation   | Warnings (v1.10+)                    | Errors on invalid configs                           |
| Manifest version  | v12                                  | v20 (not backward-compatible)                       |
| Threading         | `--threads N` (strict limit)         | Auto-optimized per adapter                          |
| Install           | `pip install dbt-core dbt-<adapter>` | `curl .../install.sh \| sh`                         |
| VS Code LSP       | None                                 | Full IntelliSense, go-to-definition                 |
| Compile errors    | Stops on first error                 | Continues compiling rest of DAG                     |
| Fusion readiness  | N/A                                  | Requires resolving all deprecation warnings         |

**Migration path:** `dbt-autofix run` → resolves deprecation warnings → test on Core 1.10/1.11 → validate with Fusion.

---

## Part 2 — Five Claude Code Plugin Design Candidates

> Each candidate is evaluated on 10 criteria (0–100). Aspect weights reflect the priorities of an open-source dbt project.

### Evaluation Criteria

| #   | Criterion                       | Description                                               |
| --- | ------------------------------- | --------------------------------------------------------- |
| 1   | **Coverage**                    | Breadth of lifecycle phases addressed                     |
| 2   | **Cohesion**                    | How tightly the plugin's concerns belong together         |
| 3   | **Hierarchy Fit**               | How cleanly it maps to the 5-level agent hierarchy        |
| 4   | **Implementation Simplicity**   | Lower complexity → higher score                           |
| 5   | **Extensibility**               | Ease of adding new capabilities over time                 |
| 6   | **Separation of Concerns**      | How well it avoids leaking into adjacent plugins          |
| 7   | **Developer Experience**        | Daily usefulness for an analytics engineer                |
| 8   | **Complementarity**             | How well it fills gaps left by existing Lightdash plugins |
| 9   | **Fusion/Core 1.10+ Alignment** | Targets latest dbt features and migration path            |
| 10  | **Maintainability**             | Long-term ease of updates                                 |

---

### Candidate A — `dbt-resource-factory`

**Design Philosophy:** _The Daily Authoring Assistant_
Generate and scaffold every dbt resource file from intent — the "factory pattern" where an analytics engineer describes what they want and the plugin produces production-quality SQL, YAML, and docs.

**Analogy:** Like `dbt-codegen` but AI-powered, context-aware, and opinionated about best practices.

#### Lifecycle Coverage

- ✅ Phase 1: Project init & config scaffolding
- ✅ Phase 2: Package recommendation
- ✅ Phase 3: Model SQL generation (with materialization configs)
- ✅ Phase 4: Source YAML scaffolding
- ✅ Phase 5: Test YAML + unit test generation
- ✅ Phase 6: Snapshot scaffolding
- ✅ Phase 7: Documentation generation (doc blocks, descriptions)
- ✅ Phase 9: UDF scaffolding (v1.11+)
- ⚠️ Phase 8: Semantic model scaffolding (basic only)
- ❌ Phase 10: CI/CD (out of scope)

#### Architecture (5-Level Hierarchy)

```text
L0 — central-orchestrator
  └─ L1 — dbt-resource-factory-orchestrator
        ├─ L2 — sql-modeling-team
        │    ├─ L3 — model-architect     (design model structure, DAG planning)
        │    └─ L3 — sql-generator       (write model SQL with Jinja)
        ├─ L2 — schema-team
        │    ├─ L3 — yaml-scaffolder     (generate/update schema YAML)
        │    ├─ L3 — test-suggester      (suggest and write generic+singular tests)
        │    └─ L3 — unit-test-author    (write unit_tests blocks)
        └─ L2 — resource-team
             ├─ L3 — source-definer      (scaffold _sources.yml)
             ├─ L3 — snapshot-builder    (scaffold SCD snapshots)
             ├─ L3 — seed-manager        (scaffold seed + YAML)
             └─ L3 — udf-author          (scaffold functions/ for v1.11+)
```

#### Skills

- `scaffold-model-sql` — Generate staging/intermediate/mart SQL from intent
- `generate-schema-yaml` — Create or update schema.yml for a model
- `suggest-generic-tests` — Recommend tests based on column semantics (PKs, FKs, dates)
- `write-unit-tests` — Generate `unit_tests:` blocks with mock data
- `scaffold-source-yaml` — Generate `_sources.yml` from warehouse introspection
- `scaffold-snapshot` — Generate snapshot SQL with correct strategy
- `generate-doc-blocks` — Write `{% docs %}` blocks and column descriptions
- `scaffold-udf` — Generate function files for v1.11+ UDFs

#### MCP Integration

```json
{
  "mcpServers": {
    "dbt": {
      "command": "uvx",
      "args": ["dbt-mcp"],
      "env": { "DBT_PROJECT_DIR": "${workspaceFolder}" }
    }
  }
}
```

Uses `get_model_lineage_dev`, `get_macro_details`, `get_seed_details` for project-aware generation.

#### Hooks

- `PreToolUse[Write]`: Validate generated YAML against JSONSchema before writing
- `PreToolUse[Bash]`: Guard against `dbt run --full-refresh` in production targets
- `PostToolUse[Write]`: Audit log all resource file writes

#### Slash Commands

```text
/dbt-resource-factory:new-model       — Scaffold model + YAML + tests from description
/dbt-resource-factory:new-source      — Generate source YAML from warehouse table
/dbt-resource-factory:add-tests       — Suggest + generate tests for existing model
/dbt-resource-factory:new-snapshot    — Scaffold snapshot SQL
/dbt-resource-factory:enrich-docs     — Add descriptions and doc blocks
/dbt-resource-factory:new-udf         — Scaffold UDF for v1.11+
```

#### Scoring

| Criterion                   | Score    | Rationale                                           |
| --------------------------- | -------- | --------------------------------------------------- |
| Coverage                    | 70       | Deep on authoring phases; no CI/CD                  |
| Cohesion                    | **92**   | Tightly unified around resource generation          |
| Hierarchy Fit               | **90**   | 3 teams, clean L3 boundaries, no lateral calls      |
| Implementation Simplicity   | 78       | Moderate — LLM generation + YAML validation         |
| Extensibility               | **88**   | Add resource types as new L3 sub-agents             |
| Separation of Concerns      | **87**   | Clear: create resources, not run or deploy them     |
| Developer Experience        | **95**   | Addresses the most frequent daily task              |
| Complementarity             | **90**   | Fills the gap left by Lightdash plugins entirely    |
| Fusion/Core 1.10+ Alignment | 85       | Handles contracts, UDFs, unit tests, catalog YAML   |
| Maintainability             | 82       | Each agent is isolated; easy to update individually |
| **TOTAL (avg)**             | **85.7** |                                                     |

**Verdict:** Highest developer experience score. The most impactful plugin for daily use — it directly accelerates the most common analytics engineering task: writing new dbt resources from scratch.

---

### Candidate B — `dbt-quality-guardian`

**Design Philosophy:** _The Code Quality & Governance Enforcer_
A dedicated quality enforcement plugin: SQL style linting (SQLFluff), DAG structure auditing (dbt-project-evaluator), model contract enforcement, schema validation for Fusion readiness, and deprecation scanning.

**Analogy:** An automated code reviewer with deep dbt knowledge.

#### Lifecycle Coverage

- ⚠️ Phase 1: Project structure validation
- ❌ Phase 2–3: Resource authoring (out of scope)
- ✅ Phase 4: Source freshness checks
- ✅ Phase 5: Test coverage analysis (what's missing)
- ❌ Phase 6–7: Authoring (out of scope)
- ✅ Phase 8: Semantic model contract validation
- ❌ Phase 9–10: UDF & CI/CD (out of scope)
- ✅ Cross-cutting: SQL linting, DAG best-practices, deprecation warnings

#### Architecture

```text
L0 — central-orchestrator
  └─ L1 — dbt-quality-guardian-orchestrator
        ├─ L2 — lint-team
        │    ├─ L3 — sql-linter          (SQLFluff: style, formatting, complexity)
        │    └─ L3 — yaml-validator      (JSONSchema validation, v1.10+ deprecations)
        ├─ L2 — dag-audit-team
        │    ├─ L3 — structure-auditor   (dbt-project-evaluator: DAG issues)
        │    └─ L3 — coverage-inspector  (test coverage, source freshness gaps)
        └─ L2 — governance-team
             ├─ L3 — contract-enforcer   (model contracts, access modifiers)
             ├─ L3 — deprecation-scanner (Fusion readiness, behavior flags)
             └─ L3 — naming-police       (stg_/int_/dim_/fct_ conventions)
```

#### Skills

- `run-sqlfluff-lint` — Execute SQLFluff against changed models, parse results
- `audit-dag-structure` — Run dbt-project-evaluator, summarize violations
- `validate-yaml-schema` — Check YAML files against dbt JSONSchema (v1.10+)
- `check-test-coverage` — Identify models missing not_null/unique/relationship tests
- `scan-deprecations` — Find deprecated configs incompatible with Fusion
- `enforce-naming-conventions` — Validate stg*/int*/dim*/fct* naming rules
- `validate-contracts` — Check enforced contracts have correct column types

#### MCP Integration

Uses `get_model_lineage_dev` to understand downstream impact of contract violations.

#### Hooks

- `PreToolUse[Write][*.sql]`: Auto-lint SQL before committing (SQLFluff format check)
- `PreToolUse[Write][*.yml]`: JSONSchema validation before writing YAML
- `PostToolUse[Bash][dbt build]`: Parse test failures, route to coverage-inspector

#### Slash Commands

```text
/dbt-quality-guardian:lint            — SQLFluff lint changed models
/dbt-quality-guardian:audit           — Full dbt-project-evaluator run
/dbt-quality-guardian:coverage        — Test coverage report
/dbt-quality-guardian:check-fusion    — Fusion readiness scan
/dbt-quality-guardian:enforce-naming  — Naming convention audit
```

#### Scoring

| Criterion                   | Score    | Rationale                                             |
| --------------------------- | -------- | ----------------------------------------------------- |
| Coverage                    | 58       | Focused on quality; no authoring/deployment           |
| Cohesion                    | **94**   | All quality concerns; very tight domain               |
| Hierarchy Fit               | **91**   | Clean 3-team, 7-agent structure                       |
| Implementation Simplicity   | 80       | External tool integrations (SQLFluff, evaluator)      |
| Extensibility               | 84       | Add new quality rules as L3 agents                    |
| Separation of Concerns      | **93**   | Quality only; no resource generation or deployment    |
| Developer Experience        | 84       | Valuable but not daily-use frequency                  |
| Complementarity             | **91**   | No Lightdash plugin covers code quality               |
| Fusion/Core 1.10+ Alignment | **90**   | Deprecation scanning, JSONSchema, contract validation |
| Maintainability             | **85**   | Small, focused agents; easy to add/remove rules       |
| **TOTAL (avg)**             | **85.0** |                                                       |

**Verdict:** Highest cohesion and separation of concerns. The natural complement to `dbt-resource-factory` — one generates, the other validates. Especially valuable as teams prepare for Fusion migration.

---

### Candidate C — `dbt-lifecycle-orchestrator`

**Design Philosophy:** _The Full-Stack dbt Engineer_
A comprehensive single plugin covering all 10 lifecycle phases, organized into lifecycle-phase teams: `develop → quality → deploy → observe`. Maximizes coverage at the cost of cohesion.

**Analogy:** A senior analytics engineer who can handle anything dbt-related.

#### Lifecycle Coverage

- ✅ All 10 phases covered
- Teams map to lifecycle phases rather than concern areas

#### Architecture

```text
L0 — central-orchestrator
  └─ L1 — dbt-lifecycle-orchestrator
        ├─ L2 — develop-team
        │    ├─ L3 — resource-author     (models, YAML, tests, docs)
        │    └─ L3 — macro-craftsman     (macros, packages)
        ├─ L2 — quality-team
        │    ├─ L3 — linter              (SQLFluff)
        │    └─ L3 — dag-auditor         (project-evaluator)
        ├─ L2 — deploy-team
        │    ├─ L3 — ci-configurator     (GitHub Actions, GitLab CI)
        │    └─ L3 — artifact-manager    (manifest.json, slim CI)
        └─ L2 — observe-team
             ├─ L3 — freshness-checker   (source freshness)
             └─ L3 — elementary-reporter (Elementary OSS integration)
```

#### Scoring

| Criterion                   | Score    | Rationale                                                 |
| --------------------------- | -------- | --------------------------------------------------------- |
| Coverage                    | **95**   | All lifecycle phases                                      |
| Cohesion                    | 62       | Wide scope weakens internal cohesion                      |
| Hierarchy Fit               | 70       | Risk of 400-line agent bloat; many cross-cutting concerns |
| Implementation Simplicity   | 48       | Most complex to implement correctly                       |
| Extensibility               | 70       | Adding phases is hard; concerns blur                      |
| Separation of Concerns      | 60       | L3 agents often need each other's context                 |
| Developer Experience        | 82       | Convenient single-plugin install                          |
| Complementarity             | 78       | Overlaps with other Lightdash concerns at the edges       |
| Fusion/Core 1.10+ Alignment | 84       | Can cover everything but none deeply                      |
| Maintainability             | 52       | Hard to update one phase without affecting others         |
| **TOTAL (avg)**             | **70.1** |                                                           |

**Verdict:** High coverage but the lowest scores in cohesion, implementation simplicity, and maintainability. Violates the spirit of the platform hierarchy's separation-of-concerns principle. Better to achieve similar coverage by combining Candidates A + B + D as individual plugins.

---

### Candidate D — `dbt-cicd-engineer`

**Design Philosophy:** _The DevOps Bridge for Open-Source dbt_
Focuses exclusively on the CI/CD gap that every open-source dbt team faces: generating GitHub Actions / GitLab CI workflows, configuring Slim CI with manifest state management, setting up PR-scoped schemas, and automating artifact lifecycle. Zero dbt Cloud dependency.

**Analogy:** A DevOps engineer specializing in dbt data pipelines.

#### Lifecycle Coverage

- ⚠️ Phase 1: Profile/target validation for CI environments
- ❌ Phase 2–9: Resource authoring and quality (out of scope)
- ✅ Phase 10: Full CI/CD lifecycle — lint, build, test, deploy, artifacts
- ✅ Cross-cutting: Slim CI optimization, PR automation, manifest management

#### Architecture

```text
L0 — central-orchestrator
  └─ L1 — dbt-cicd-engineer-orchestrator
        ├─ L2 — pipeline-setup-team
        │    ├─ L3 — workflow-generator  (GitHub Actions / GitLab CI YAML authoring)
        │    └─ L3 — env-configurator    (dev/ci/prod targets, profiles.yml)
        ├─ L2 — slim-ci-team
        │    ├─ L3 — manifest-manager    (S3/GCS upload/download, artifact versioning)
        │    └─ L3 — state-advisor       (state:modified+ selector strategies)
        └─ L2 — pr-automation-team
             ├─ L3 — schema-isolator     (PR-scoped schema generation, teardown macros)
             └─ L3 — quality-gate        (test result parsing, PR comment generation)
```

#### Skills

- `generate-github-actions-workflow` — Full CI/CD YAML for GitHub Actions
- `generate-gitlab-ci-config` — `.gitlab-ci.yml` for dbt projects
- `configure-slim-ci` — `state:modified+`, `--defer`, `--state` configuration
- `manage-manifest-artifact` — S3/GCS/Azure artifact storage strategies
- `scaffold-pr-schema-macro` — `generate_schema_name` for isolated PR schemas
- `setup-pre-commit` — SQLFluff + YAML lint + secrets hooks config
- `parse-ci-test-results` — Parse `manifest.json` run_results to generate PR comments

#### Hooks

- `PreToolUse[Write][*.yml/.yaml][github|gitlab]`: Validate CI YAML syntax
- `PreToolUse[Bash][dbt run]`: Block `--target prod` in non-production contexts
- `PostToolUse[Bash][dbt build]`: Parse exit code + test results, route failures

#### Slash Commands

```text
/dbt-cicd-engineer:setup-github-actions  — Generate full GHA workflow
/dbt-cicd-engineer:setup-slim-ci         — Configure state-based selective builds
/dbt-cicd-engineer:setup-pr-schemas      — Generate schema isolation macros
/dbt-cicd-engineer:explain-artifacts     — Guide on manifest.json strategy
/dbt-cicd-engineer:setup-precommit       — Scaffold .pre-commit-config.yaml
```

#### Scoring

| Criterion                   | Score    | Rationale                                                |
| --------------------------- | -------- | -------------------------------------------------------- |
| Coverage                    | 55       | Deep on CI/CD only                                       |
| Cohesion                    | **93**   | Very tight: deployment and automation concerns only      |
| Hierarchy Fit               | **89**   | 3 clean teams, well-bounded agents                       |
| Implementation Simplicity   | 75       | File generation + bash integration (moderate)            |
| Extensibility               | **87**   | Add new CI platforms as L3 agents                        |
| Separation of Concerns      | **92**   | No resource generation, no quality enforcement           |
| Developer Experience        | **88**   | Addresses the #1 pain point for open-source dbt teams    |
| Complementarity             | **94**   | Lightdash plugins have zero CI/CD coverage               |
| Fusion/Core 1.10+ Alignment | 82       | Fusion manifest v20 awareness, state-aware orchestration |
| Maintainability             | **85**   | Isolated concerns; easy to swap CI platforms             |
| **TOTAL (avg)**             | **84.0** |                                                          |

**Verdict:** Best complementarity score — fills the most urgent gap not covered anywhere in the existing plugin set. Open-source dbt teams consistently struggle with CI/CD setup, and this plugin addresses it directly with zero dbt Cloud dependency.

---

### Candidate E — `dbt-semantic-architect`

**Design Philosophy:** _The Metrics & Semantic Layer Specialist_
Dedicated plugin for the dbt Semantic Layer (MetricFlow): authoring semantic models, defining metrics, writing saved queries, validating the semantic layer, integrating with the dbt MCP server, and generating the new Fusion-era embedded YAML format. Directly complements and extends `lightdash-development`.

**Analogy:** A specialist who bridges data engineering (dbt models) and business intelligence (metrics, KPIs, semantic definitions).

#### Lifecycle Coverage

- ❌ Phase 1–7: Resource authoring (out of scope, use `dbt-resource-factory`)
- ✅ Phase 8: Full Semantic Layer lifecycle
- ⚠️ Phase 9: UDFs that support metric calculation
- ❌ Phase 10: CI/CD (use `dbt-cicd-engineer`)
- ✅ MCP: Deep dbt MCP server integration for semantic context

#### Architecture

```text
L0 — central-orchestrator
  └─ L1 — dbt-semantic-architect-orchestrator
        ├─ L2 — semantic-model-team
        │    ├─ L3 — entity-mapper        (define entities, PKs, FKs from model schema)
        │    └─ L3 — dimension-builder    (time dimensions, categorical dimensions)
        ├─ L2 — metrics-team
        │    ├─ L3 — metric-author        (simple, ratio, cumulative, derived metrics)
        │    ├─ L3 — saved-query-builder  (saved queries, exports)
        │    └─ L3 — metric-validator     (MetricFlow validation, naming rules)
        └─ L2 — governance-team
             ├─ L3 — lineage-tracer       (MCP: trace model → semantic model → metric)
             └─ L3 — mcp-integrator       (configure dbt MCP server for local Fusion)
```

#### Skills

- `generate-semantic-model` — Create semantic_models YAML from model SQL analysis
- `define-entities` — Identify and configure primary/foreign entities
- `scaffold-time-dimensions` — Create time dimensions with correct granularity configs
- `author-simple-metric` — Simple metric from a measure
- `author-derived-metric` — Ratio/derived metrics with filters
- `scaffold-saved-query` — Reusable saved queries for BI export
- `validate-metricflow` — Run MetricFlow validation (`dbt sl validate`)
- `configure-dbt-mcp` — Generate `server.json` for local dbt MCP integration
- `embed-semantic-yaml` — Convert legacy semantic YAML to new Fusion-embedded format

#### MCP Integration

The most MCP-intensive plugin — uses `get_semantic_model_details`, `execute_sql` (via Semantic Layer), `get_model_lineage_dev` to build semantic models that accurately reflect underlying model SQL.

```json
// .claude-plugin/mcp/server.json
{
  "mcpServers": {
    "dbt": {
      "command": "uvx",
      "args": ["dbt-mcp"],
      "env": {
        "DBT_PROJECT_DIR": "${workspaceFolder}",
        "DISABLE_PLATFORM_FEATURES": "true"
      }
    }
  }
}
```

#### Hooks

- `PreToolUse[Write][*semantic*|*metric*|*saved_query*]`: Validate MetricFlow YAML syntax
- `PostToolUse[Bash][dbt sl validate]`: Parse validation output, route failures to validator

#### Slash Commands

```text
/dbt-semantic-architect:generate-semantic-model  — From model SQL → semantic_model YAML
/dbt-semantic-architect:author-metrics           — Define metrics from business requirements
/dbt-semantic-architect:validate                 — Run MetricFlow validation
/dbt-semantic-architect:embed-yaml               — Migrate to Fusion embedded YAML format
/dbt-semantic-architect:setup-mcp                — Configure local dbt MCP server
/dbt-semantic-architect:trace-lineage            — Trace model → metric lineage via MCP
```

#### Scoring

| Criterion                   | Score    | Rationale                                                      |
| --------------------------- | -------- | -------------------------------------------------------------- |
| Coverage                    | 48       | Deep on semantic layer only                                    |
| Cohesion                    | **96**   | Single, very well-defined domain                               |
| Hierarchy Fit               | **94**   | 3 teams, clean bounded agents, no lateral calls                |
| Implementation Simplicity   | 76       | MetricFlow YAML is complex; MCP integration moderate           |
| Extensibility               | 82       | Add metric types as new L3 agents                              |
| Separation of Concerns      | **95**   | Zero overlap with authoring/quality/CI-CD                      |
| Developer Experience        | **90**   | Semantic Layer is notoriously hard; huge value-add             |
| Complementarity             | **97**   | Lightdash-development assumes metrics exist; this creates them |
| Fusion/Core 1.10+ Alignment | **97**   | New Fusion-embedded YAML spec, MCP server, MetricFlow GA       |
| Maintainability             | **87**   | Tight domain; well-specified MetricFlow spec to follow         |
| **TOTAL (avg)**             | **86.2** |                                                                |

**Verdict:** Highest complementarity and Fusion/Core alignment scores. Directly feeds the `lightdash-development` plugin with correctly-defined metrics and semantic models, creating a natural plugin pair. The semantic layer is the hardest part of dbt to author correctly — an AI specialist here is high-leverage.

---

## Part 3 — Comparative Scorecard

```text
                          A              B              C              D              E
                    dbt-resource-  dbt-quality-  dbt-lifecycle- dbt-cicd-     dbt-semantic-
                    factory        guardian       orchestrator   engineer      architect
─────────────────────────────────────────────────────────────────────────────────────────
Coverage            70            58             95 ★           55            48
Cohesion            92            94 ★           62             93            96 ★★
Hierarchy Fit       90            91             70             89            94 ★
Impl. Simplicity    78            80 ★           48             75            76
Extensibility       88            84             70             87            82
Separation          87            93             60             92            95 ★
Dev Experience      95 ★★         84             82             88            90
Complementarity     90            91             78             94 ★          97 ★★
Fusion Alignment    85            90             84             82            97 ★★
Maintainability     82            85             52             85            87 ★
─────────────────────────────────────────────────────────────────────────────────────────
AVERAGE             85.7          85.0           70.1           84.0          86.2 ★
```

**Rankings:** E (86.2) > A (85.7) > B (85.0) > D (84.0) > C (70.1)

---

## Part 4 — Recommended Plugin Combination Strategy

Rather than choosing one, the highest-value approach is to implement a **coordinated plugin trio** in priority order:

```text
Phase 1 (Now):    dbt-resource-factory     ← Daily authoring; highest dev experience
Phase 2 (Soon):   dbt-semantic-architect   ← Feeds lightdash-development; high Fusion alignment
Phase 3 (Later):  dbt-quality-guardian     ← Quality gates + Fusion readiness scanning
Phase 4 (Later):  dbt-cicd-engineer        ← CI/CD automation for open-source teams
Phase ∞ (Never):  dbt-lifecycle-orchestrator  ← Too broad; solve with composition
```

**Why this order:**

1. `dbt-resource-factory` delivers immediate daily value for every analytics engineer.
2. `dbt-semantic-architect` directly extends the existing Lightdash plugin investment and is uniquely positioned for the Fusion/MCP era.
3. `dbt-quality-guardian` builds the automated quality gates to enforce standards as the team grows.
4. `dbt-cicd-engineer` closes the loop on full open-source deployment automation.

These four plugins share a clean handoff:

```text
resource-factory (create) → quality-guardian (validate) → cicd-engineer (deploy) ─→ lightdash plugins (consume)
                                                                  ↑
                                         semantic-architect (define metrics) ─────┘
```

---

## Part 5 — Key Design Decisions for Any dbt Plugin

Regardless of which candidate is chosen, these decisions must be made:

1. **Read-only vs. write-safe vs. write-destructive modes** — follow the existing Lightdash pattern with explicit mode declarations in `plugin.json`
2. **dbt MCP server scope** — use local Fusion CLI mode only (no dbt Cloud dependency) with `DISABLE_PLATFORM_FEATURES=true`
3. **`--target` guardrails** — hooks must block production target access except in explicit deploy commands
4. **Fusion compat flag** — every generated YAML should pass Fusion's strict JSONSchema validation
5. **`dbt-autofix` integration** — the quality-guardian should offer a `fix` mode that runs `dbt-autofix` for Fusion migration
6. **Node selector patterns** — skills should always use proper node selectors (`--select`, `--exclude`, `--models`) rather than full-project runs
7. **`dbt_project.yml` versioning** — generated configs must include `require-dbt-version: ">=1.10.0"` and explicit behavior flags
