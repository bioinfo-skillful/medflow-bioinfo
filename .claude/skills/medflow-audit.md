---
name: "MedFlow Audit"
description: Audit a compiled workflow before execution — verify config keys, data flow, group columns, sample IDs, node versions, and conda packages
category: Workflow
tags: [workflow, audit, medflow]
---

Audit a compiled workflow.json before execution. Run after compile, before dispatch.
If any check fails, stop — do not proceed to run.

## Checks

### 1. Config Key Audit

For every config key on every step:
- Transform: underscore → hyphen → add `--` prefix
- Match against the target node's `parameters[].name` (from SKILL.md)
- **If no match: CRITICAL.** Report "Config key `<key>` on step `<id>` does not match any parameter on node `<node>`. Available parameters: `<list>`"

### 2. Upstream Data Flow Audit

For every `bind: upstream` parameter on every step:
- Identify which upstream step(s) produce files
- Read upstream node's `outputs` in SKILL.md
- Verify at least one upstream output's `semantic_type` matches the parameter's expected type
- **If no match: CRITICAL.** Report "Parameter `--<name>` on step `<id>` requires upstream file of type `<type>`, but upstream step `<upstream>` produces: `<list>`"

### 3. Group Column Coverage Audit

If the workflow involves group comparison (deg step with `--map` parameter):
- Open the metadata CSV from every upstream dataset
- List all column names (direct columns AND `characteristics_ch1` keys)
- Verify the group column exists in every dataset
- **If missing from any dataset: CRITICAL.** Report which dataset lacks the column, list available columns that may be equivalent
- Count non-NA P/N values per dataset. Flag if any dataset has <10 samples per group.

### 4. Sample ID Cross-Reference Audit

If `sample_group_map.csv` (or equivalent) exists:
- Read the group file: verify column 1 contains sample IDs, column 2 contains group labels
- Read the expression matrix that feeds the deg step
- Cross-reference: sample IDs in the group file must match column headers in the expression matrix
- **If no overlap: CRITICAL.** Report "Sample IDs in group file do not match expression matrix columns. Group file uses `<id_type>` (example: `<example>`), expression matrix uses `<other_type>` (example: `<other_example>`)."
- If partial overlap: warn what fraction matched.

### 5. Node Version Audit

For every node directory in `nodes/`:
- Run `git log --oneline -1` to get the current HEAD
- Run `git fetch origin && git log origin/main --oneline -1` to get the latest remote
- If local HEAD != remote HEAD: **WARNING.** Report "Node `<name>` is at `<local>` but latest is `<remote>`. Clone fresh."

### 6. Conda Package Audit

For every node's environment file (`env.yaml` or `envs/env-r-*.yaml`):
- Parse all listed conda/pipped packages
- Check each package exists: `conda search <pkg>` (conda) or `pip index versions <pkg>` (pip)
- **If any package unavailable: CRITICAL.** Report "Package `<pkg>` in node `<name>` env file is not available from declared channels."

## Output

On completion, output a single NDJSON result line:

```json
{"level":"result","status":"pass","checks":6,"warnings":0,"critical":0}
```

Or on failure:

```json
{"level":"result","status":"fail","checks":6,"warnings":0,"critical":[{"check":1,"step":"fetch1","key":"rename","msg":"..."}]}
```

The agent MUST NOT proceed to execution if `status` is `fail`.
