# datacontract-skills: Copilot CLI instructions

> This file is loaded automatically when the plugin is installed with Copilot CLI. Do not copy it into your project.

This repository is a coding-agent plugin for working with [data contracts](https://datacontract.com) in the Open Data Contract Standard (ODCS), using the [Data Contract CLI](https://github.com/datacontract/datacontract-cli) and the [Data Contract Editor](https://github.com/datacontract/datacontract-editor). Each capability is a **skill**: a markdown file under `skills/<name>/SKILL.md` that you read top to bottom and execute step by step.

When a user request matches a skill, **read the corresponding `SKILL.md` completely before acting.** Each skill's frontmatter `description` is authoritative for when to use it.

## Skills

| When the user asks about… | Follow this skill |
|---|---|
| Creating, writing, importing, or generating a data contract; fixing lint errors | `skills/create-data-contract/SKILL.md` |
| Testing data against a contract, failing contract tests, connecting to a data source | `skills/test-data-contract/SKILL.md` |
| Adding quality rules, quality checks, SLAs, freshness | `skills/quality-rules/SKILL.md` |
| Whether a contract change is breaking, version bumps, comparing contract versions, reviewing a contract PR | `skills/breaking-change-check/SKILL.md` |
| Generating SQL, dbt, Avro, JSON Schema, Pydantic, HTML, or other code from a contract | `skills/export-data-contract/SKILL.md` |
| Running contract checks in CI/CD or on a schedule | `skills/contract-ci/SKILL.md` |
| Opening, viewing, sharing, or customizing the Data Contract Editor | `skills/datacontract-editor/SKILL.md` |
| Installing the `datacontract` CLI, extras, credentials | `skills/datacontract-cli/SKILL.md` |

Every skill checks the CLI first (`datacontract --version`, see `skills/datacontract-cli/SKILL.md`).

## Conventions when running skills

- **Lint after every edit** of a contract (`datacontract lint <file>`), and fix all errors.
- **Ask before connecting to data** (`test`, `ci`, live `import`) the first time in a session, naming the server. Prefer `--dry-run` or `--metadata-only` when that is enough.
- **Ask before anything outward-facing:** `publish`, `--publish`, pushing, opening pull requests, commenting, or sharing links.
- **Never ask for secrets in the chat** and never print them. Credentials are `DATACONTRACT_*` environment variables or a `.env` file the user maintains.
- **Never output unmasked personal data** from test results or failed samples.
- **Do not loosen a contract to make a failing test pass** without the owner's decision.
- **Look it up instead of guessing:** `datacontract <command> --help` and [docs.datacontract.com](https://docs.datacontract.com) ([llms.txt](https://docs.datacontract.com/llms.txt)).
