---
name: export-data-contract
description: "Data Product Builders. Generates code and artifacts from an ODCS data contract with the Data Contract CLI: SQL DDL, dbt models, sources, staging SQL and tests (`datacontract dbt sync`), Avro, JSON Schema, Protobuf, Pydantic, SQLAlchemy, Spark, Iceberg, BigQuery schemas, Great Expectations, SodaCL, DQX, HTML, Markdown, Mermaid, Excel, and an HTML catalog of many contracts, then wires the output into the user's project. Use when the user wants to implement a data product from a contract, create tables, generate dbt files or tests, get a schema or model class for a contract, render documentation, or keep code in sync with the contract."
---

# Export a Data Contract

**Persona:** data engineer, analytics engineer, software engineer
**Trigger:** contract-first implementation, or whenever the contract changed and generated code must follow
**Output:** generated files in the user's project, consistent with the contract

## Step 0: CLI

Follow the `datacontract-cli` skill, Step 1, and `datacontract lint` the contract: exporting an invalid contract produces wrong code.

## Step 1: Pick the target

Map the user's goal to an exporter:

| Goal | Command |
|---|---|
| Create tables | `export sql --dialect <snowflake\|postgres\|databricks\|bigquery\|sqlserver\|mysql\|trino\|oracle\|clickhouse>` (or `--server <name>` to take the dialect from the server) |
| Select statement for consumers | `export sql-query` |
| dbt project (provider) | `dbt sync` (preferred, updates an existing project in place), or `export dbt-models` |
| dbt project (consumer) | `export dbt-sources`, `export dbt-staging-sql` |
| Event schemas | `export avro`, `export avro-idl`, `export protobuf`, `export jsonschema`, `export xsd` |
| Application code | `export pydantic-model`, `export sqlalchemy`, `export go` |
| Platform schemas | `export spark`, `export iceberg`, `export bigquery` |
| Quality engines | `export great-expectations`, `export sodacl`, `export dqx`, `export data-caterer` (test data) |
| Documentation | `export html`, `export markdown`, `export mermaid`, `export dbml`, `export rdf` |
| Business review | `export excel` (ODCS Excel template; import back with `import excel`) |
| Your own format | `export custom --template my.jinja` |
| Many contracts as a website | `datacontract catalog --files "contracts/**/*.yaml" --output catalog/` |

Check `datacontract export <format> --help` for format-specific options (dialect, server, schema name, engine options) before running.

## Step 2: Export

Write to a file with `--output`; without it, the result goes to stdout.

```bash
datacontract export sql --server production --output models/ddl/orders.sql contracts/orders.odcs.yaml
datacontract export pydantic-model --output app/models/orders.py contracts/orders.odcs.yaml
datacontract export html --output docs/orders.html contracts/orders.odcs.yaml
```

For contracts with several schemas, `--schema-name <name>` exports one at a time where the exporter supports it; `sql-query` requires it. Pass `--server <name>` to SQL and dbt exports so data types map to that server's dialect.

Read the warnings on stderr. `Cannot map type ... for field ...` means the field was exported with the declared `physicalType` or a fallback: check those fields in the output.

## Step 3: dbt

For a provider's dbt project, prefer `dbt sync`. It writes the contract's columns, descriptions, data types, tags, and tests into the existing model YAML files and adds singular SQL tests for constraints dbt cannot express natively. Everything it generates is marked, so later syncs update it without touching hand-written content.

```bash
datacontract dbt sync contracts/orders.odcs.yaml --project-dir ./warehouse --dry-run   # preview
datacontract dbt sync contracts/orders.odcs.yaml --project-dir ./warehouse             # apply
datacontract dbt test contracts/orders.odcs.yaml --project-dir ./warehouse             # run the generated tests
```

- Without a contract argument, `dbt sync` picks up every `*.odcs.yaml` in the project.
- `--model-resolution physicalName` maps schemas to dbt models by `physicalName` instead of `name`.
- `--prune` removes columns and tags that are not in the contract. It deletes content: show the dry run first and ask.
- `dbt test` runs dbt against the warehouse; ask before running it.

See [Sync with dbt](https://cli.datacontract.com/dbt).

## Step 4: Integrate the output

Generated code is a starting point that must fit the project:

- Read the generated file and compare it with existing code in the project (naming, folder layout, formatting). Move it to the right place and adapt imports.
- Keep a short header comment saying the file is generated from which contract and how to regenerate it, if the project does not already have such a convention.
- Never hand-edit the parts that the contract owns. If the generated output is wrong, fix the contract (and lint) and export again, so contract and code do not drift.
- If something cannot be expressed by the exporter (for example a check that has no dbt equivalent), say so instead of silently dropping it.

## Step 5: Keep it in sync

Offer to add the export to the project's build or CI so a contract change regenerates the files, for example a `make generate` target or a CI step that fails when `git diff --exit-code` shows the generated files are stale. See `contract-ci`.

Report the files written, the exporter options used, and anything from the contract that the target format could not represent.
