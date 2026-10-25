# Data Contract Skills

Skills that teach your coding agent (Claude Code, Codex, GitHub Copilot CLI, or any agent that supports [Agent Skills](https://agentskills.io)) to work with [data contracts](https://datacontract.com) in the [Open Data Contract Standard (ODCS)](https://bitol-io.github.io/open-data-contract-standard/latest/), using the open-source [Data Contract CLI](https://github.com/datacontract/datacontract-cli) and [Data Contract Editor](https://github.com/datacontract/datacontract-editor).

Ask in plain language, for example "create a data contract for the orders table in Snowflake", and the agent imports the schema, writes descriptions, keys, and quality rules, lints the contract until it is valid, tests it against your data, and explains every failing check.

## Skills

| Skill | What you get |
|---|---|
| **[Create a Data Contract](skills/create-data-contract/SKILL.md)** | A valid ODCS contract, imported from a table, DDL, dbt, Avro, JSON Schema, or Excel (data-first), or written from requirements (contract-first), with descriptions, types, keys, servers, and owner. |
| **[Test a Data Contract](skills/test-data-contract/SKILL.md)** | A test run against Snowflake, Databricks, BigQuery, Postgres, S3, Kafka, and 15+ more sources, with the cause and a fix for every failing check: the data, the contract, or the connection. |
| **[Quality Rules](skills/quality-rules/SKILL.md)** | Quality rules and service levels for each quality dimension, with thresholds grounded in the data, linted and test-run. |
| **[Breaking Change Check](skills/breaking-change-check/SKILL.md)** | For a changed contract or a pull request: breaking, needs review, or safe, including semantic changes no tool detects, the version bump, and a note for consumers. |
| **[Export a Data Contract](skills/export-data-contract/SKILL.md)** | SQL DDL, dbt models and tests, Avro, JSON Schema, Protobuf, Pydantic, Spark, Great Expectations, HTML, and more, integrated into your project. |
| **[Data Contracts in CI/CD](skills/contract-ci/SKILL.md)** | Pipelines that lint and check for breaking changes on every pull request and test the data on a schedule, in GitHub Actions, GitLab, Azure DevOps, Airflow, or cron. |
| **[Data Contract Editor](skills/datacontract-editor/SKILL.md)** | The contract open in the visual editor while the agent edits the YAML, share links for stakeholders, and team deployments with your own fields. |
| **[Data Contract CLI](skills/datacontract-cli/SKILL.md)** | Installs the CLI with the right extras, sets up credentials safely, and defines the conventions the other skills follow. Used automatically. |

## Install

### Claude Code

```
claude plugin marketplace add https://github.com/datacontract/datacontract-skills
claude plugin install datacontract-skills@datacontract-skills
```

### OpenAI Codex

```
codex plugin marketplace add https://github.com/datacontract/datacontract-skills
codex plugin add datacontract-skills@datacontract-skills
```

### GitHub Copilot CLI

```
copilot plugin marketplace add https://github.com/datacontract/datacontract-skills
copilot plugin install datacontract-skills@datacontract-skills
```

### Any other agent

Clone the repository and point your agent at `skills/`, or reference [`AGENTS.md`](AGENTS.md) from your project's own `AGENTS.md`.

### Update

Claude Code (restart Claude Code afterwards):

```
claude plugin marketplace update datacontract-skills
claude plugin update datacontract-skills@datacontract-skills
```

Codex: `codex plugin marketplace upgrade datacontract-skills`, then `codex plugin add datacontract-skills@datacontract-skills`. Copilot CLI: `copilot plugin update datacontract-skills@datacontract-skills`.

## Requirements

The skills use the [Data Contract CLI](https://cli.datacontract.com). The agent installs it on request, or install it yourself:

```bash
uv tool install --python python3.11 --upgrade 'datacontract-cli[all]'
```

Credentials for your data sources are environment variables (or a `.env` file), see [Test your Data](https://cli.datacontract.com/testing). The skills never ask you to paste secrets into the chat.

## Usage

- "Create a data contract for the `orders` table in Snowflake."
- "Write a data contract for a customer churn dataset from these requirements: ..."
- "Test the orders contract against production and tell me why it fails."
- "Add quality rules and a freshness SLA to `contracts/orders.odcs.yaml`."
- "Is my change to the orders contract breaking? Which version should it get?"
- "Generate the dbt models and tests from the contract."
- "Run the data contract tests every morning in GitHub Actions."
- "Open the contract in the Data Contract Editor."

## Principles

- **Valid or not done.** Every change to a contract is linted with `datacontract lint`.
- **Ask before touching data or anything outward-facing.** Connecting to a data source, publishing, pushing, and notifying people need your go-ahead.
- **No secrets in chat, no personal data in output.** Credentials stay in environment variables; sample values are masked.
- **Don't loosen a contract to make a test pass.** A failing check is either a data problem for the owner or a contract error with evidence.

## Related

- [datacontract.com](https://datacontract.com): what data contracts are
- [Data Contract CLI documentation](https://cli.datacontract.com)
- [Data Contract Editor](https://editor.datacontract.com)
- [Entropy Data Skills](https://github.com/entropy-data/entropy-data-skills): analyses for data contracts and data products managed in [Entropy Data](https://entropy-data.com)

## License

[MIT](LICENSE)
