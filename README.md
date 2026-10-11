# Merkle Data Engineer Case Study

This project builds a three-layer data lake on **Databricks Free Edition** from two public CSV files in S3, with a
`top_item` datamart as the final layer.

## Repository contents

| File | Content |
|---|---|
| [`merkle_de_case_study.ipynb`](merkle_de_case_study.ipynb) | **Main deliverable.** Loads the source files from S3 and contains all required DDLs, Layers 1–3, the validation and the documentation of the `top_item` datamart |
| [`data_exploration.ipynb`](data_exploration.ipynb) | Data profiling of both sources, which supports the design decisions |
| [`OPEN_QUESTIONS.md`](OPEN_QUESTIONS.md) | Answers to the open questions on industrialization and dbt-core |

## Data lake layout

```text
S3 (item.csv, event.csv)
 │   downloaded unchanged into the UC volume workspace.layer1_raw.landing
 ▼
Layer 1 — Raw       item, event
 │
 ▼
Layer 2 — Curated   item, event
 │
 ▼
Layer 3 — Mart      top_item
```

| Layer | Schema | Purpose |
|---|---|---|
| Layer 1 | `workspace.layer1_raw` | Source data as-is: original column names, all columns `STRING`, no transformations |
| Layer 2 | `workspace.layer2_curated` | `snake_case` names and proper data types. The event payload is flattened and pivoted, and `event` is partitioned by `event_year` |
| Layer 3 | `workspace.layer3_mart` | `top_item` datamart with yearly views, rank and most-used platform(s) per item |

## How to run

1. Create a Databricks Free Edition workspace.
2. Import `merkle_de_case_study.ipynb` using **Workspace → Import**, or clone this repository as a Git folder.
3. Attach the notebook to **Serverless** compute and click **Run all**. It takes about 1–2 minutes, and re-running
   it rebuilds all tables.
4. Optionally run `data_exploration.ipynb` afterwards. It reads the Layer 1 tables created by the main notebook.

The committed notebooks include small outputs from a reference run on Free Edition: sample rows, counts, validation
results and the top items per year. The results can therefore be reviewed without running the notebooks.

The catalog is configured at the top of the notebook (`CATALOG = "workspace"`, the default catalog in Free Edition).

## Source data

- **`item.csv`** is the product catalogue: 2,198 items. It is the dimension table, with `id` as its key.
- **`event.csv`** contains the tracking events: 853,640 rows. The file has **one row per event parameter**, so each
  of its 428,364 events spans one or two rows. The `event.payload` column holds JSON with `event_name`, `platform`,
  `parameter_name` and `parameter_value`.

| `event_name` | Parameters | Events |
|---|---|---:|
| `view_item` | `item_id`, `referrer` | 262,786 |
| `test_assignment` | `test_id`, `test_assignment` | 162,490 |
| `view_user_profile` | `viewed_user_id` | 3,088 |

The datasets are linked through `item.id` and the `item_id` parameter of `view_item` events. Every viewed item exists
in the catalogue.

## Design decisions

### Clarified with the interviewer

| Question | Decision |
|---|---|
| Items with the same yearly view count | They share the same rank. `RANK()` is used, producing ranks such as 1, 1, 3 |
| Several platforms tied for most views | All tied platforms are included in `most_used_platforms`, sorted alphabetically for deterministic output |
| Item/year combinations without views | Only combinations with at least one view are included |
| Prices with floating-point noise | Stored as `DECIMAL(10,2)`, following the two-decimal convention used by the target systems |

### Engineering decisions based on the data

| Topic | Decision |
|---|---|
| Item view | One `view_item` event (`event_id`) is one view. Counting raw rows would double-count views, because each view has separate `item_id` and `referrer` parameter rows |
| Platform | The `platform` value of the `view_item` event |
| Views before item `created_at` | Kept and counted, and reported as a data-quality observation, because the source doesn't show which timestamp is wrong |
| Year / time zone | `view_year` is derived from `event_time`. The source has no time-zone information, so timestamps are stored as `TIMESTAMP_NTZ` |
| Partial years | The available data from 2013 and 2018 is included, and both years are documented as partial |
| Layer 2 `event` shape | Pivoted to one row per event, with each payload parameter as its own column |
| Partitioning | `event` is partitioned by `event_year`. The datamart aggregates by year, and daily partitioning would create about 1,900 small partitions for this dataset |
| Types | IDs are `INT`, timestamps are `TIMESTAMP_NTZ`, and `test_assignment` stays `INT`, because the source doesn't define Boolean semantics for it |
| Naming | `snake_case` from Layer 2 onward, with keys named `<entity>_id` |

## Validation

The notebook ends with reconciliation checks that stop the run if any of them fails:

- **Events:** Layer 2 has exactly one row per source `event_id`, so no event is lost or duplicated.
- **Pivot:** every raw parameter row ends up in exactly one populated column of Layer 2.
- **Items:** every view refers to an existing item.
- **`top_item`:**
  - its views add up to all view events;
  - it has one row per item and year.

## Data-quality observations

- **Views before item creation:** 3,148 views occur before the item's `created_at`. They are kept, not filtered.
- **Prices:**
  - 156 items have a price of `0.00`;
  - source prices contain floating-point noise, removed by the two-decimal type.
- **IDs:** source IDs are strings formatted as decimals, such as `2512.0`.
  - The `.0` suffix is removed before casting to `INT`.
  - This is exact, and any other non-integer value would fail the cast instead of being truncated.
- **Partial years:** 2013 starts on 2013-03-09 and 2018 ends on 2018-06-01.

Detailed profiling and supporting counts are in [`data_exploration.ipynb`](data_exploration.ipynb).
