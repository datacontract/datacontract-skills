---
name: quality-rules
description: "Data Product Builders and Owners. Proposes and adds data quality rules and service levels to an ODCS data contract: schema constraints, library metrics (rowCount, nullValues, missingValues, invalidValues, duplicateValues), SQL rules, freshness and retention, each with a dimension, id, and tags, grounded in the field semantics and, if the user agrees, in a profile of the actual data, then lints and test-runs them. Use when the user wants to add quality checks, data quality expectations, SLAs, freshness checks, or make a contract stricter or more useful for consumers."
---

# Add Quality Rules and Service Levels

**Persona:** data engineer, data product owner
**Trigger:** after a contract is created, before it is marked `active`, or when consumers report a data problem a check should have caught
**Output:** quality rules and service levels in the contract, linted and test-run, with a coverage summary per quality dimension

## Step 0: CLI

Follow the `datacontract-cli` skill, Step 1.

## Step 1: Read the contract

Read the contract and `datacontract lint` it first. List per schema: grain, keys, required fields, closed value sets, amounts and their units, timestamps, relationships, and existing `quality` and `slaProperties`. Ask the owner what consumers rely on most and what has gone wrong before: those are the rules worth having.

## Step 2: Prefer schema constraints

Many expectations are not quality rules but schema attributes, which the CLI checks automatically and every exporter (dbt, SQL, JSON Schema, Pydantic, ...) understands:

| Expectation | Write it as |
|---|---|
| no nulls | `required: true` |
| no duplicates | `unique: true` |
| identifier | `primaryKey: true` (composite: on each member, with `primaryKeyPosition`) |
| reference to another table | `relationships: [{type: foreignKey, to: customers.customer_id}]` |
| closed set of values | `enum: [{value: a}, {value: b}]` |
| length, range, format | `logicalTypeOptions`: `minLength`, `maxLength`, `minimum`, `maximum`, `pattern`, `format` |

Use `quality` rules for what these cannot express.

## Step 3: Propose quality rules

For each schema, consider at least:

| Dimension | Typical rule |
|---|---|
| completeness | `rowCount` above a floor; `missingValues` with `['', 'N/A']` on text fields; `nullValues` with `unit: percent` where a few nulls are acceptable |
| uniqueness | `duplicateValues` over the natural key (schema level, `arguments.properties: [...]`) |
| conformity | `invalidValues` with `validValues` on open-ended but known code lists |
| consistency | `sql`: header total equals sum of line items; end date after start date |
| accuracy | `sql`: share of amounts in a plausible band (`mustBeBetween`) |
| timeliness | service level `freshness` (Step 4) |

Rule shapes:

```yaml
schema:
  - name: orders
    quality:
      - id: orders_not_empty
        type: library
        metric: rowCount
        mustBeGreaterThan: 1000
        dimension: completeness
        tags: [critical, cheap]
        description: A normal day has more than 1,000 orders; fewer means a partial load.
      - id: orders_total_matches_items
        type: sql
        dimension: consistency
        description: The order total equals the sum of its line items.
        query: |
          SELECT COUNT(*) FROM ${table} o
          JOIN (SELECT order_id, SUM(price) AS s FROM line_items GROUP BY order_id) li
            ON li.order_id = o.order_id
          WHERE o.order_total <> li.s
        mustBe: 0
    properties:
      - name: customer_email
        quality:
          - id: customer_email_mostly_present
            type: library
            metric: nullValues
            unit: percent
            mustBeLessThan: 1
            dimension: completeness
```

Rules:

- Every rule gets a `description` that says **why** the threshold is what it is, a `dimension` (one of `accuracy`, `completeness`, `conformity`, `consistency`, `coverage`, `timeliness`, `uniqueness`), and an `id` unique in the contract. Add `tags` (`critical`, `cheap`, `expensive`) so pipelines can run subsets with `--tag`.
- Library metrics: `rowCount` is schema level; `nullValues`, `missingValues`, `invalidValues` are property level; `duplicateValues` is either. Exactly one comparator per rule: `mustBe`, `mustNotBe`, `mustBeGreaterThan`, `mustBeGreaterOrEqualTo`, `mustBeLessThan`, `mustBeLessOrEqualTo`, `mustBeBetween`, `mustNotBeBetween`.
- SQL rules return a single number, are read-only, use the server's dialect, and should use placeholders (`${table}`, `${schema}`, `${column}`) rather than hard-coded names.
- `type: text` documents an expectation nobody can automate. Use it sparingly.
- Thresholds come from the owner or from data, never from thin air. If neither is available, propose a value and mark it as an assumption in the description.

Check the details of a rule type before using an option you have not seen: [quality rules](https://docs.datacontract.com/quality-rules), [library](https://docs.datacontract.com/quality-rules/library), [SQL](https://docs.datacontract.com/quality-rules/sql).

## Step 4: Service levels

Add `slaProperties` at the contract root. `freshness` and `retention` are checked by the CLI when they name the timestamp column in `element`:

```yaml
slaProperties:
  - property: freshness
    value: "24"
    unit: hours
    element: orders.order_timestamp
    description: New orders arrive within 24 hours.
  - property: retention
    value: "2"
    unit: years
    element: orders.order_timestamp
  - property: availability
    value: 99.9%
```

Other properties (`availability`, `latency`, `support`, ...) document promises and are not tested. See [service levels](https://docs.datacontract.com/service-levels).

## Step 5: Ground the thresholds in data (optional)

Only with the user's go-ahead to query the server. Run each candidate rule once and look at the actual value:

```bash
datacontract test contracts/orders.odcs.yaml --server production --checks quality,slaProperties \
  --output "$SCRATCH/quality.json" --output-format json
```

A rule that fails on today's data is either a real problem (tell the owner) or a threshold that is too strict (adjust it with a note). Never silently loosen a threshold until it passes.

## Step 6: Apply and report

Show the additions as a diff, apply on confirmation, `datacontract lint --all-errors`, and test-run with `--dry-run` (no credentials needed) to confirm that every rule can be planned.

Report a coverage table per schema, one row per dimension: rules before, rules after, and which important field has no check yet. Point to `contract-ci` to run the rules on a schedule, for example cheap rules on every load (`--tag cheap`) and expensive ones nightly.
