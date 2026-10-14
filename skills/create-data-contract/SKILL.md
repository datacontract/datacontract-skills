---
name: create-data-contract
description: "Data Product Builders and Owners. Creates a new data contract in the Open Data Contract Standard (ODCS), either data-first by importing it from an existing table, SQL DDL, dbt manifest, Avro, JSON Schema, or Excel template with the Data Contract CLI, or contract-first from requirements described in plain language, then fills in descriptions, semantics, keys, servers, team, and terms, and lints it until it is valid. Use when the user wants to create, write, draft, scaffold, or generate a data contract, turn a table or schema into a data contract, or fix lint errors in an ODCS file."
---

# Create a Data Contract

**Persona:** data engineer, data product owner, analytics engineer
**Trigger:** at the start of a new data product, or to put an existing table under contract
**Output:** a valid `*.odcs.yaml` file that passes `datacontract lint`

A good data contract is more than a schema dump: it says what the data means, who owns it, how to reach it, and what consumers can rely on. This skill produces that, in ODCS, and leaves the file valid.

## Step 0: CLI

Follow the `datacontract-cli` skill, Step 1. Run `datacontract --version`; install only after the user agrees.

## Step 1: Pick the approach

Ask which one fits, unless the request already says:

- **Data-first:** the data already exists (a table, a DDL file, a dbt model, an Avro or JSON Schema, an Excel template). Import it, then enrich it. Fastest way to a contract that matches reality.
- **Contract-first:** the data product does not exist yet. Start from the consumer requirements and write the contract, then generate code from it (see `export-data-contract`).

Also settle the file path. Use the repository's convention if there is one (look for existing `*.odcs.yaml` or `datacontract.yaml` files); otherwise propose `contracts/<id>.odcs.yaml`.

## Step 2a: Data-first, import

Pick the importer for the source. Common ones:

```bash
# SQL DDL file
datacontract import sql --source ddl.sql --dialect snowflake --output contracts/orders.odcs.yaml

# dbt manifest (run `dbt parse` or `dbt compile` first to get target/manifest.json)
datacontract import dbt --source target/manifest.json --model orders --output contracts/orders.odcs.yaml

# live sources (need credentials, see the datacontract-cli skill, Step 2)
datacontract import snowflake --source <account> --database ORDER_DB --schema PUBLIC --output contracts/orders.odcs.yaml
datacontract import databricks --table catalog.schema.orders --output contracts/orders.odcs.yaml
datacontract import bigquery --project my-project --dataset sales --table orders --output contracts/orders.odcs.yaml
datacontract import postgres --source localhost --database shop --schema public --table orders --output contracts/orders.odcs.yaml

# files and schemas
datacontract import avro --source orders.avsc --output contracts/orders.odcs.yaml
datacontract import jsonschema --source orders.schema.json --output contracts/orders.odcs.yaml
datacontract import excel --source orders.xlsx --output contracts/orders.odcs.yaml
datacontract import parquet --source orders.parquet --output contracts/orders.odcs.yaml
```

Run `datacontract import --help` for the full list (Glue, Unity Catalog, Iceberg, Protobuf, Power BI, OData, OpenAPI, Trino, Oracle, SQL Server, S3, ADLS, GCS, ...) and `datacontract import <format> --help` for its options. Pass `--id` and `--owner` when you know them. Importing from a live source reads only metadata, but still ask before connecting (see the conventions in the `datacontract-cli` skill).

Live importers and the data file importers (`csv`, `parquet`, `json`) also write the `servers` block. For DDL and schema imports, add it yourself in Step 3.

Data file importers profile the sample they read and **overfit**: they set `required` and `unique` on every column that happens to be complete and distinct, and `minimum`/`maximum` to the sample's range. Review each of these and keep only what is a real guarantee (a status column is not unique; an amount has no maximum of 4999).

## Step 2b: Contract-first, write from requirements

Ask for, or derive from what the user gave you:

- the purpose and the intended consumers and use cases,
- the entities (one ODCS `schema` object per table, topic, or file) and their fields with meaning, type, and whether they are mandatory,
- the keys and the relationships between entities,
- the platform where the data will live (determines `servers` and physical types),
- the owning team.

Start from `datacontract init contracts/<id>.odcs.yaml` (a minimal valid template) and fill it in, or write the file directly using the skeleton in Step 3. Do not invent business facts: if a description, unit, or allowed value set is not known, leave a `TODO` in the description and list it in the summary for the user.

## Step 3: Enrich the contract

An imported contract has names and types, little else. Bring every contract to this level:

```yaml
apiVersion: v3.2.0
kind: DataContract
id: orders                         # stable, unique, URL-safe
name: Orders
version: 1.0.0                     # semantic version of the contract
status: draft                      # proposed | draft | active | deprecated | retired
domain: sales
description:
  purpose: What the data is for and which questions it answers.
  usage: How consumers should use it, and how not.
  limitations: Known gaps, history cut-offs, latency.
schema:
  - name: orders                   # logical name
    physicalName: orders           # table, view, topic, or file name, if different
    physicalType: table
    description: One row per order since 2020-01-01, including cancelled orders.
    properties:
      - name: order_id
        logicalType: string        # string | integer | number | boolean | date | timestamp | time | object | array
        physicalType: UUID         # native type on the server
        description: Internal order identifier.
        primaryKey: true
        examples: ["99e8bb10-3785-4634-9664-8dc79eb69d43"]
      - name: order_total
        logicalType: integer
        physicalType: INTEGER
        description: Order total in cents, including tax, after discounts.
        required: true
      - name: customer_email
        logicalType: string
        physicalType: TEXT
        classification: confidential
        tags: ["pii"]
      - name: order_status
        logicalType: string
        physicalType: TEXT
        required: true
        enum:                      # closed set of allowed values
          - value: pending
          - value: paid
          - value: shipped
            description: Handed over to the carrier.
          - value: delivered
          - value: cancelled
servers:
  - server: production
    type: snowflake                # postgres | databricks | bigquery | s3 | kafka | ...
    environment: prod
    account: my-account
    database: ORDER_DB
    schema: PUBLIC
team:
  name: sales
  members:
    - username: jane@example.com
      role: owner
slaProperties:
  - property: freshness
    value: "24"
    unit: hours
    element: orders.order_timestamp
```

Checklist:

- **Descriptions** for the contract, every schema, and every property. Say what a row is (the grain), the unit of every amount, the time zone of every timestamp, and what each status value means. These descriptions are what consumers and AI agents read.
- **Types:** `logicalType` on every property; `physicalType` when the platform is known. On catalog-backed servers (Snowflake, BigQuery, Databricks, Postgres, ...) the test compares `physicalType` with the real column type, so it must match exactly.
- **Keys:** `primaryKey` (with `primaryKeyPosition` for composite keys), `unique`, `required`, and `relationships` with `type: foreignKey` and `to: other_schema.field`.
- **Constraints:** `logicalTypeOptions` (`minLength`, `maxLength`, `pattern`, `minimum`, `maximum`, `format`) and `enum` (a list of `- value: ...` entries, optionally with `label` and `description`) where the domain is closed.
- **Sensitive data:** `classification` and a `pii` tag on personal data fields. Never put real personal data in `examples`.
- **Servers:** connection details only, no credentials. Use `${VAR}` references for values that differ per environment.
- **Team and status:** an owner, and `status: draft` until consumers have agreed.
- Quality rules and service levels: add the obvious ones (row count, freshness); for a thorough set use the `quality-rules` skill.

Before writing an attribute you are not sure exists in ODCS, look it up in the [ODCS reference](https://bitol-io.github.io/open-data-contract-standard/latest/) or the [CLI docs on schemas](https://docs.datacontract.com/schema). `datacontract lint` rejects unknown attributes; custom metadata goes into `customProperties` (`- property: <name>` / `value: <value>`).

## Step 4: Lint until valid

```bash
datacontract lint contracts/orders.odcs.yaml --all-errors
```

Fix every error and run it again. Typical causes: a misspelled attribute, a value outside an enum (`status`, `logicalType`, `classification`, quality `dimension`), a missing required attribute (`apiVersion`, `kind`, `id`, `version`, `status`), a `relationships.to` that points to a field that does not exist, and `null` inside an `enum`.

## Step 5: Verify against the data (data-first, optional)

If the data exists and the user agrees to connect, run a quick check that the imported contract matches reality:

```bash
datacontract test contracts/orders.odcs.yaml --server production --metadata-only
```

`--metadata-only` checks field presence and types without scanning rows. For a full run and for debugging failures, follow the `test-data-contract` skill.

## Step 6: Hand over

Report:

- the file path, the number of schemas and properties,
- what you filled in from facts the user gave you, and what is a `TODO` that only the owner can answer (units, meanings, allowed values, owner),
- the lint result,
- the next steps: review it visually with `datacontract edit <file>` (see `datacontract-editor`), add quality rules (`quality-rules`), generate code (`export-data-contract`), and run it in CI (`contract-ci`).
