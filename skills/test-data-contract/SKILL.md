---
name: test-data-contract
description: "Data Product Builders and Owners. Runs `datacontract test` against the data source in a contract's servers block (Snowflake, Databricks, BigQuery, Postgres, S3, Kafka, local files, and more), sets up the connection and credentials, and when checks fail, investigates each failure and says whether the data, the contract, or the connection is wrong, with a concrete fix. Use when the user wants to test, validate, or check data against a data contract, asks why a contract test fails, gets a schema or type mismatch, or needs to connect the CLI to a data source."
---

# Test a Data Contract

**Persona:** data engineer, data product owner
**Trigger:** after creating or changing a contract or the pipeline behind it, or when a test fails
**Output:** a test run, and for every failed check: the cause (data, contract, or setup) and a fix

`datacontract test` connects to a server from the contract and runs three kinds of checks: **properties** (presence, types, required, unique, keys, constraints), **quality** rules, and **service levels** (freshness, retention).

## Step 0: CLI

Follow the `datacontract-cli` skill, Step 1. The server type needs its extra (for example `datacontract-cli[snowflake]`); `[all]` covers every source.

## Step 1: Find the contract and the server

Locate the contract (`*.odcs.yaml`, `datacontract.yaml`) or use the path or URL the user gives. Read its `servers` block and pick the server to test. If there are several, ask, and default to a non-production one for a first run.

Without a `servers` block there is nothing to connect to: add one (see `create-data-contract`, Step 3), or for a local try-out test a file:

```yaml
servers:
  - server: local
    type: local
    path: ./data/orders.parquet    # or .csv, .json, delta directory
    format: parquet
```

## Step 2: Credentials

Look up the variables for the server type on its page under [Test your Data](https://cli.datacontract.com/testing) (for example [Snowflake](https://cli.datacontract.com/testing/snowflake), [Databricks](https://cli.datacontract.com/testing/databricks), [BigQuery](https://cli.datacontract.com/testing/bigquery), [Postgres](https://cli.datacontract.com/testing/postgres), [S3](https://cli.datacontract.com/testing/s3)). Check which ones are set, without printing values:

```bash
for v in DATACONTRACT_SNOWFLAKE_USERNAME DATACONTRACT_SNOWFLAKE_PASSWORD DATACONTRACT_SNOWFLAKE_WAREHOUSE DATACONTRACT_SNOWFLAKE_ROLE; do
  if [ -n "${!v}" ] || grep -q "^$v=" .env 2>/dev/null; then echo "$v set"; else echo "$v MISSING"; fi
done
```

Tell the user which variables are missing and let them set them (shell, `.env`, or secret store). Never ask for the values in the chat (see the `datacontract-cli` skill, Step 2).

## Step 3: Plan, then run

Show what will run without touching the data:

```bash
datacontract test contracts/orders.odcs.yaml --server production --dry-run
```

Then, with the user's go-ahead to connect:

```bash
datacontract test contracts/orders.odcs.yaml --server production \
  --output "$SCRATCH/results.json" --output-format json
```

Use `$SCRATCH` as a temporary directory outside the repository. Options to keep runs fast and cheap:

| Need | Option |
|---|---|
| Only one table | `--schema-name orders` |
| Only schema and types, no row scans | `--metadata-only` |
| Only recent rows on a big table | `--filter "ingested_at >= CURRENT_DATE - 1"` (server's SQL dialect; with several schemas use `--filters '{"orders": "..."}'`) |
| Only some categories | `--checks properties,quality,slaProperties` |
| One quality aspect | `--dimension completeness` |
| Specific quality rules | `--quality-id <id>` or `--tag critical` |
| Example rows for failed checks | `--include-failed-samples` (ask first; mask personal data in what you show) |

## Step 4: Explain every failure

Read the results file and group the non-passed checks (`failed`, `warning`, `error`) by schema and field. For each, decide which side is wrong:

| Symptom | Usually | Check |
|---|---|---|
| Field is not present | renamed or dropped column, or `physicalName` missing | Compare with the table: `datacontract import <source> ... ` into `$SCRATCH` and diff the property names |
| Type mismatch | `physicalType` does not match the catalog type exactly (`VARCHAR` vs `VARCHAR(16777216)`, `NUMBER(38,0)` vs `INTEGER`, `TIMESTAMP_NTZ` vs `TIMESTAMP`) | Fresh import shows the real type. Fix the contract unless the pipeline is supposed to produce the declared type |
| Missing values / duplicates | real data problem, or the key or `required` is wrong | Run a count query on the server only with the user's go-ahead; look at the pipeline's recent changes |
| Invalid values / enum | a new value appeared upstream | Was it added on purpose? Then extend the contract (non-breaking for most consumers) |
| Row count | empty or partial load | Load date and pipeline run; tighten `--filter` |
| Freshness | late pipeline, or `element` points to the wrong column, or time zone | Max of the `element` column vs. now |
| Connection or `error` result | credentials, network, missing extra, wrong `servers` fields | Run again with `--logs --debug`; check the source page's required fields |
| Warning "cannot connect" for a server type | type not testable by the CLI | See the supported list under [Test your Data](https://cli.datacontract.com/testing) |

Rules:

- A failure is a finding about the **data** only when the contract is right. Do not "fix" a real data problem by loosening the contract without the owner's decision: that changes what consumers rely on (see `breaking-change-check`).
- When the contract is wrong (typo, wrong `physicalType`, a key that never was one), propose the edit as a diff, apply it on confirmation, `datacontract lint`, and test again.
- When the data is wrong, name the evidence (check, counts, filter used) and suggest where to look in the pipeline.

## Step 5: Report

```
Contract: contracts/orders.odcs.yaml   Server: production (snowflake)   Filter: none
Result: 🔴 3 failed, 1 warning, 41 passed

1. orders.order_total: 1,204 missing values (required)          -> data: ...
2. orders.created_at: type TIMESTAMP_NTZ, contract TIMESTAMP_TZ  -> contract: ...
3. ...
Next: ...
```

Mention any filter or `--metadata-only` used: a filtered run only makes a statement about the rows it tested. Delete `$SCRATCH` at the end. To keep tests running automatically, offer the `contract-ci` skill.
