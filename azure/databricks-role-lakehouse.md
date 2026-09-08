# What is the role of Databricks? A lakehouse, not just a warehouse or a wrapper

The plain-answer version: Databricks is **neither a classic data warehouse nor a thin
wrapper — it's a "lakehouse," and you can use it two ways** (store-and-transform the data,
or compute-over-data-that-lives-elsewhere). Both are possible on the same platform.

Microsoft defines it as a **data lakehouse**: a data-management system that combines the
benefits of a data lake and a data warehouse. Source:
`learn.microsoft.com/azure/databricks/lakehouse/`.

---

## Layman's tier

Think of Databricks as a kitchen, and the data as the food.

- **"Data cloned and massaged" (WAREHOUSE style).** You bring copies of your data into the
  kitchen, clean and reshape them (the "massaging"), and serve them up for reports, BI, and
  machine learning. This is the data-warehouse flavour: data comes in, gets transformed,
  gets served.
- **"Just a wrapper" (COMPUTE style).** You point the kitchen's tools at food that already
  lives elsewhere (for example, your existing Postgres/MySQL databases, or files in Azure
  storage) and cook/analyse it **without making a copy**. This is the wrapper flavour — the
  engine does the work, it doesn't own the food.

The crucial idea: Databricks **separates the engine (compute) from the storage**. The data
files are **not** locked inside Databricks — they live in your Azure storage account in an
**open format called Delta**. Databricks gives you the cook (the processing engine), the
recipe book (the table catalogue), and the rules about who may enter the kitchen
(the governance layer).

So the answer to "warehouse, wrapper, or both?" is: **both are possible.** Same kitchen, two
ways to use it. But be precise — it isn't a classic warehouse that locks your data into a
private format, and it isn't a thin wrapper, because it genuinely adds its own storage format
and query engine.

---

## Pro tier

**It's a lakehouse.** The three pillars (per Microsoft's lakehouse overview):

| Pillar | What it does | Why it matters for "warehouse vs wrapper" |
| --- | --- | --- |
| **Apache Spark** (compute) | Massively parallel processing engine, **decoupled from storage** | The engine that "massages" data, and it can run against external data |
| **Delta Lake** (storage format) | ACID transactions, schema enforcement, time travel, over files in Azure Data Lake Storage | This is why it's **more than a wrapper**: it owns a transactional *format*, not just an orchestrator |
| **Unity Catalog** (governance) | Catalog + fine-grained access, lineage, isolation, row filters / column masking | The governance layer — the enforcement point if an AI agent queries it |

### As a warehouse (data cloned and massaged) — yes

- You ingest (batch or stream) data into Delta tables on ADLS and refine it through a
  **medallion architecture**: **bronze** (raw landing) → **silver** (cleaned/validated) →
  **gold** (curated/aggregated for consumers).
- **Delta Live Tables** provide declarative, reliable ETL/ELT pipelines — the "massaging."
- **Databricks SQL / SQL Data Warehouse** provides the SQL compute endpoint → BI/reporting,
  so it *functions as* a warehouse for analytical queries.
- **Key nuance:** the table *metadata* and catalogue live in Databricks, but the **data
  files live in ADLS in an open format**. It is **not** a clone into a separate proprietary
  store — that is exactly why Microsoft calls it a *lake*house rather than a classic
  warehouse. (The open format also matters for machine learning, which proprietary warehouse
  formats often hinder.)

### As a wrapper / compute orchestration (just the engine) — yes

- Run notebooks and jobs over data already in ADLS / files / event streams without owning it.
- Read from operational databases (Postgres / MySQL / SQL Server) via connectors.
- **Lakehouse Federation** lets you query external data sources **in place** without copying
  — this is the strongest "wrapper, no clone" form. (Exact supported sources and behaviour
  vary by version; treat as version-dependent rather than settled.)

### So both — but the framing matters

- Calling it purely a **warehouse** is wrong: it stores open-format files in the lake, not a
  private store.
- Calling it purely a **wrapper** is wrong: it ships a transactional storage format, a
  catalogue, governance, and a SQL engine — a thin wrapper wouldn't.

### The freshness / "stale copy" trade-off

Because you already run operational databases, this matters:

- ETL/ELT **into Delta = copies** → there is a refresh lag and duplication, exactly like a
  classic warehouse.
- Mitigate with **streaming / change-data-capture (CDC)** for near-real-time, or **query
  federation / live connections** to read the source without a copy (no copy, no staleness,
  but the live source does the work).
- So "how stale is the Databricks data?" is a **design decision**, not a fixed property.

### Where it sits in this workspace's stack

- **Postgres / MySQL:** the operational (transactional) system of record.
- **SharePoint:** documents.
- **Databricks:** typically the **analytics / ELT / ML engine** — it ingests from those
  sources, transforms and integrates, and serves BI + data science + AI.
- Because it is a place where an AI agent might run queries, it needs **Unity Catalog** row
  filters and column masking (see `ai-permission-aware-data-access.md`).
