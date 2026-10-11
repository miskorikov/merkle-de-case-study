# Open Questions

## 1. What steps are missing to industrialize the solution further?

The transformation logic is complete. A production pipeline would additionally need:

- **CI/CD and environments:** deployment from Git, separate dev and prod catalogs, and deployment through
  Databricks Asset Bundles or Terraform. Automated tests on pull requests should cover the logic most likely to
  break: the payload pivot, the ranking and the platform ties.
- **Orchestration and monitoring:** a scheduled job running L1 → L2 → L3 with retries, alerts, freshness checks,
  row-count monitoring, and a documented procedure for reruns and backfills. The job runs as a service principal,
  not as a personal user.
- **Incremental processing:** ingest only new files (Auto Loader or `COPY INTO`), deduplicate events by `event_id`,
  handle late arrivals, and update only the affected data (e.g. the affected years of `top_item`) instead of
  rebuilding every table.
- **Data quality:** automated checks with alerting or quarantine instead of only notebook `assert`s, including
  monitoring of known anomalies such as views before `created_at`.
- **Governance:** Unity Catalog permissions per layer, ownership and lineage, and schema-evolution rules. Today a
  new event parameter stops the run in validation, which is safe, but someone must be alerted and extend the model.
  `user_id` also needs a review for personal-data controls.


## 2. If implemented in dbt-core, how would the architecture change? Would other cloud resources be needed?

The storage layers and Databricks remain the same. The main change is how Layers 2 and 3 are built: dbt-core would
compile SQL models and execute them on Databricks compute, typically a SQL warehouse. Ingestion into Layer 1 stays
outside dbt, because dbt only transforms data that is already in the warehouse.

```text
S3 ──► ingestion job ──► layer1_raw (dbt sources)
                              │
                              ▼
                       dbt-core + dbt-databricks
                              │
                              ▼
                       layer2_curated
                              │
                              ▼
                       layer3_mart.top_item
```

- **Ingestion:** remains outside dbt, for example as a Databricks Job using `COPY INTO` or Auto Loader; Layer 1 tables are declared as dbt `sources`.
- **Transformations:** Layers 2 and 3 become SQL models with `table` or `incremental` materializations. The JSON
  parsing and the pivot are written in SQL.
- **Validation:** the notebook's assertions become dbt tests (`unique`, `not_null`, `relationships`,
  `accepted_values`), plus custom reconciliation tests and model contracts for the column types.
- **Documentation:** model and column descriptions, plus lineage, are generated with `dbt docs`.
- **Compute:** dbt runs the compiled SQL on Databricks compute instead of inside the notebook.

**A few additional resources are needed:**

| Resource | Why |
|---|---|
| A runtime for dbt-core | dbt is a command-line tool and needs somewhere to run: a `dbt` task in a Databricks Job, a CI runner, or a container |
| A SQL warehouse | The usual execution target for the `dbt-databricks` adapter |
| A service principal | Non-personal authentication of dbt to Databricks |
| Databricks Workflows (or Airflow) | Orchestrates ingestion and `dbt build`, ensuring transformations start only after ingestion succeeds |

Databricks Workflows and a SQL warehouse allow the pipeline to stay within Databricks. Airflow or an external CI/container runtime would introduce additional infrastructure to operate.

## 3. What would dbt-core bring? Upsides and downsides

**Upsides**

- Modular, reviewable SQL models with explicit dependencies through `ref()`.
- Standard tests, model contracts, documentation and lineage, kept close to the transformation code.
- Straightforward dev, CI and prod targets, and easy integration with Git-based CI.
- Standard patterns for incremental processing and SCD Type 2 history tracking.
- Easier collaboration for teams working mainly in SQL.

**Downsides**

- Another tool to maintain, along with its adapter and Python environment.
- Ingestion still stays outside dbt, which creates an orchestration dependency between two tools.
- SQL with Jinja templating is less natural than PySpark for procedural or complex logic, such as parsing and
  pivoting the event payload.
- dbt-core has no built-in scheduler or operational UI; those come from the orchestrator or from dbt Cloud.
- For two sources and one mart the notebook is simpler. dbt becomes more valuable as the project grows.

## 4. Effort estimate for implementing the solution in dbt-core

I would estimate approximately 2 person-days to reproduce the current case study implementation as-is in dbt-core.

| Work item | Effort |
|---|---|
| dbt project setup, authentication and compute | 0.5 day |
| Ingestion job and dbt `sources` | 0.5 day |
| Layer 2 and Layer 3 models | 0.5 day |
| Tests, result validation and documentation | 0.5 day |
| **Total** | **≈ 2 days** |

For a more production-oriented implementation, I would add approximately additional 2 person-days for CI/CD, orchestration and monitoring, incremental processing, late-arriving data and backfill handling, and stronger automated data-quality checks. This would bring the total to approximately 4 person-days.
