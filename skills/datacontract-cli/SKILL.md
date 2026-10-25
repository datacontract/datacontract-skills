---
name: datacontract-cli
description: "Core. Install, upgrade, and configure the Data Contract CLI (`datacontract`), the open-source tool every other data contract skill builds on, and look up its documentation. Use when the user wants to install the CLI, pick the right extras for a data source, set up credentials for a server, check the CLI version, or when another skill's Step 0 says to verify the CLI."
---

# Data Contract CLI

**Persona:** anyone working with data contracts
**Output:** a working `datacontract` installation with credentials for the data sources the user needs

The [Data Contract CLI](https://github.com/datacontract/datacontract-cli) is an open-source command-line tool for data contracts in the [Open Data Contract Standard (ODCS)](https://bitol-io.github.io/open-data-contract-standard/latest/). It lints contracts, tests data against them on 20+ data sources, imports contracts from existing schemas, exports them to 25+ formats, compares versions, and opens the visual Data Contract Editor.

## Step 1: Check the installation

```bash
datacontract --version
```

If the command is missing or older than the version on [PyPI](https://pypi.org/project/datacontract-cli/), install or upgrade it. Prefer [uv](https://docs.astral.sh/uv/):

```bash
uv tool install --python python3.11 --upgrade 'datacontract-cli[all]'
```

Alternatives:

- `pip install 'datacontract-cli[all]'` in a virtual environment (Python 3.10 or newer)
- `uvx --from 'datacontract-cli[all]' datacontract <command>` for a one-off run without installing
- Docker: `docker run --rm -v "${PWD}:/home/datacontract" datacontract/cli <command>`

`[all]` pulls in every data source driver. When the user only needs one source, a smaller extra is faster to install: `[snowflake]`, `[bigquery]`, `[databricks]`, `[postgres]`, `[s3]`, `[kafka]`, `[sqlserver]`, `[trino]`, `[api]` (needed for `datacontract edit` and `datacontract api`), and more, see [Installation](https://cli.datacontract.com/installation).

Ask before installing anything. Never install with `sudo`.

## Step 2: Credentials

Connection details (host, database, schema, location) live in the `servers` block of the data contract. **Credentials never go into the contract.** They are environment variables named `DATACONTRACT_<SOURCE>_<OPTION>`, for example:

```bash
DATACONTRACT_POSTGRES_USERNAME
DATACONTRACT_POSTGRES_PASSWORD
DATACONTRACT_SNOWFLAKE_USERNAME
DATACONTRACT_SNOWFLAKE_PASSWORD
DATACONTRACT_SNOWFLAKE_WAREHOUSE
DATACONTRACT_SNOWFLAKE_ROLE
DATACONTRACT_DATABRICKS_TOKEN
DATACONTRACT_BIGQUERY_ACCOUNT_INFO_JSON_PATH
DATACONTRACT_S3_ACCESS_KEY_ID
DATACONTRACT_S3_SECRET_ACCESS_KEY
```

The exact variables for each source are on its page under [Test your Data](https://cli.datacontract.com/testing). Where they can come from, in order of precedence:

1. already set environment variables,
2. a `.env` file in the working directory or the nearest parent,
3. a YAML config file (`--config-file`, or `./datacontract-config.yaml`, or `~/.datacontract/config.yaml`), see [Configuration](https://cli.datacontract.com/configuration).

Rules for the agent:

- **Never ask the user to paste a password, token, or key into the chat.** Tell them which variables to set and where (a `.env` file, their shell, a secret store), and let them set the values themselves.
- Never print the content of `.env` or a config file with secrets. To check that a variable is set, test for presence only: `test -n "$DATACONTRACT_POSTGRES_PASSWORD" && echo set`.
- Make sure `.env` is listed in `.gitignore` before the user writes secrets to it.
- In contracts, server fields may reference variables as `${VAR}` or `${VAR:-default}`, for example to switch the schema per environment.

## Step 3: Look up what you are not sure about

The CLI evolves quickly. Before using an option you have not seen in `--help`, check it:

```bash
datacontract <command> --help
datacontract export <format> --help
datacontract import <format> --help
```

The full documentation is at [cli.datacontract.com](https://cli.datacontract.com), also as [llms.txt](https://cli.datacontract.com/llms.txt) (an index) and [llms-full.txt](https://cli.datacontract.com/llms-full.txt) (every page in one file). Fetch the specific page you need instead of guessing option names or YAML attributes.

## Command overview

| Command | What it does | Skill with details |
|---|---|---|
| `init` | Create a new contract from a template | `create-data-contract` |
| `import <format>` | Create a contract from SQL DDL, dbt, Avro, JSON Schema, Excel, or a live source (Snowflake, Databricks, BigQuery, Postgres, ...) | `create-data-contract` |
| `lint` | Validate the contract against the ODCS JSON Schema and the CLI's rules | `create-data-contract` |
| `edit` | Open the contract in the Data Contract Editor (web UI) | `datacontract-editor` |
| `test` | Run schema, quality, and service level checks against a server | `test-data-contract` |
| `ci` | `test` for pipelines: several contracts, annotations, job summary, exit-code control | `contract-ci` |
| `changelog` / `breaking` | Compare two contract versions, classify breaking changes | `breaking-change-check` |
| `export <format>` | Generate SQL DDL, dbt, Avro, JSON Schema, Pydantic, HTML, ... | `export-data-contract` |
| `dbt sync` / `dbt test` | Generate dbt model properties and tests from contracts | `export-data-contract` |
| `catalog` | Build a static HTML catalog of many contracts | `export-data-contract` |
| `publish` | Publish a contract to [Entropy Data](https://entropy-data.com) | |
| `api` | Run the CLI as a REST API server | |

## Conventions for all data contract skills

- **Files, not chat.** Write contracts to `*.odcs.yaml` files in the user's repository (ask for the path if there is no obvious convention), and show diffs instead of whole files.
- **Lint after every edit.** Run `datacontract lint <file>` after each change and fix every error before moving on. Use `--all-errors` to see everything at once.
- **Ask before connecting to data.** `test`, `import` from a live source, and `ci` read from the user's data platform. Say which server and which credentials will be used, and get a go-ahead the first time in a session. Prefer `--dry-run` or `--metadata-only` when the user only wants to see what would run.
- **Ask before anything outward-facing:** `publish`, `--publish <url>`, pushing to a remote, opening a pull request.
- **No personal data in output.** When test results or failed samples (`--include-failed-samples`) contain values that look personal (names, emails, phone numbers, IDs), mask them in what you show.
