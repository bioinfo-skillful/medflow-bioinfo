---
name: "MedFlow Audit"
description: Audit and, when safe, remediate a compiled workflow by verifying configuration, data compatibility, identifiers, pinned node revisions, and runtime packages; produce labeled reproducible code for every repair
category: Workflow
tags: [workflow, audit, medflow]
---

Audit a compiled workflow after compilation and again after prerequisite fetch
steps when checks depend on runtime data.

The audit rejects demonstrated incompatibility, not naming differences or
dependencies with a documented, verified installation fallback.

The audit may edit a pinned node checkout or write adapter/helper code when a
finding can be repaired without silently changing scientific meaning. Every
repair must be labeled, reproducible from the pinned inputs and node commit,
tested, and re-audited before execution continues.

## Severity Model

- **CRITICAL**: proven incompatible data, a missing required parameter, or a
  package that cannot be installed or loaded.
- **WARNING**: a compatible semantic alias, a fallback installation was
  required, or a remote default branch is not named `main`.
- **DEFERRED**: a check requires upstream runtime output that does not exist
  yet. Deferred checks permit only prerequisite fetch steps; repeat the audit
  before merge, DEG, enrichment, or filtering.

## Remediation Authority

Diagnose and assign a stable finding ID before changing anything. The audit may
then perform one of these remediations:

- **INPUT_ADAPTER**: write deterministic code that creates a new run-local
  derived input, such as column renaming, exact label filtering, transposition,
  feature subsetting, or schema normalization.
- **NODE_PATCH**: edit the pinned node checkout to fix an implementation defect
  while preserving the declared scientific method and public contract.
- **HELPER_CODE**: write a missing deterministic conversion, validation, or
  orchestration helper required to connect declared-compatible artifacts.
- **ENVIRONMENT_FIX**: amend only the isolated runtime environment or install a
  documented dependency fallback.

The audit must not automatically make a change that alters the biological
contrast, statistical method, normalization, identifier semantics, imputation,
feature-selection meaning, model target, output schema, or public node
contract. Such a change requires explicit user confirmation and corresponding
protocol, registry, and node `SKILL.md` documentation updates.

Never overwrite or modify raw/source input files. Write derived inputs beneath:

```text
runs/<workflow>/audit-remediation/<finding-id>/outputs/
```

Do not push, commit, or publish a node patch unless the user separately asks.

## Affected Input Inventory

Before writing an `INPUT_ADAPTER`, report each affected file with:

- absolute run-local path and SHA-256;
- producing step or external-input provenance;
- observed format, delimiter, dimensions, identifier column, required columns,
  and a small non-sensitive schema sample;
- the exact incompatibility and downstream parameter it blocks;
- whether the proposed change is mechanical or changes scientific meaning;
- adapter code path, derived output path, expected schema, and acceptance
  checks.

Label the source file `immutable: true`. The adapter must read it and create a
new finding-ID-labeled output; it must never edit the file in place.

## Labeled Reproducibility Bundle

Store every repair beneath:

```text
workflows/remediations/<workflow>/<finding-id>/
├── remediation.yaml
├── code/
├── node.patch                 # NODE_PATCH only
├── run_command.ps1            # Windows, when applicable
├── run_command.sh             # POSIX, when applicable
├── environment.yaml           # or an equivalent lock/session record
├── checksums.sha256
├── tests/
└── logs/
```

`remediation.yaml` must label the repair with:

- `label: MEDFLOW_AUDIT_GENERATED_REMEDIATION`;
- finding ID, remediation type, UTC creation time, and rationale;
- workflow path and pre-remediation workflow SHA-256;
- node name, registry URL, declared version, default branch, and pinned base
  commit when a node is involved;
- raw input paths and SHA-256 checksums without copying or changing them;
- every generated, modified, and derived file;
- exact command, working directory, environment, random seeds, and expected
  outputs;
- whether scientific meaning or the public contract changed;
- tests run, exit codes, observed outputs, and post-remediation audit result.

Begin generated source files with a language-appropriate comment containing
`MEDFLOW_AUDIT_GENERATED_REMEDIATION`, the finding ID, purpose, and bundle path.
For formats that cannot contain comments, such as CSV, use a filename containing
the finding ID and record the label in a sidecar `remediation.yaml` entry.

For a node edit, preserve the registry commit as the immutable base, export the
complete change as `node.patch`, and record its SHA-256. A future run must clone
the pinned base commit and apply that exact patch; an unrecorded dirty checkout
is never an executable source.

Use this minimum manifest shape:

```yaml
schema: medflow-audit-remediation@1
label: MEDFLOW_AUDIT_GENERATED_REMEDIATION
finding_id: MF-AUD-001
type: INPUT_ADAPTER  # INPUT_ADAPTER | NODE_PATCH | HELPER_CODE | ENVIRONMENT_FIX
created_at_utc: "<ISO-8601>"
scientific_meaning_changed: false
public_contract_changed: false
workflow:
  path: workflows/<name>.json
  sha256: "<sha256>"
node:
  name: "<node-or-null>"
  url: "<registry-url-or-null>"
  version: "<version-or-null>"
  base_commit: "<sha-or-null>"
inputs:
  - path: "<immutable-source-path>"
    sha256: "<sha256>"
    immutable: true
artifacts:
  - path: code/<script>
    sha256: "<sha256>"
    label: MEDFLOW_AUDIT_GENERATED_REMEDIATION
reproduction:
  working_directory: "<sandbox-relative-path>"
  command: "<exact-command>"
  environment: environment.yaml
  seeds: {}
verification:
  tests: []
  smoke_test_exit_code: null
  reaudit_status: null
```

## Remediation Verification

After writing a repair:

1. Recreate or verify the repair from its recorded command in an isolated
   remediation directory.
2. Verify all input, code, patch, environment, and output checksums.
3. Run focused unit/validation tests and a minimal affected-step smoke test when
   safe test data are available.
4. Re-run every audit check affected by the finding.
5. For a node patch, start from a clean clone of the pinned commit, apply
   `node.patch`, and prove the resulting diff and code hashes match the bundle.
6. Write `workflows/<name>.audited.json` containing the original compiled
   workflow SHA-256 plus references and checksums for all accepted remediation
   bundles. Do not overwrite the original compiled workflow.

Only a successfully reproduced and re-audited repair may change a finding from
CRITICAL to remediated.

## Runtime Agent-Review Findings

`medflow-run` performs an automatic agent review after every node attempt. When
that review returns `retry` because adapter/helper code or a node edit is
needed, treat the review as a new audit finding and apply the remediation rules
above. Preserve the failed node-run workspace and its `agent-review.json`;
include its workspace ID, path, and SHA-256 in `remediation.yaml` as the
triggering evidence.

After remediation, re-audit the affected node, its input/output contract, and
all downstream assumptions invalidated by the change. Return an updated
audited workflow with the remediation bundle reference. The run agent then
reruns the node in a new unique workspace, links it to the failed parent
workspace in the append-only registry, and reviews it again. Routine
agent review and contract-preserving retries do not require user approval;
scientific-meaning or public-contract changes still do.

## Checks

### 1. Config Key Audit

For every config key on every step:

- Transform underscore to hyphen and add the `--` prefix.
- Match the result against the target node's `parameters[].name` in
  `SKILL.md`.
- A missing required parameter or unsupported config key is **CRITICAL**.
- Report the step, key, transformed parameter, and available parameters.

### 2. Upstream Data Compatibility Audit

For every `bind: upstream` parameter:

- Identify the upstream step and its declared outputs.
- Normalize these semantic aliases before comparison:
  - `gene_expression_matrix`
  - `expression_matrix_gene`
  - `expression_matrix`
  - normalized value: `expression_matrix`
- A normalized match is compatible. Report spelling differences as
  **WARNING**, not failure.
- When files exist, inspect their actual structure:
  - first column contains feature identifiers;
  - remaining expression columns are numeric;
  - sample IDs are compatible with the corresponding metadata or map.
- Mark **CRITICAL** only when structure or content proves incompatibility.
- If required files do not exist yet, mark the structural checks
  **DEFERRED**.
- Validate runtime file bindings and output-directory arguments against the
  entry point's actual path-resolution and write behavior. A binding that names
  a compatible artifact but passes it through the wrong argument or resolves
  it from the wrong working directory is **CRITICAL**. Re-audit any direct run
  adjustment that changes an argument, path, config value, or file binding.

### 3. Group Column Coverage Audit

For group comparisons:

- Open metadata from every upstream dataset.
- Inspect direct columns and keys encoded in `characteristics_ch1`.
- Verify the configured group column or a compiled per-dataset equivalent.
- When the research question names an exact contrast and upstream data contains
  additional labels, verify the compiled group-map binding declares those
  contrast labels as `allowed_values`. Evaluate group counts after applying
  that allowlist and report excluded-label counts as a warning.
- Count valid group values and warn when any group contains fewer than ten
  samples.
- A missing group with no valid compiled equivalent is **CRITICAL**.
- If fetch metadata does not exist yet, mark this check **DEFERRED**, run only
  the required fetch steps, and repeat the audit before merge or DEG.

### 4. Sample ID Cross-Reference Audit

When the group map and expression matrix exist:

- Verify that column one of the group file contains sample IDs and column two
  contains group labels.
- Cross-reference those IDs against expression-matrix column headers.
- No overlap is **CRITICAL**; partial overlap is a **WARNING** with the matched
  fraction.
- If either file does not exist yet, mark the check **DEFERRED** and repeat it
  after fetch and required inline transformations.

### 5. Node Version Audit

For every node directory:

- Require the compiled workflow step to record its registry URL, declared
  version, resolved default branch, exact commit SHA, and node `SKILL.md`
  SHA-256. A workflow without an exact commit or contract hash is **CRITICAL**
  and must be recompiled.
- Confirm the workflow URL matches the current `registry.yaml` entry. A
  mismatch is **CRITICAL** because node provenance is ambiguous.
- Run `git fetch origin`.
- Verify the recorded commit exists in the cloned repository and compare local
  HEAD with that commit. A mismatch is **CRITICAL**; checkout the pinned commit
  and repeat the audit rather than switching to the latest remote revision.
- Hash the pinned node's `SKILL.md` and compare it with the compiled contract
  hash. A mismatch is **CRITICAL** because parameter provenance is ambiguous.
- Resolve the current remote default branch through `refs/remotes/origin/HEAD`.
  If it has advanced beyond the pinned commit, report a **WARNING** for a newer
  available revision but continue auditing the pinned workflow.
- A default branch named `master` or anything other than `main` is a
  **WARNING**, not a failure.

### 6. Runtime Package Audit

For every declared environment:

- Parse conda and pip dependencies and check their availability.
- If a package is unavailable from conda on the current platform:
  1. Determine whether it is an R/CRAN or Bioconductor package.
  2. Create the declared isolated environment.
  3. Install CRAN packages with `install.packages()` and Bioconductor packages
     with `BiocManager::install()`.
  4. Verify loading inside that environment with
     `Rscript -e 'library(package)'`.
  5. Report a successful fallback as **WARNING**.
- Distinguish "not available from conda on this platform" from "not
  installable."
- Mark **CRITICAL** only when installation or loading fails.

## Output

Emit one NDJSON result line.

Pass:

```json
{"level":"result","status":"pass","checks":6,"warnings":[],"deferred":[],"critical":[]}
```

Pass with reproducible remediation:

```json
{"level":"result","status":"pass_with_remediation","checks":6,"warnings":[],"deferred":[],"critical":[],"remediated":[{"finding_id":"MF-AUD-001","type":"INPUT_ADAPTER","bundle":"workflows/remediations/<workflow>/MF-AUD-001/remediation.yaml","bundle_sha256":"<sha256>","verification":"pass"}],"audited_workflow":"workflows/<name>.audited.json"}
```

Deferred:

```json
{"level":"result","status":"deferred","checks":6,"warnings":[],"deferred":[{"check":3,"step":"fetch1","msg":"Metadata not produced yet; run fetch steps and repeat audit."}],"critical":[]}
```

Failure:

```json
{"level":"result","status":"fail","checks":6,"warnings":[],"deferred":[],"critical":[{"check":1,"step":"fetch1","key":"rename","msg":"Unsupported required configuration."}]}
```

Execution policy:

- `fail`: execute nothing.
- `deferred`: execute only prerequisite fetch steps identified by the audit,
  then repeat the audit.
- `pass`: proceed with the remaining DAG.
- `pass_with_remediation`: execute only
  `workflows/<name>.audited.json`. medflow-run must verify every remediation
  label and checksum, clone the pinned base commit, apply recorded patches or
  run recorded adapters exactly, and reject any unrecorded edit.
