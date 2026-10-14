---
name: contract-ci
description: "Data Product Builders and Platform Engineers. Sets up continuous data contract checks with the Data Contract CLI in GitHub Actions, GitLab CI, Azure DevOps, Airflow, or plain cron: lint and breaking-change checks on every pull request, data tests after every load and on a schedule, credentials from the CI secret store, and optional publishing of results to Entropy Data. Use when the user wants to run data contract tests in CI/CD, on a schedule, in a pipeline or orchestrator, block breaking contract changes in pull requests, or monitor data quality continuously."
---

# Data Contracts in CI/CD

**Persona:** data engineer, platform engineer
**Trigger:** once a contract is in the repository and tests pass locally
**Output:** a pipeline definition in the user's repository, committed only on request

Two different jobs, often in the same workflow:

1. **On every pull request** (no warehouse access needed): lint every changed contract, check for breaking changes against the main branch, and plan the tests with `--dry-run`.
2. **After each load and on a schedule** (needs credentials): run the data tests against production to catch data drift between code changes.

## Step 0: Inventory

Find the contracts (`**/*.odcs.yaml`, `**/datacontract.yaml`), their server types (for the right install extra), and the CI system in use (`.github/workflows/`, `.gitlab-ci.yml`, `azure-pipelines.yml`, Airflow DAGs, ...). Ask which servers to test on a schedule and how often (daily is a good default). Make sure tests pass locally first (`test-data-contract`).

## Step 1: Pull request checks

GitHub Actions example. Adapt the paths and the extra to the project:

```yaml
# .github/workflows/datacontract-pr.yml
name: Data contract checks
on:
  pull_request:
    paths: ["contracts/**"]
jobs:
  contracts:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: astral-sh/setup-uv@v6
      - run: uv tool install --python 3.11 'datacontract-cli[snowflake]'
      - name: Lint
        run: |
          for f in contracts/*.odcs.yaml; do datacontract lint "$f" --all-errors; done
      - name: Breaking changes
        run: |
          status=0
          for f in $(git diff --name-only --diff-filter=M origin/${{ github.base_ref }}...HEAD -- 'contracts/*.odcs.yaml'); do
            git show "origin/${{ github.base_ref }}:$f" > /tmp/before.yaml
            datacontract breaking /tmp/before.yaml "$f" || status=1
          done
          exit $status
      - name: Plan tests
        run: |
          for f in contracts/*.odcs.yaml; do datacontract test "$f" --dry-run; done
```

A breaking change fails the check on purpose. If the team wants to allow it with a major version bump, add a step that only fails when `breaking` fails **and** the major version did not change. Semantic changes (Step 3 of `breaking-change-check`) still need a human reviewer.

## Step 2: Data tests on a schedule

```yaml
# .github/workflows/datacontract-test.yml
name: Data contract tests
on:
  push:
    branches: [main]
  schedule:
    - cron: "0 6 * * *"   # daily, after the nightly load
  workflow_dispatch:
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v6
      - run: uv tool install --python 3.11 'datacontract-cli[snowflake]'
      - run: datacontract ci contracts/*.odcs.yaml --server production
        env:
          DATACONTRACT_SNOWFLAKE_USERNAME: ${{ secrets.DATACONTRACT_SNOWFLAKE_USERNAME }}
          DATACONTRACT_SNOWFLAKE_PASSWORD: ${{ secrets.DATACONTRACT_SNOWFLAKE_PASSWORD }}
          DATACONTRACT_SNOWFLAKE_WAREHOUSE: ${{ secrets.DATACONTRACT_SNOWFLAKE_WAREHOUSE }}
          DATACONTRACT_SNOWFLAKE_ROLE: ${{ secrets.DATACONTRACT_SNOWFLAKE_ROLE }}
```

`datacontract ci` runs the same checks as `test` on several contracts at once and adds GitHub Actions annotations, a job summary, and `--fail-on error|warning|never` to control the exit code. It has no options to narrow a run; use `datacontract test` with `--tag`, `--quality-id`, `--filter`, or `--schema-name` for that (for example cheap checks after every load, expensive ones nightly).

The ready-made [datacontract-action](https://github.com/datacontract/datacontract-action) runs `datacontract test` for one contract and writes a JUnit report:

```yaml
      - uses: datacontract/datacontract-action@main
        with:
          location: contracts/orders.odcs.yaml
          server: production
        env:
          DATACONTRACT_SNOWFLAKE_USERNAME: ${{ secrets.DATACONTRACT_SNOWFLAKE_USERNAME }}
```

Other systems: see [Scheduling](https://docs.datacontract.com/scheduling) for Azure DevOps, Airflow (Data Contract provider), plain cron with the `datacontract/cli` Docker image, and calling the Python library from Dagster, Prefect, or Databricks jobs. For GitLab CI, use the same commands in a job with `image: python:3.11` and `rules:` for merge requests and schedules.

## Step 3: Credentials

- Credentials come from the CI secret store as `DATACONTRACT_<SOURCE>_<OPTION>` environment variables. List the secrets the user must create; never write values into the workflow file, and never ask for them in the chat.
- Use a dedicated read-only technical user for the tests. Tests read data; they need no write access.
- Pull request jobs from forks must not get production secrets. Keep data tests on `push`, `schedule`, and `workflow_dispatch` only.

## Step 4: Publish results (optional)

To track results over time in [Entropy Data](https://entropy-data.com):

```yaml
      - run: datacontract ci contracts/*.odcs.yaml --server production --publish https://api.entropy-data.com/api/test-results
        env:
          ENTROPY_DATA_API_KEY: ${{ secrets.ENTROPY_DATA_API_KEY }}
```

For a self-hosted Entropy Data, also set `ENTROPY_DATA_HOST`. Add publishing only when the user asks for it.

## Step 5: Hand over

Write the workflow files, show them, and list the secrets to create. Do not commit, push, or create secrets unless the user asks. Suggest a first manual run (`workflow_dispatch`) to confirm the setup before relying on the schedule.
