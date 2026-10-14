---
name: breaking-change-check
description: "Data Product Owners and Builders. Compares two versions of an ODCS data contract (a working copy against git HEAD, a branch, a pull request, or two files or URLs) with `datacontract changelog` and `datacontract breaking`, adds the semantic changes no tool detects, and returns a verdict (breaking, needs review, safe), the version bump, a changelog entry, and a migration note for consumers. Use when the user changes a data contract and asks whether the change is breaking or backward compatible, which version to bump, what changed between two contract versions, or wants to review a contract pull request."
---

# Breaking Change Check

**Persona:** data product owner, data engineer, reviewer of a contract pull request
**Trigger:** before merging a change to a data contract
**Output:** a verdict, the classified changes, the version bump, a changelog entry, and a note for consumers

Works locally on files with the Data Contract CLI. If the contract is managed in [Entropy Data](https://entropy-data.com) and the user wants to know which consumers are affected, the `contract-change-review` skill of the [Entropy Data skills](https://github.com/entropy-data/entropy-data-skills) adds the consumer and lineage impact.

## Step 0: CLI

Follow the `datacontract-cli` skill, Step 1.

## Step 1: Get both versions

Work in a scratch directory `$SCRATCH` outside the repository. Get the **before** (`v1`) and **after** (`v2`) version:

| Situation | v1 | v2 |
|---|---|---|
| Uncommitted edit | `git show HEAD:<path> > "$SCRATCH/v1.yaml"` | the working copy |
| Branch or PR against main | `git show origin/main:<path> > "$SCRATCH/v1.yaml"` (after `git fetch`) | `git show <branch>:<path>` or the working copy; for a PR, `gh pr diff <n>` lists the contract files |
| Two files or URLs | the older | the newer |

If the PR touches several contracts, check each. A contract that is new in the PR has no v1: lint it instead and report it as added.

Lint v2 first (`datacontract lint`): an invalid contract is a blocker regardless of compatibility.

## Step 2: Let the CLI classify

```bash
datacontract changelog "$SCRATCH/v1.yaml" path/to/contract.odcs.yaml
datacontract breaking  "$SCRATCH/v1.yaml" path/to/contract.odcs.yaml
```

`breaking` adds a severity to every change and exits with `1` if any change is breaking:

- **ERROR**: backward-incompatible, existing consumers or tests can fail (a removed property, a narrowed type, a new `required`, a removed enum value, ...).
- **WARNING**: potentially incompatible, needs a human decision.
- **INFO**: metadata, relaxed constraints, additions.

The detail table is long because every nested attribute is listed. Summarize by the top-level element (the property, schema, server, or service level) and keep the highest severity per element.

## Step 3: Add what the tool cannot see

Read the diff yourself and check for changes that are breaking in meaning but not in structure:

- **Semantics changed under the same name:** a different unit (euros to cents), currency, time zone, grain (one row per order to one row per line item), or definition (revenue now includes refunds). These are breaking. A changed `description` is often the only hint; ask the owner when in doubt.
- **Renames** show up as a removal plus an addition: breaking for consumers, even if the data is the same.
- **`physicalName`, server `schema`, `database`, `location`, or topic changes:** consumers query a different place.
- **Tighter quality thresholds or service levels:** not breaking for consumers, but they can break the provider's own pipeline. Run `datacontract test --dry-run` on v2 to see the planned checks.
- **Looser guarantees** (removed `required`, lowered freshness, removed quality rule): structurally relaxed, but consumers may rely on them. Classify as needs review.
- **`status` to `deprecated` or `retired`:** breaking announcement; needs a sunset date and a successor.

## Step 4: Verdict and version

| Verdict | When | Version bump |
|---|---|---|
| 🔴 Breaking | any ERROR, or a semantic break from Step 3 | major (`1.4.2` to `2.0.0`) |
| 🟡 Needs review | WARNINGs or relaxed guarantees, no ERROR | minor, after the owner decides |
| 🟢 Safe | only additions and documentation | minor for additions, patch for documentation only |

Check that v2's `version` is bumped accordingly; if not, propose the bump. For a breaking change, propose the usual ways to avoid breaking consumers: add the new field next to the old one and deprecate the old one first, or publish the new version as a separate contract (`orders-v2`) and run both in parallel for a transition period.

## Step 5: Report

```
Contract: contracts/orders.odcs.yaml   1.4.2 -> 1.5.0 (proposed: 2.0.0)
Verdict: 🔴 Breaking

Breaking
- orders.order_status removed                             (CLI: ERROR)
- orders.order_total now in cents instead of euros        (semantic, from description)
Needs review
- orders.customer_email no longer required
Safe
- orders.channel added (optional)

Changelog entry: ...
Note to consumers: what changes, from which version or date, what to do.
```

Keep the consumer note short and concrete. Do not post comments on the pull request or notify anyone without the user's confirmation; offer the text as a draft. Delete `$SCRATCH`. To run this check on every pull request, see `contract-ci`.
